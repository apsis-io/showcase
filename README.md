# Periapsis

### From the Seas to the Stars — Kubernetes' Pawn

**A high-density Kubernetes node agent that runs pods as `systemd-nspawn` machines, not CRI containers.**

Periapsis is a fork of [virtual-kubelet](https://github.com/virtual-kubelet/virtual-kubelet) that absorbs the full **perigeos** stack: a Kubernetes node agent that bypasses the CRI and containerd entirely and runs pods directly on a Linux host as `systemd-nspawn` machines (with host-process and host-runtime WASM launch modes alongside). A single physical host registers as many independent virtual nodes — called **pawns** — each with its own TLS identity, kubelet API endpoint, pod CIDR, and cgroup slice. The scheduler treats them as separate nodes; the workloads stay native units on the host.

This public repository is intentionally information-only. It contains public project material, test-surface summaries, and benchmark summaries. It does not contain source code, ADRs, or operational secrets (for now, expect a release in around 2 to 3 months).

```text
[engi@engix99 ~]$ sudo apsis status
Hostname:    engix99
Pawns:       30
Pods:        287
RSS:         177 MiB
Machines:    287
Netns:       298

[engi@engix99 ~]$ kubectl get nodes
NAME               STATUS   ROLES                   AGE   VERSION
compute-00         Ready    pawn                    42d   perigeos://dev
...
compute-29         Ready    pawn                    42d   perigeos://dev
engix99            Ready    control-plane,primary   44d   v1.35.2+k3s1

[engi@engix99 ~]$ machinectl | head -2
MACHINE                                        CLASS     SERVICE        OS     VERSION
pod-01922ac2-…-nginx                           container systemd-nspawn alpine 3.21.3
pod-021aa7a8-…-nginx                           container systemd-nspawn alpine 3.21.3
```

---

## Kubernetes is not kubelet + CRI

Kubernetes is the orchestration API and object model: the API server, the scheduler, controllers, and objects like Node, Pod, Service, and Deployment that describe *what* should run and *where*. None of that changes with Periapsis — a pawn is a normal Node object, a pod is a normal Pod object, and standard `kubectl`, controllers, and the scheduler all work unmodified.

What Kubernetes has historically delegated to each node is a **kubelet** (the per-node agent that watches assigned pods and makes them real) talking to a **CRI runtime** (containerd, CRI-O) over the Container Runtime Interface to actually create containers. That kubelet+CRI pairing is one implementation of "make pods real on a Linux host" — it is not Kubernetes itself, and it is not the only way to satisfy the node contract.

Periapsis replaces that one pairing, not Kubernetes. It speaks the same node-facing contracts (the kubelet HTTP API, the pod lifecycle semantics) but skips CRI/containerd entirely, running pods as native `systemd-nspawn` units instead of OCI containers behind a shim. From the API server down to the scheduler, nothing changes; the substitution happens below the node boundary, not above it.

Periapsis can also go a step further and bootstrap the control plane itself: `perigeos` self-hosts the apiserver, controller-manager, and scheduler as static pods, cold-starting them from on-disk static manifests (the same kind `kubeadm` generates) and handing off to normal reconciliation — with no k3s or kubeadm running as the ongoing orchestrator. This doesn't change the boundary above — it's still a standard API server, standard objects, standard scheduler — it just means Periapsis can stand up and keep running that side too, or you can keep running k3s/kubeadm in front of Periapsis nodes as before.

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

See [benches/README.md](benches/README.md) for the public benchmark notes, including the 1,772-pod density/throughput result.

---

## Use cases

Periapsis fits trusted, density-first deployments where the CRI / microVM tax is the bottleneck.

- **A real multi-node cluster on one machine** — instead of `minikube` / `kind` / `k3d` (a single node, or nodes faked as nested containers), one host registers as 30+ pawns, so you can exercise what a single-node dev cluster can't: scheduling and (anti-)affinity, topology spread, `NetworkPolicy` and Services *across nodes*, PodDisruptionBudgets, and node cordon/drain — on a laptop or one server.
- **High-density CI/CD and batch** — thousands of short-lived build/test pods with no containerd shim per pod; a pod is a transient `systemd-nspawn` unit, so per-pod overhead is sub-MB and churn is cheap.
- **Edge & small VPS** — the stock kubelet+containerd stack is heavy before any workload runs; a single-pawn Periapsis daemon idles in the 70–130 MB range and leaves the box for the workload (plus the WASM (WASIp2 components) `runtimeClass` path for tiny, sandboxed edge workloads).
- **Homelab / bare-metal saturation** — fill a big Xeon/Threadripper box past the 110-pod cap without underutilising it or adding the KubeVirt/Kata microVM latency and memory tax. Co-exists with an existing k3s/kubelet node on the same host.

It is the **wrong** tool for hostile multi-tenancy: pods share the host kernel, so untrusted code that can exploit a kernel bug is out of scope — reach for a microVM runtime there.

**Terminology:** a *host* (or *machine*, in the `apsis status` / `machinectl` sense — there a "machine" is a running pod, not a host) is not the same thing as a Kubernetes *node* (a *pawn*, in Periapsis terms). One physical host can present many pawns: one daemon process, many scheduler-visible nodes.

**More pawns does not mean a bigger blast radius, and it does not buy high availability.** Every pawn on a host shares that host's kernel, hardware, and power supply — pawns are not independent failure domains. Slicing one host into more pawns changes density and scheduler-visible node count; it does not add or remove an isolation boundary, and it does not make the host itself redundant. If the host goes down, every pawn and every pod on it goes down together, regardless of how many pawns it was sliced into. High availability still comes from spreading workloads across separate physical **hosts** (with anti-affinity / topology spread), the same as it would with stock kubelet nodes.

---

## Key features

**Pod lifecycle (kubelet-conformant semantics)**
- Create / update / delete; restart policies; CrashLoopBackOff (systemd owns the restart, the FSM observes).
- Init containers, native sidecars (init w/ `restartPolicy: Always`), and liveness / readiness / startup probes (exec, httpGet, tcpSocket, grpc).
- postStart / preStop lifecycle hooks with a shared grace budget and a hard SIGKILL-at-grace cap.
- Pod phases, conditions, `terminationMessage`, OOMKilled detection (including nested-cgroup payload OOMs), signal exit codes.
- Restart durability: the reconciler's plan position is snapshotted and restored, so a running pod survives a daemon restart with no `Pending` flap.

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
- Per-pod PID cap / fork-bomb containment (`periapsis.io/max-pids`).
- Node-pressure eviction ranked kubelet-style (QoS → priority → usage); disk-pressure image/layer GC before pod eviction.
- Per-pawn slice caps and a per-host budget cap, leaving headroom for a co-located real kubelet. **Dynamic pawn scaling** from the budget: `apsis pawns scale <set> <count>` reconciles a pawn set to a target count, sizing each pawn to an even share of the host budget.

**WASM (Trail) workloads** — a pod with `runtimeClassName: trail` runs a WASIp2/p3 **component** via the **host-runtime path**: a transient unit joined to the pod netns, `wasi:sockets` bound on the pod IP (a WASM runtimeClass with no installed runtime fails closed). Bare wasip1 CLI runtimes (running a raw core module with no component model, no host-capability profile, no checkpoint/restore) are deliberately not supported in favor of Trail's component path; `trail` is the only WASM runtimeClass today (earlier separate `wasmtime`/`wasmedge`/`wasmer` classes have been consolidated into it).

```text
$ kubectl logs hello-wasm                  # runtimeClassName: trail, a WASIp2 component
hello from wasm
```

**Networking** — a Cilium-based CNI fork ("Constellation") for the multi-pawn case: eBPF datapath, VXLAN cross-host routing, per-pod netns. Standard CNI (`/etc/cni/net.d`) covers standalone 1:1 host-to-node deployments. Cluster DNS, `dnsConfig`/`dnsPolicy`, managed `/etc/hosts` + `hostAliases`. Pod-to-pod (same/cross host), ClusterIP Services across hosts, NetworkPolicy, and L7 ingress all work with pawn-hosted workloads.

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

Pull policies, `imagePullSecrets`, and digest verification are supported. **`apsis ingest`** loads an OCI/docker image tar straight into a node's library — served P2P, usable by pods, pinned against GC — so an image reaches the cluster with no registry. For WebAssembly, it also accepts a raw `.wasm` or `.wat` program and packs it as a minimal WASM OCI image automatically.

The layer cache is **garbage-collected** automatically: under disk pressure perigeos reclaims orphaned layers, then evicts whole unused (not running, not pinned) images LRU before falling back to evicting pods — plus `apsis cleanup` / `apsis images rm` on demand.

**Storage** — `configMap`, `secret`, `projected` (incl. rotated serviceAccountToken), `emptyDir` (incl. `medium: Memory`), `downwardAPI`, `hostPath`, and PVC/CSI.

**Operability**
- Kubelet HTTP API: `exec` (incl. stdin), `attach`, `logs` (incl. `--previous`), `port-forward`.
- Native `journald` logs and `machinectl` visibility — operators inspect pods with the same tooling they already use.
- Zero-downtime daemon upgrades (`KillMode=process`): restarting/upgrading perigeos does not stop the pods; it rediscovers them and resumes reconciling.
- The **`apsis`** CLI: `status`, `doctor`, `pods`, `top`, `cleanup`, `images` (list / `rm` / `verify`), `ingest` (load an image into the node library), `pawns scale` (dynamic scaling), `rollout`.

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

## Build & test

The commands below describe the private source tree's build and test flow; they aren't runnable from this repository, which contains no source. See [tests/README.md](tests/README.md) for the public test-surface summary.

```bash
# Build the daemon (and the apsis CLI)
go build ./cmd/perigeos
make build              # build-perigeos + build-apsis with version stamping
make build-wasm-samples # build the sample WASIp2 components

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

- systemd v250+ (v260+ recommended) and cgroups v2
- Linux kernel 6.6+ (eBPF features used by the multi-pawn CNI path)
- Kubernetes 1.34+
- Go 1.26+ to build
- Optional: the Constellation CNI fork for cross-host networking and multi-pawn isolation; without it, pods use host-network veth bridges (single-pawn only)
- Optional: kernel arg `swapaccount=1` for swap enforcement

---

## Naming

| Name | Role |
|------|------|
| **Periapsis** | The project / repository (closest orbital approach) |
| **Perigeos** | The daemon binary (Earth-specific periapsis) |
| **Pawn** | A virtual Kubernetes node — wordplay on systemd-ns**pawn** |
| **Scout** | The `PodLifecycleHandler` seam bridging the VK framework to the reconciler |
| **Recon** | The sharded reconciler (Foci shards, Groundfall signal store, PodSM) |
| **Clade** | The pure FSM + status kernel driving each container's state |
| **Horizon** | The sole pod-status + event writer to the API server |
| **Constellation** | The Cilium fork for multi-pawn networking |
| **Trail** | The WASIp2 component runtime path |
| **Apsis** | The CLI for introspection and debugging |

---

## Related projects

- [virtual-kubelet](https://github.com/virtual-kubelet/virtual-kubelet) — the upstream fork base (Apache 2.0)

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
