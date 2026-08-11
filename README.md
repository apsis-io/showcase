# [ :: ] Periapsis

### From the Seas to the Stars — Kubernetes' Pawn

**A high-density Kubernetes node agent that runs pods as `systemd-nspawn` machines, not CRI containers.**

Kubernetes has exactly one kubelet. Periapsis is a **second implementation of the Kubernetes node** — it preserves kubelet *semantics* while replacing the CRI execution model with systemd, on the claim that the node is an interface, not a component. It never masquerades as kubelet: events say `perigeos`, the node version says `perigeos://`, and the cluster sees an honest node, not a costume.

Periapsis is a Kubernetes node agent that bypasses the CRI and containerd entirely and runs pods directly on a Linux host as `systemd-nspawn` machines (with host-process and host-runtime WASM launch modes alongside). A single physical host registers as many independent virtual nodes — called **pawns** — each with its own TLS identity, kubelet API endpoint, pod CIDR, and cgroup slice, and each shaped independently (a compute pawn, an I/O-capped storage pawn, a memory-heavy pawn, on one box). The scheduler treats them as separate nodes; the workloads stay native units on the host.

This public repository is intentionally information-only. It contains public project material, test-surface summaries, and benchmark summaries. It does not contain source code, ADRs, or operational secrets (for now, expect a release in around 2 to 3 months). The long-form write-up — motivation, architecture, and the numbers with their methodology — is on Habr (Russian): [«Памятник kubelet, или Kubernetes != CRI»](https://habr.com/ru/articles/1058526/).

```text
[engi@engix99 ~]$ sudo apsis status | head -20
Hostname:    engix99
Version:     4743a2b4
Uptime:      17m8s
Pawns:       7
Pods:        14
Kernel:      7.1.4-x64v3-xanmod1
Arch:        linux/amd64
Go:          go1.26.5-X:nodwarf5
Memory:      18563 / 47996 MiB
CPU cores:   28
Load avg:    1.46 1.85 1.87
PSI cpu:     0.3%
PSI memory:  0.0%

Machines:    16
Disk dirs:   75
Units:       16
RSS:         103 MiB
LXC veths:   8
Netns:       7

[engi@engix99 ~]$ kubectl get nodes
NAME               STATUS   ROLES     AGE     VERSION
compute-09         Ready    pawn      5d10h   perigeos://4743a2b4
engifire           Ready    primary   20d     perigeos://4743a2b4
engifire-pawn-01   Ready    pawn      20d     perigeos://4743a2b4
engifire-scale-1   Ready    pawn      18d     perigeos://4743a2b4
engipi             Ready    primary   14d     perigeos://2a0b9a5a
engix99            Ready    primary   20d     perigeos://4743a2b4
engix99-e2e-1      Ready    pawn      23h     perigeos://4743a2b4
engix99-e2e-2      Ready    pawn      20d     perigeos://4743a2b4
engix99-scale-1    Ready    pawn      18d     perigeos://4743a2b4
engix99-trail-1    Ready    pawn      20d     perigeos://4743a2b4
engix99-trail-2    Ready    pawn      20d     perigeos://4743a2b4

[engi@engix99 ~]$ sudo machinectl | head -4
MACHINE                                                         CLASS     SERVICE        OS     VERSION           ADDRESSES
pod-0f0190c9-92d1-4da2-ac72-4818f091e03a-app                    container systemd-nspawn arch   20260712.0.555161 10.0.67.162…
pod-2d5950fe-9d92-437a-804e-0460fecb1f00-kube-apiserver         container systemd-nspawn wolfi  20230201          -
pod-585678a0-df26-4ee3-86a7-5cac27626773-kube-controll-4a994d4a container systemd-nspawn wolfi  20230201          -
```

Eleven nodes, three physical hosts, two CPU architectures (`engipi` is a Raspberry Pi Zero 2 W — see [benches/README.md](benches/README.md)), and **no kubelet anywhere in the cluster**: every node reports a `perigeos://` version, and the control plane itself is running as `systemd-nspawn` machines. That is a live working fleet rather than a density run — for density, the 1,772-pod / 33-node benchmark is in [benches/README.md](benches/README.md).

---

## What a pawn is

A **pawn** is a virtual Kubernetes node: a full, scheduler-visible `Node` object backed by a slice of a physical host rather than by a machine of its own. The name is wordplay on systemd-ns**pawn**.

```text
[engi@engix99 ~]$ sudo apsis status | grep -A8 '^Pawns:$'
Pawns:
NAME             ROLE     PORT   IP  PODS  CPU(ms)   MEM(MiB)
compute-09       pawn     12261      0     26531     167
engix99          primary  12260      7     15420353  2411
engix99-e2e-1    pawn     12262      0     31305     64
engix99-e2e-2    pawn     12263      0     31647     4
engix99-scale-1  pawn     12264      0     30409     5
engix99-trail-1  pawn     12265      4     241704    176
engix99-trail-2  pawn     12266      3     84757     80
```

Each pawn owns the things a node is expected to own:

- **Its own identity** — a distinct TLS client certificate and node name, so the API server authenticates it as a separate node and the usual node-restriction rules apply to it individually.
- **Its own kubelet API endpoint** — its own port on the host, serving `exec`, `attach`, `logs`, `port-forward`, and metrics for its own pods. `kubectl exec` reaches a pod through the pawn that owns it, exactly as it would through a kubelet.
- **Its own capacity** — CPU and memory allocatable carved from a host-wide budget, enforced as a cgroup slice rather than advertised as a number. The sum of pawns cannot oversubscribe the host, and a per-host budget cap leaves headroom for a co-located real kubelet. Pod capacity per pawn is configurable (the fleet above runs 256 per pawn against the stock kubelet's hard 110).
- **Its own scheduling surface** — labels, taints, cordon/drain, pod CIDR, and per-pod network namespaces. Affinity, anti-affinity, topology spread, and `PodDisruptionBudget` all behave as they would across separate machines.

Allocatable is per pawn and independently sized, so one host can present a large node and several small ones — and every one of them advertises a pod capacity well past the stock kubelet's hard 110:

```text
[engi@engix99 ~]$ kubectl get nodes -o custom-columns=NAME:.metadata.name,PODS:.status.capacity.pods,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
NAME               PODS   CPU   MEM
compute-09         256    2     2G
engifire           256    1     2G
engifire-pawn-01   256    1     512M
engifire-scale-1   256    1     1536Mi
engipi             256    1     512M
engix99            256    4     3Gi
engix99-e2e-1      256    2     2G
engix99-e2e-2      256    2     2G
engix99-scale-1    256    2     2Gi
engix99-trail-1    256    5     9Gi
engix99-trail-2    256    2     2G
```

One host's first node takes the host's own name and the **`primary`** role; it carries the host-level duties, including any static pods and the self-hosted control plane. Additional nodes on that host register as ordinary **`pawn`** nodes. Above the node boundary nothing can tell them apart from hardware nodes except the honest `perigeos://` version string.

Three structural points that are easy to miss:

**Pawns are shaped, not uniform.** A pawn is not just "a fraction of the host" — each one is sized and specialised independently, so a single machine can present a *heterogeneous* set of nodes: a compute pawn, an I/O-capped pawn, a storage pawn, a memory-heavy pawn, on the same box at the same time. What can differ per pawn:

| Knob | Effect |
|---|---|
| CPU quota and CPU weight | hard cap plus relative priority under contention — a compute pawn can be given the CPU that's left when others are idle, without being able to starve them |
| Memory cap | the pawn's slice, enforced as `MemoryMax` on the parent cgroup |
| Read / write bandwidth caps | cgroup-v2 IO limits on the pawn's slice, so an I/O-heavy tenant can't saturate the disk out from under the rest of the host |
| Swap policy | swap allowed, sized, or refused per pawn, overriding the host default |
| Topology (`region` / `zone`) | drives `topologySpreadConstraints` and zone-aware affinity |
| Labels and taints | how workloads are actually routed to the right shape |
| Log isolation | each pawn can own a dedicated `journald` namespace with its own disk cap, so a runaway pod floods its own pawn's journal and neither the host's nor another pawn's |

The routing needs nothing custom: label a pawn and taint it, and ordinary `nodeSelector`, affinity, and tolerations do the rest — the scheduler is choosing between nodes, and these are nodes. That is the part a stock kubelet structurally cannot do. One kubelet is one node with one flat capacity profile and one shared IO and log domain; splitting a host into a fast compute node and a bandwidth-capped storage node means either buying a second machine or paying for a VM layer to fake one.

Two honest limits on that. The shaping is cgroup enforcement, not hardware partitioning — a bandwidth cap is `io.max` on a slice, not a dedicated controller. And the topology override lets you *model* zones, so it should describe real failure domains rather than manufacture them; pawns on one host are not independent no matter what zone label they carry (see below).

**One daemon, many nodes.** Pawns are not one process each — a single `perigeos` process serves every pawn on the host, which is why idle overhead grows so slowly with pawn count (see the RSS figures in [benches/README.md](benches/README.md)). Pawn count is a configuration choice rather than a property of the hardware, and it is changeable at runtime: `apsis pawns add`, `resize`, and `scale` reshape a live host's node topology without restarting the daemon, and `apsis pawns remove --drain` cordons and evicts a pawn's pods before retiring it. A shrink that would fall below what its pods have already been promised is refused rather than silently overcommitted.

**A pawn is not an isolation or failure boundary.** It is not a VM, adds no kernel, and adds no hypervisor. Containment comes from the per-pod cgroup slice and namespaces, not from the pawn — two pods on different pawns of one host are isolated from each other exactly as much as two pods on the same pawn, and no more. Consequently **more pawns does not mean a bigger blast radius, and it does not buy high availability.** Every pawn on a host shares that host's kernel, hardware, and power supply. If the host goes down, every pawn and every pod on it goes down together, no matter how finely it was sliced. High availability still comes from spreading workloads across separate physical **hosts** with anti-affinity or topology spread, the same as it would with stock kubelet nodes.

**Terminology:** a *host* is not the same thing as a Kubernetes *node*. One host presents many pawns, and each pawn is a node. Confusingly, `apsis status` and `machinectl` also use the word *machine* — there a "machine" is a running **pod**, not a host.

---

## Kubernetes is not kubelet + CRI

Kubernetes is the orchestration API and object model: the API server, the scheduler, controllers, and objects like Node, Pod, Service, and Deployment that describe *what* should run and *where*. None of that changes with Periapsis — a pawn is a normal Node object, a pod is a normal Pod object, and standard `kubectl`, controllers, and the scheduler all work unmodified.

What Kubernetes has historically delegated to each node is a **kubelet** (the per-node agent that watches assigned pods and makes them real) talking to a **CRI runtime** (containerd, CRI-O) over the Container Runtime Interface to actually create containers. That kubelet+CRI pairing is one implementation of "make pods real on a Linux host" — it is not Kubernetes itself, and it is not the only way to satisfy the node contract.

Periapsis replaces that one pairing, not Kubernetes. It speaks the same node-facing contracts (the kubelet HTTP API, the pod lifecycle semantics) but skips CRI/containerd entirely, running pods as native `systemd-nspawn` units instead of OCI containers behind a shim. From the API server down to the scheduler, nothing changes; the substitution happens below the node boundary, not above it.

Periapsis can also go a step further and bootstrap the control plane itself: `perigeos` self-hosts the apiserver, controller-manager, and scheduler as static pods, cold-starting them from on-disk static manifests (the same kind `kubeadm` generates) and handing off to normal reconciliation — with no k3s or kubeadm running as the ongoing orchestrator. This doesn't change the boundary above — it's still a standard API server, standard objects, standard scheduler — it just means Periapsis can stand up and keep running that side too, or you can keep running k3s/kubeadm in front of Periapsis nodes as before.

---

## Conformance

"It satisfies the node contract" is a claim that has to be *measured*, not asserted. Periapsis is validated against the **upstream Kubernetes end-to-end node suite** — the same Ginkgo specs the project ships to certify a kubelet, run unmodified against a real cluster with the version-matched `v1.36` e2e binary, pointed at pawn nodes.

Latest tracked run (2026-07-25), across all specs run to date:

| | |
|---|---|
| Distinct specs run | **752** |
| Passing | **657** |
| Actionable failures | **0** |
| Deferred (real gaps, consciously postponed) | 3 |
| Ignored (non-actionable) | 92 |

The three **deferred** specs are one CSI feature (per-pod service-account-token plumbing through `NodePublishVolume`) that is in scope and simply not on the current worklist.

The 92 **ignored** specs are excluded with a written reason each, and the reasons are of three kinds:

- **Off-by-default feature gates** that the cluster under test doesn't enable (e.g. `PodCertificateRequest`, alpha pod-level-resources resize), so the spec cannot run meaningfully either way.
- **Control-plane concerns that aren't a node's job** — kubeadm bootstrap-token behaviour, API-server storage/chunking semantics.
- **Identity divergences** — specs that assert the node is *literally* a kubelet: kubelet-branded metric names, kubelet-internal cgroup layout, and version-skew artifacts where an upstream release renamed a spec family. Periapsis deliberately does not pretend to be kubelet, so these fail by design rather than by defect; each one is recorded with what it asserts and why the divergence is intentional.

Selected results worth calling out: the in-place pod resize family (KEP-1287) is **9/9** on v1.36 — app containers, native sidecars, running plain init containers, and rollback — with a running container's cgroup limits moved live and `status.containerStatuses[].resources` reflecting the enacted value.

Conformance is tracked as a living artifact in the private tree (a per-spec verdict table regenerated from the JUnit output of every run, most-recent-run-wins), not as a one-off screenshot. Numbers above are a snapshot of that table.

---

## Why

A stock kubelet carries a hard **110-pod ceiling** and the weight of containerd plus a runc shim *per container*. On powerful hardware that forces a choice between underutilising the box and adding a hypervisor layer (KVM / KubeVirt / Kata) that pays a microVM tax in memory, boot latency, and jitter. On a small VPS or edge machine the standard stack is heavy before a single workload runs.

Periapsis removes the entire CRI stack. A pod is a `systemd-nspawn` transient unit registered with `systemd-machined` — there is no shim process to account for, and supervision is `systemd`'s own. It chooses **OS-level isolation** (Linux namespaces + `systemd-nspawn`, sharing the host kernel) over hardware virtualisation: you trade hostile multi-tenant isolation for extreme density. For trusted internal infrastructure, CI/CD pipelines, and edge compute, that is the right trade.

| | Stock kubelet + containerd | Periapsis |
|---|---|---|
| Idle daemon RSS | ~350 MB / node | **~70–130 MB**, depending on pawn count (one process) |
| Per-pod tax | ~15–20 MB (shim) | **< 1 MB** (native unit) |
| Pods / host | 110 (hard cap) | **thousands** |
| Per-runtime-op daemon | yes (containerd) | none (talks to `systemd-machined`) |
| Logs | CRI text files | native `journald` |
| Visibility | `crictl` / `ctr` | transparent (`machinectl`) |
| Daemon upgrades | disruptive (drain node) | zero-downtime (`KillMode=process`) |

See [benches/README.md](benches/README.md) for the public benchmark notes, including the 1,772-pod density/throughput result and a direct head-to-head against stock kubelet on identical hardware (2026-08-11): ~1.4x faster to deploy 200 pods, ~2.2x faster from Scheduled to Running, and ~1.8x less host memory — with the caveats, including the ones that cut the other way, stated there.

---

## Use cases

Periapsis fits trusted, density-first deployments where the CRI / microVM tax is the bottleneck.

- **A real multi-node cluster on one machine** — one host registers as 30+ independent pawns, so a laptop or a single server can exercise the cross-node scheduling, networking, and disruption behaviour a single-node dev cluster cannot. And because pawns are shaped independently, it can be a *heterogeneous* cluster — a fast compute node beside a bandwidth-starved storage node beside a 512 MB edge node — so a workload meets that shape before real hardware does. (Compared against `minikube` / `kind` / `k3d` under [Comparisons](#comparisons).)
- **High-density CI/CD and batch** — thousands of short-lived build/test pods with no containerd shim per pod; a pod is a transient `systemd-nspawn` unit, so per-pod overhead is sub-MB and churn is cheap.
- **Edge & small VPS** — the stock kubelet+containerd stack is heavy before any workload runs; a single-pawn Periapsis daemon idles in the 70–130 MB range and leaves the box for the workload (plus the WASM (WASIp3 components) `runtimeClass` path for tiny, sandboxed edge workloads).
- **Homelab / bare-metal saturation** — fill a big Xeon/Threadripper box past the 110-pod cap without underutilising it or adding the KubeVirt/Kata microVM latency and memory tax. Co-exists with an existing k3s/kubelet node on the same host.

It is the **wrong** tool for hostile multi-tenancy: pods share the host kernel, so untrusted code that can exploit a kernel bug is out of scope — reach for a microVM runtime there. It is also the wrong tool for high availability on its own: see [what a pawn is](#what-a-pawn-is) for why slicing one host into more pawns buys density, not redundancy.

---

## Key features

**Pod lifecycle (kubelet-conformant semantics)**
- Create / update / delete; restart policies; CrashLoopBackOff (systemd owns the restart, the FSM observes).
- Init containers, native sidecars (init w/ `restartPolicy: Always`), and liveness / readiness / startup probes (exec, httpGet, tcpSocket, grpc).
- postStart / preStop lifecycle hooks with a shared grace budget and a hard SIGKILL-at-grace cap.
- Pod phases, conditions (including `PodReadyToStartContainers`), `terminationMessage`, OOMKilled detection (including nested-cgroup payload OOMs), signal exit codes.
- Restart durability: the reconciler's plan position is snapshotted and restored, so a running pod survives a daemon restart with no `Pending` flap.
- `sd_notify` from pod workloads: a systemd-native service inside a pod can report readiness over `NOTIFY_SOCKET` — no CRI equivalent exists.

**Security** — full SecurityContext support; fields perigeos can't honour faithfully are *rejected* at config build (a `FailedCreateContainerConfig` warning event in `kubectl describe`) rather than silently mis-launched.

```text
# runAsUser:1000 runAsGroup:1000 fsGroup:2000 supplementalGroups:[3000,4000], with an emptyDir
$ kubectl exec demo -- id
uid=1000 gid=1000 groups=2000,3000,4000
$ kubectl exec demo -- stat -c '%U:%G %A' /data
root:2000 drwxrwsr-x                       # fsGroup owns the volume, setgid bit set
```

`runAsNonRoot`, `allowPrivilegeEscalation`→`NoNewPrivileges`, `capabilities` add/drop (nspawn and userns paths, ambient + bounding handling), `seccompProfile`, `sysctls`, `readOnlyRootFilesystem`, `privileged` are all honoured. **Root in the pod is not root on the host:** systemd userns mapping maps a container's UID 0 to an unprivileged high host UID, so a container escape lands as a host *nobody*. (Pods share the host kernel — Periapsis targets trusted workloads, not hostile multi-tenancy.)

**Resources & containment**
- `resources.limits` become real cgroup-v2 caps (`memory.max` / `cpu.max` / `pids.max`), not scheduler hints — an over-limit container is contained, not the host.
- **In-place pod resize (KEP-1287)**: a running container's CPU/memory limit is moved on its live cgroup without a restart (`resizePolicy: NotRequired`), or with a deliberate restart where the policy asks for one — for app containers, native sidecars, and running init containers alike, with a `ResizeCompleted` event and enacted values in the pod status. **Pod-level resources** (the pod-scoped `spec.resources`) and pod-level resize are supported too, and the node advertises the matching feature so the scheduler can rely on it.
- Per-pod PID cap / fork-bomb containment (`peri.apsis/max-pids`).
- Node-pressure eviction ranked kubelet-style (QoS → priority → usage); disk-pressure image/layer GC before pod eviction.
- Per-pawn slice caps and a per-host budget cap, leaving headroom for a co-located real kubelet. **Dynamic pawn scaling** from the budget: `apsis pawns scale <set> <count>` reconciles a pawn set to a target count, sizing each pawn to an even share of the host budget; `apsis pawns add` / `resize` / `remove --drain` change a live host's node topology without a restart.
- Opt-in **KSM memory dedup** per pod (`peri.apsis/memory-ksm`) — many near-identical pods on one host share identical pages.

**Idle & wake (scale-to-zero)** — a pod opts in with `peri.apsis/scale-to-zero`, and collapses after an inactivity window. There are two tiers, and the pod object survives both:

- **Frozen** (`peri.apsis/freeze-after`) — the same process, the same memory, simply taken off the CPU via the cgroup-v2 freezer. A warmed-up JIT, connection pools, caches, loaded model weights all stay warm; thawing clears the freeze flag and the process continues at the instruction it stopped on. Honestly stated: this saves CPU and wake time, **not** RAM.
- **Idled** (`peri.apsis/idle-after`, default 5m) — the process really stops and its memory is reclaimed, but the pod is not deleted: it stays in the API server, its network namespace, IP, and rootfs are retained, and the image is still on the node. Waking starts a fresh process inside the environment that is already there. Idle apps cost disk, not RAM.

An idled pod reports a truthful `Ready=False, reason=Idled` rather than pretending to be pending or crashed. The wake itself measures **~1.4–1.6 s**, consistently (see [benches/README.md](benches/README.md) — timed around an explicit wake command, so it is the wake operation rather than an end-to-end request).

Traffic triggers it automatically, through an **activator** that stays out of the datapath: an eBPF program on the existing CNI datapath counts per-pod flow activity (which is what detects idleness in the first place) and traps a SYN to an idled pod's IP, while the activator agent — reading events, never packets — fires the wake. One honesty note on that path: eBPF cannot complete a handshake to a backend that does not exist yet, so it drops the SYN and lets the client's own TCP stack retransmit. The connection then succeeds, and from the client's side it looks like a slow connect rather than an error, but the observable floor is quantized to the client's retransmit schedule (≥1 s) on top of the wake itself: a pod that is genuinely ready in 200 ms is still only reachable at the next retry. In exchange it covers any TCP protocol rather than HTTP only, and puts nothing in the hot path of a warm request. An HTTP-aware wake hook at the edge, which would hold the request and remove the quantization, is designed but not yet built.

```text
$ kubectl get pods -n nginx-demo
NAME                           READY   STATUS    RESTARTS      AGE
nginx-whoami-758f9d579-nd2gw   0/1     Idled     0             5d10h   # RAM freed, IP retained
nginx-whoami-758f9d579-x69vp   1/1     Running   1 (13h ago)   5d10h
```

Alongside the automatic path, an operator can drive it directly: `apsis sunset` / `dawn` (idle and wake), `apsis freeze` / `thaw` (the freezer tier), and wall-time / CPU-time budgets (`apsis budget`, `apsis fuel`) that force a pod idle once it has consumed its allowance.

How this differs from Knative's scale-to-zero — a different mechanism, not a different tuning — is in [Comparisons](#comparisons) below.

**WASM (Trail) workloads** — a pod with `runtimeClassName: trail` runs a WASIp3 **component** via the **host-runtime path**: a transient unit joined to the pod netns, `wasi:sockets` bound on the pod IP (a WASM runtimeClass with no installed runtime fails closed). Bare wasip1 CLI runtimes (running a raw core module with no component model, no host-capability profile, no checkpoint/restore) are deliberately not supported in favor of Trail's component path; `trail` is the only WASM runtimeClass today (earlier separate `wasmtime`/`wasmedge`/`wasmer` classes have been consolidated into it).

```text
$ kubectl logs hello-wasm                  # runtimeClassName: trail, a WASIp3 component
hello from wasm
```

Trail is **deny-by-default**: a component gets no filesystem, no network, no environment, and no host capability unless the pod grants it, one annotation per capability, with node-wide default and ceiling policies that apply even to a pod that asks for nothing. An un-granted import fails closed rather than silently succeeding.

Two capabilities of the component model that a container runtime structurally cannot offer:

- **Composition at launch.** A component's WIT imports can be satisfied by fusing a provider into a single component at launch, by plugging one in-process with live hot-swap (`apsis pods swap-plug`, no consumer restart), or by binding to a provider running on *another node* — with a scheduler moving a workload between those tiers from live traffic signals.
- **Cross-architecture live migration.** A component that exports the checkpoint contract can be moved to another node — **including a node of a different CPU architecture** — with its in-memory state intact, via `apsis pods migrate <pod> --to <node>`: the source is checkpointed on a coordinated stop, the bytes are relayed, and the target restores and resumes. A `.wasm` component has no per-architecture build, so a cross-ISA move is the same mechanism as a same-host one. Live-validated **amd64 → arm64** (2026-07-09), with the target resolving the component image peer-to-peer and the counter continuing rather than resetting.

Both of those capabilities need something above the node to decide *what binds to what*, and that is **Radiant** — the Trail operator, a leader-elected Deployment and the seam's cluster-side brain. It holds the registry that resolves a pod's declared seam need to a concrete provider (or derives it from the component's own WIT imports), promotes and demotes workloads between the composition tiers above from live signals, and owns the migration object behind `apsis pods migrate`. Binding is resolved at launch and then held by the node, so a Radiant outage stalls *new* policy decisions without breaking a single already-bound pod. Not to be confused with **meteor**, which shares only the sky — see [Naming](#naming).

Checkpoint/restore here is *cooperative* by design — the component serializes its own state through a WIT seam — rather than an attempt to snapshot the engine's execution state, which a feasibility study found infeasible for components in the first place.

**Networking** — [Constellation](https://github.com/malformed-c/constellation), a Cilium CNI fork, for the multi-pawn case: eBPF datapath, VXLAN cross-host routing, per-pod netns. Standard CNI (`/etc/cni/net.d`) covers standalone 1:1 host-to-node deployments. Cluster DNS, `dnsConfig`/`dnsPolicy`, managed `/etc/hosts` + `hostAliases`. Pod-to-pod (same/cross host), ClusterIP Services across hosts, NetworkPolicy, and Envoy Gateway L7 ingress all work with pawn-hosted workloads. Pods with a conflicting `hostPort` are rejected at admission the way a kubelet rejects them, rather than failing later at launch.

```text
# NetworkPolicy enforcement (server + client on different pawns):
$ kubectl exec client -- wget -qO- http://<server>:8080/      # no policy
hello ...
# apply default-deny ingress on the server:
$ kubectl exec client -- wget -T5 -qO- http://<server>:8080/
wget: download timed out                                       # blocked
# apply allow-from podSelector{app=client}:
$ kubectl exec client -- wget -qO- http://<server>:8080/
hello ...                                                      # allowed again
```

**Images & P2P distribution** — OCI pull into a content-addressable store; layers shared copy-on-write via OverlayFS. Both **layers and the manifest** are seeded **peer-to-peer** across pawns and hosts, so a node can pull an image with **no registry access at all** — a private (403) image, or a node whose only path to the registry is down — as long as one peer holds it. The peer fabric is:

- **locality-aware** — peers are tried closest-tier-first (same subnet → same private network → remote), so a node prefers a near peer over a WAN one;
- **HTTP/3 (QUIC)** with a per-peer TCP fallback — better on a lossy/long-haul link, falling back automatically where UDP is blocked;
- **registry-independent (P2P manifests)** — the manifest is served peer-to-peer, not just layer blobs, and resolution honours pull policy: `Always` still resolves from the registry **first** (so images keep updating) and falls back to a peer only on failure; `IfNotPresent` tries a peer **first**, registry last; `Never` stays local. A node that resolves from a peer then serves the manifest onward (gossip).

```text
# A node pulling a PRIVATE (403) image entirely from a peer — manifest *and* layers, no registry:
$ kubectl get events
Normal  ResolvingManifest  pod/wasix-engifire  Resolving image manifest: ghcr.io/.../wasix-info:latest
Normal  PulledFromPeer     pod/wasix-engifire  Layer a03d287ccca9 pulled from peer 192.168.100.200:12261 (h3, tier 0)
Normal  Started            pod/wasix-engifire  Container server started
```

Pull policies, `imagePullSecrets`, and digest verification are supported. **`apsis ingest`** loads an OCI/docker image tar straight into a node's library — served P2P, usable by pods, pinned against GC — so an image reaches the cluster with no registry. It also accepts a raw `.wasm`/`.wat` component or a native ELF binary and packs it into a minimal runnable image automatically. Measured end to end (2026-07-22): a 3.4 MB `.wasm` file on disk to a pod serving real HTTP traffic in **~1.5 s** — no Dockerfile, no registry, no CI.

The layer cache is **garbage-collected** automatically: under disk pressure perigeos reclaims orphaned layers, then evicts whole unused (not running, not pinned) images LRU before falling back to evicting pods — plus `apsis cleanup` / `apsis images rm` on demand.

**Storage** — `configMap`, `secret`, `projected` (incl. rotated serviceAccountToken), `emptyDir` (incl. `medium: Memory`), `downwardAPI`, `hostPath`, PVC/CSI (including CSI-requested service-account tokens and node volume expansion), and **image volumes** (KEP-4639) — an OCI image mounted as a read-only volume rather than run as a container.

**Self-hosted control plane** — Periapsis can host the control plane it registers with: `kube-apiserver`, controller-manager, and scheduler as static pods on the primary pawn, backed by SQLite instead of an etcd cluster. The chicken-and-egg (the node agent must start the API server it needs) is resolved explicitly rather than by luck:

- **Cold bootstrap** — on a fresh host, or one whose control plane is down, the daemon runs a minimal API-server-independent pipeline whose only job is launching the static pods; while it makes progress it keeps extending its own systemd start timeout, the systemd-native way to say "still coming up" for a phase with no fixed upper bound. Once the API server answers, the same process registers each static pod into the API, re-admits it under its real API-assigned UID, and hands off to normal reconciliation.
- **Warm start** — with a control plane already up, the daemon boots straight into normal mode and *adopts* what is already running, rather than restarting it.
- **Host reboot** — the daemon observes that believed-alive machines are gone and relaunches them, entering cold bootstrap first if the reboot took the API server down too. The control plane self-heals up, then the workloads follow; no manual sequence and no external orchestrator.

Static pods carry extra durability rules so the control plane can never eat itself: they are exempt from orphan reaping and immune to scale-to-zero idling, and structural or resources-only edits to a static manifest apply in place. The whole ladder — cold-booting a complete self-hosted control plane on an unprepared host — is validated end to end on a 1 vCPU / 2 GB VPS.

**Operability**
- Kubelet HTTP API: `exec` (incl. stdin), `attach`, `logs` (incl. `--previous`), `port-forward`.
- Native `journald` logs and `machinectl` visibility — operators inspect pods with the same tooling they already use.
- Zero-downtime daemon upgrades (`KillMode=process`): restarting/upgrading perigeos does not stop the pods; it rediscovers them and resumes reconciling. Measured under sustained ~1,500 req/s load: zero failed requests and zero container restarts across a daemon restart (see [benches/README.md](benches/README.md)).
- Metrics beyond the kubelet's: PSI CPU/memory/IO stall pressure for the host and the daemon's own slice, and kernel netns/skb slab occupancy — the things that actually bite at high pod density.
- The **`apsis`** CLI: `status`, `doctor`, `inspect`, `pods`, `pawns`, `machines`, `top`, `showcase`; `images` (list / `rm` / `verify` / `prune` / GC pins), `ingest`, `pull`; `sunset` / `dawn` / `freeze` / `thaw` / `budget` / `fuel`; `pawns add` / `resize` / `scale` / `remove`; `pods migrate` / `swap-plug`; `rollout`, `decommission`, `cleanup`, `stop`.

### Not supported / in progress

- **Ephemeral containers** (`kubectl debug`): supported — launched into the pod's netns, statused in `ephemeralContainerStatuses`, exec/logs-able, never restarted. Only the `--target` PID-namespace variant is rejected fail-closed for now.
- **`seccompProfile: Localhost`**: rejected — a custom on-node BPF profile can't be loaded via nspawn (fail-closed). `Unconfined` is accepted with a warning. `kernel.*`/`fs.*` `sysctls` are rejected (no namespace equivalent).
- **Windows / non-systemd Linux**: not supported, not planned.
- **StatefulSet stable identity**: supported — stable hostname/DNS and PVC naming work via the existing volume and DNS paths.
- **Generic ephemeral volumes**: supported via the auto-provisioned PVC path.
- **Inline CSI ephemeral volumes**: rejected at admission with a clear error — use a PVC + CSI StorageClass instead.
- **VolumeSnapshot**: works via the CSI/PVC path.

Periapsis can **coexist** with a standard kubelet or k3s node on the same cluster. Standard CNIs (Calico, Flannel, upstream Cilium) work for 1:1 node-to-host deployments; multiplexing one host into multiple pawns ("pawn slicing") requires the Constellation CNI fork.

---

## Comparisons

What this is *not*, working from the neighbouring layers of the stack inward to the comparison that matters.

### vs k3s

k3s is a compact **control plane** — one binary, kine instead of etcd — but its node is the vanilla one: kubelet plus containerd, and the same 110-pod ceiling. k3s solves cluster *installation*; Periapsis solves the *node*.

It now does the former as well: `perigeos` cold-starts its own apiserver, controller-manager, and scheduler from kubeadm-style static manifests, so nothing has to keep running as a permanent orchestrator underneath. The manifest format stays; the standing dependency goes. Validated on a clean 1 vCPU / 2 GB VPS that had never seen the project: control plane plus kube-proxy, CoreDNS, and a workload fit in roughly 523 MB, came up end to end, and served real external requests through a NodePort.

So the boundary here isn't hard. Run k3s in front of Periapsis nodes exactly as before, or don't run it at all.

### vs Capsule

Capsule is a **tenancy** layer: it groups namespaces into Tenants and constrains what each one may ask for — quotas, RBAC delegation, allowed ingress classes, network policy defaults — enforced by admission webhooks above the node. Tenants still share nodes, a kernel, and one container runtime.

A pawn is not a tenancy concept at all. It is a node: its own cgroup slice, its own TLS identity, its own pod CIDR, its own scheduler entry. Where Capsule tells a tenant what it may request, a pawn is a boundary the kernel enforces whether anyone requested anything or not.

That makes them complementary rather than competing, and the combination is the interesting one: give a tenant its own pawns via labels and taints, and its ceiling stops being a quota the control plane accounts for and becomes a cgroup the host enforces. Capsule keeps doing the part Periapsis has no opinion about — Periapsis has no tenancy model, no tenant owners, no self-service namespaces, and no plans for any.

**One thing worth stating plainly, because "isolation" invites the wrong reading:** a pawn is not a security boundary against a hostile tenant, any more than a container is. It is namespaces and cgroups on a shared kernel. It buys resource containment and blast radius, not protection from a tenant attacking the kernel — for that, see [vs KubeVirt](#vs-kubevirt). Neither Capsule nor pawns change that; Capsule governs what tenants may do, pawns bound what they can consume.

### vs LXC / Incus

Incus is a system container manager in its own right, with no Kubernetes above it: its own CLI, its own API, its own model of profiles and networks.

Periapsis uses a similar isolation primitive — namespaces and cgroups via `systemd-nspawn`, the same class of thing as LXC — but remains a **Kubernetes node**: the same API server, the same scheduler, Deployments, Services, NetworkPolicy, and the whole existing toolchain and YAML ecosystem people already know. The difference isn't the isolation. It's what sits on top of it.

### vs KubeVirt

Both put something that is not an OCI container into a Kubernetes pod, and both use the word *machine*. They mean different things by it.

KubeVirt runs a full virtual machine — its own kernel, its own bootloader, hardware-level isolation — inside a `virt-launcher` pod, which is itself an ordinary container. So a VM costs a guest kernel plus QEMU plus the entire CRI stack underneath it. Periapsis's machines are OS-level: namespaces and cgroups on the *host* kernel via `systemd-nspawn`. A pod really is a machine in the `machinectl` sense — it can run an init system, it appears in the machine registry — but it shares the kernel it runs on.

**Where KubeVirt wins is not a tuning difference and no amount of nspawn work closes it:** a different kernel, a different operating system, or a tenant you do not trust with yours. Windows, an old kernel a vendor still requires, genuinely hostile multi-tenancy — that is KubeVirt or Kata, not this. The floor is the honest summary: a KubeVirt VM starts at a guest kernel and a QEMU process; a machine here starts at a supervisor process measured at 48 KB.

They also compose. Nothing stops a cluster running KubeVirt nodes beside Periapsis nodes; they answer different questions about the same word.

### vs minikube / kind / k3d

A single-node cluster on a laptop — or a pod-in-pod one, as with kind and k3d — is good for checking that a manifest applies. It is not good for checking the things Kubernetes exists for in the first place: how (anti-)affinity behaves, topology spread, Service and NetworkPolicy and CNI *between* nodes, or a PodDisruptionBudget under a real cordon and drain.

One Periapsis host registers as 30+ independent pawns. That is an actual multi-node cluster to test against, not one node wearing several names — and since [pawns are shaped independently](#what-a-pawn-is), it can be a *heterogeneous* one.

### vs Multus

These get compared because both put more than one network identity on a single host, but they are orthogonal and neither substitutes for the other.

Multus is a meta-CNI: it attaches **multiple interfaces to one pod**, so a workload can sit on a management network and a dataplane network at once, or reach an SR-IOV device directly. The pod stays on one node.

Pawns go the other way. Each pawn is a separate **node** with its own pod CIDR, its own network namespace per pod, its own kubelet endpoint and TLS identity — and a pod on it still gets exactly one interface, as it would anywhere else. Pawns are a node-identity feature that happens to include networking, not a networking feature.

So they answer different questions, and they should compose: Multus is invoked through the standard CNI path, which a Periapsis node with standard CNI configuration uses unchanged. **That combination has not been validated here**, and it is worth saying rather than implying — the multi-pawn case runs on [Constellation](https://github.com/malformed-c/constellation), and what a meta-CNI does across pawn boundaries has not been tested.

### vs Knative

The shared premise is the same: a pod nobody is talking to should collapse. The mechanism is not.

Knative Serving takes the Revision's Deployment to zero replicas, and the ReplicaSet deletes the pod outright. The next request therefore builds a **new** pod — new UID, new IP, a trip through the scheduler, CNI IPAM from scratch. In Knative's model, "collapsing" a pod *is* deleting it.

In Periapsis the pod object never goes away. A frozen pod is the same process taken off the CPU; an idled pod is the same pod with its process stopped — same UID, same IP, same network namespace, image already local, scheduler not involved. That is what the ~1.4–1.6 s figure describes, over 9 clean cycles.

**No Knative number is quoted next to it, deliberately.** There is no reproducible measurement of Knative taken on this hardware, and published cold-start benchmarks depend far too much on configuration to make an honest side-by-side. The two are structurally different operations regardless: one resumes an existing pod, the other assembles a new one.

**Both have an activator. They are opposite designs.** Knative's sits *in* the request path: traffic for a scaled-to-zero Revision is routed to the activator, which holds the request until a pod is ready and then proxies it through — plus a queue-proxy sidecar riding along in every pod. Periapsis's activator never sees a packet. It is a node-side control agent driving the eBPF program on the datapath that is already there: per-pod-IP flow counters feed idle detection, and a SYN to an idled pod's IP is caught, reported, and dropped, so **the client's own TCP retransmit is the buffer** — nothing holds the connection, and no proxy is added anywhere.

That is a deliberate trade, and it was taken the other way first: an earlier design did put an in-path L7 activator in front of every app, and it was rejected because a userspace hop that exists purely for the cold case gets paid on 100% of traffic, warm requests included. The price of the current design is exactly the retransmit quantization described above — a coarser wake, in exchange for zero cost on every warm request.

They also aren't substitutes. Knative is a portable serving layer *above* the node — revisions, traffic splitting, request-driven replica autoscaling, eventing — and it runs on any conformant cluster. Periapsis's version is a node-level primitive on ordinary pods: no new API objects, no per-pod sidecar, no ingress dependency, but equally no revision model and no replica autoscaling, and it needs Periapsis nodes. Nothing structural stops Knative from running on top of Periapsis nodes, since everything above the node boundary is ordinary Kubernetes — though that combination hasn't been validated here.

### vs runwasi

runwasi is a containerd v2 shim: it runs a Wasm module in place of an ordinary OCI container, but stays **inside** the CRI / containerd / shim architecture — the same process-per-workload, the same layer this project removes, just with a Wasm runtime behind it instead of runc.

Here the same job is `runtimeClassName: trail`: a WASIp2/p3 component executed directly by the host runtime, joined to the pod's network namespace. No shim, no extra process per workload. It is the same argument as the rest of this project — nothing between the declared state and the process on the host — applied to Wasm rather than to OCI containers.

### vs kubelet

This is the comparison that matters; everything above is a neighbouring layer.

One container's path on a standard node: **kubelet → CRI → containerd → containerd-shim → runc → process.** Every layer has its own process, its own state, its own restart, and its own bugs. The same container's path here: **perigeos → D-Bus → systemd → process** — and in that chain perigeos is not the supervisor but the *declarer*. It tells systemd what should be running and steps back to watch.

Almost everything else falls out of that one subtraction, and most of it did not have to be *built*:

- Logs aren't gathered by an agent tailing text files — they are already in `journald`, because a pod is a unit.
- Restarts and backoff aren't implemented in the daemon — they are `Restart=` and `RestartSec=`, because a pod is a unit.
- Zero-downtime daemon upgrade isn't a mechanism but a consequence of `KillMode=process`: pods are children of systemd, not of perigeos, so restarting the daemon simply doesn't concern them.
- Visibility doesn't need `crictl` — `machinectl` and `systemctl` show pods, because pods have nothing to hide: they are ordinary nspawn machines.
- "A node is a machine" stops being a law of nature — 30 pawns on one host are 30 cgroup slices with their own certificates.

A kubelet does all of this too, at the cost of that layer: three daemons, a shim per container, its own log pipeline, and its own supervisor stacked on the supervisor every OS already ships. That's where the numbers in [Why](#why) come from — not "we optimised kubelet", but "we removed what it was obliged to carry".

That claim is now measured rather than argued. On one board, the same Deployment, the same control plane on a separate machine, kubelet and Periapsis in turn: at 200 pods Periapsis deploys in ~40 s against 57 s, reaches Running ~2.2x sooner, uses 1.8x less host memory, and the node agent itself sits at ~150 MB against kubelet-plus-containerd's ~400 MB. The containerd-shims alone account for roughly 46% of kubelet's memory delta. Full tables, protocol, and the caveats that cut the other way are in [benches/README.md](benches/README.md).

And to be entirely fair: **kubelet is not a mistake.** It was designed when "node" meant "physical machine", Docker was the only runtime, and systemd could not do half of what it does now. It made containers a boring, dependable technology, and it has earned a monument. A monument just doesn't have to stand on every node and eat half a gigabyte of memory.

---

## Build & test

The commands below describe the private source tree's build and test flow; they aren't runnable from this repository, which contains no source. See [tests/README.md](tests/README.md) for the public test-surface summary.

```bash
# Build the daemon (and the apsis CLI, and the Trail WASM runtime)
go build ./cmd/perigeos
make build              # perigeos + apsis + trail, with version stamping
make fetch-deps         # fetch the patched systemd-nspawn dependencies
make build-wasm-samples # build the sample WASIp3 components

# Test (unit tier: no root, no systemd, no cluster)
go test ./...           # == make test / make test-unit
make test-wit           # validate WIT, Rust bindings, and local component linking

# Integration tests need root + a running systemd
make test-integration

# End-to-end suite (needs a live cluster + pawn node)
make test-e2e E2E_NODE=<node>
```

### Deploy

```bash
sudo make install            # binaries, nspawn deps, service file, config
sudo systemctl start perigeos

kubectl get nodes
sudo apsis status
sudo apsis doctor
```

- Config: `/etc/apsis/perigeos/perigeos.toml`
- State: `/var/lib/apsis/perigeos`
- Logs: `journalctl -u perigeos`

### Requirements

- systemd v250+ (v260+ recommended) and cgroups v2. The pod user-namespace path additionally needs the nspawn fix from [systemd#41838](https://github.com/systemd/systemd/pull/41838) (merged upstream 2026-05-28); until that reaches your distro's systemd, Periapsis ships a patched `systemd-nspawn` build alongside the daemon.
- Linux kernel 6.6+ for the multi-pawn eBPF CNI path; the single-pawn, host-network path runs on older kernels (verified on 6.1/arm64 — see [benches/README.md](benches/README.md))
- Kubernetes 1.34+ (conformance is tracked against the version-matched 1.36 e2e suite)
- Go 1.26+ to build
- Optional: the Constellation CNI fork for cross-host networking and multi-pawn isolation; without it, pods use host-network veth bridges (single-pawn only)
- Optional: kernel arg `swapaccount=1` for swap enforcement

---

## Naming

| Name | Role |
|------|------|
| **Periapsis** | The project / repository (closest orbital approach) |
| **Perigeos** | The daemon binary (Earth-specific periapsis) |
| **Apogeos** | The Periapsis operator — the cluster-side half of perigeos's own concerns (node identity, CSR approval) |
| **Apoapsis** | The self-hosted control-plane distribution: SQLite-backed apiserver + control-plane static pods (farthest orbital point) |
| **Pawn** | A virtual Kubernetes node — wordplay on systemd-ns**pawn** |
| **Scout** | The `PodLifecycleHandler` seam bridging the VK framework to the reconciler |
| **Recon** | The sharded reconciler (Foci shards, Groundfall signal store, PodSM) |
| **Clade** | The pure FSM + status kernel driving each container's state |
| **Horizon** | The sole pod-status + event writer to the API server |
| **Tidal** | Pod materialisation — the cluster's state flowing down into the pod: env, volumes, rotated tokens |
| **Constellation** | The Cilium fork for multi-pawn networking |
| **Trail** | The WASM host runtime and its capability-profile model |
| **Radiant** | The Trail operator — cluster-wide control plane for component composition and migration (the radiant is the point in the sky a meteor shower streams from) |
| **meteor** | No relation beyond the sky: the tiny container entrypoint shim that sets a pod up and `execve()`s into the workload, leaving nothing of itself behind |
| **Apsis** | The CLI for introspection and debugging |

---

## Upstream fixes

Building a node agent on `systemd-nspawn` means hitting bugs in `systemd-nspawn`. Two were found this way and sent back upstream rather than worked around locally:

- [**nspawn: fix EPERM when using `--private-users` with `--network-namespace-path`**](https://github.com/systemd/systemd/pull/41838) — **merged.** nspawn failed with `EPERM` when a container needed both a user namespace and to join an already-existing network namespace: the inner child entered the new userns first, after which the kernel refused the `setns()` into a foreign netns. For Periapsis this is not an edge case but literally every pod — UID-mapped *and* joined to the netns the CNI already set up.
- [**nspawn: fix `.mstack` bind mounts being masked by `--volatile=overlay`**](https://github.com/systemd/systemd/pull/41897) — open, in review. Bind mounts silently disappeared under `--volatile=overlay`: overlayfs ignores active mount points inside the lowerdir, so the pod's mount stack — assembled from its image layers *before* the container starts — was masked by the overlay that went on top of it.

(Status as of 2026-08-05.)

---

## Related projects

- [virtual-kubelet](https://github.com/virtual-kubelet/virtual-kubelet) — the upstream fork base (Apache 2.0)
- [Constellation](https://github.com/malformed-c/constellation) — the Cilium fork providing the multi-pawn eBPF datapath

---

## License

Periapsis is licensed under the **Business Source License 1.1** — see [LICENSE-PLANNED.md](LICENSE-PLANNED.md) for the authoritative text (published ahead of the source release so the terms can be evaluated in full). Each release converts to **GPL-3.0-or-later** no later than four years after it ships (current releases: **2030-05-21**).

In practice (the BUSL text is authoritative; this is a summary):

- **Free for internal production and development** — individuals, homelabs, researchers, and enterprises may run, modify, fork, and deploy Periapsis for their own internal infrastructure (including internal cost-allocation / cross-charge). Running your own product or SaaS *on* Periapsis is internal use and is permitted; what's prohibited is selling Periapsis itself — hosting, orchestration, management — as the service.
- **No commercial hosting / SaaS (XaaS)**, **no commercial edge/IoT bundling**, and **no paid remote management (BYOC)** offered to third parties.
- **No exposing Periapsis's APIs or core orchestration to third parties over a network** as the service being sold.
- **Consulting is allowed** — traditional, manual, non-automated setup / troubleshooting on the end-user's own infrastructure.

For a commercial product outside these terms, contact Malformed C to discuss a commercial license.

Periapsis incorporates a fork of virtual-kubelet by the VK authors (Apache 2.0). Kubernetes is a trademark of The Linux Foundation.

`:: Malformed C ::`
