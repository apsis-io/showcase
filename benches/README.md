# Periapsis Benchmark Summary

This directory contains public benchmark notes only. It intentionally does not contain source code, benchmark harness code, private manifests, or internal implementation details.

## What Is Measured

Periapsis benchmarks focus on density and host overhead rather than synthetic microbenchmarks alone:

- Idle daemon memory with one pawn.
- Idle daemon memory with many pawns in one process.
- Per-pod host overhead for lightweight pods.
- Scheduler-visible pod density on a single physical host.
- Pod create/delete churn.
- Daemon restart behavior while pods continue running.
- Image pull and peer-assisted distribution behavior.
- WASIp2/p3 HTTP workload throughput when using the Trail host runtime.
- Idle-to-wake latency for scale-to-zero pods.
- Traffic continuity across a daemon restart under load.
- Time from a local artifact to a pod serving real traffic.

## Representative Results

The following numbers are public snapshot figures from development and validation runs. They are not a universal guarantee; host kernel, image mix, workload behavior, storage, CNI, and cluster configuration all matter.

| Area | Representative result |
| --- | --- |
| Daemon idle RSS | around 70–130 MiB, depending on pawn count (67.7 MiB at 2 pawns, 71.6 MiB at 5 pawns, measured 2026-07-06) |
| Live 30-pawn host with 287 pods | around 177 MiB daemon RSS |
| Per-pod process tax target | sub-MiB daemon-side overhead |
| Scheduler-visible density target | thousands of pods per host |
| Validated density scenario | 1,772 lightweight pods across hosts |
| WASIX-era HTTP sample (2026-06-21, on the runtime class since consolidated into Trail) | about 14k requests/sec in one public stress run |
| Small-fleet HTTP latency (2026-07-06, 7 nodes / 21 pods, 2,910 req/s) | p95 1.36 ms, p99 sub-millisecond, 0% errors |

The small-fleet latency figure is a fresh data point, not a direct comparison to the WASIX-era throughput sample above — different fleet size, workload, request rate, and (by now) a different runtime class entirely. It shouldn't be read as an improvement claim over any prior measurement.

## Scale-To-Zero Wake Latency (2026-07-06)

An idled ("sunset") pod keeps its network namespace, address, and overlay rootfs while its memory is reclaimed, so waking it is a warm start rather than a fresh schedule. Measured over 9 clean idle/wake cycles on one pawn, driven by an explicit wake command and timed both externally around that call and by the daemon's own self-reported wake time (the two agreed within ~30 ms every cycle):

| Metric | Result |
| --- | --- |
| Wake latency (idled → serving again) | **1.43–1.58 s**, 9 samples, tight |
| Idle → confirmed stopped | 5.15–5.34 s, bounded by the fixture's 5 s termination grace period |
| Cold start for comparison (delete → replacement pod Ready) | bimodal, ~2.0 s or ~5.3–5.7 s, 5 samples |

Honest caveats: small sample sizes — enough for an order of magnitude and to confirm consistency, not a rigorous distribution. All samples are same-node; no cross-node numbers were taken. The cold-start bimodality was not root-caused; the plausible-but-unverified guess is network-resource reuse waiting on the old pod's full teardown. The idle-to-stopped figure is dominated by the test workload ignoring `SIGTERM`, not by Periapsis. And this is the **wake operation**, not an end-to-end client experience: a wake triggered by real traffic through the eBPF path adds the client's TCP retransmit boundary (≥1 s) on top, because the SYN that triggers the wake is dropped rather than held.

The freezer tier (process suspended in place rather than stopped) was not separately benchmarked — structurally there is little to measure there beyond a write to `cgroupfs` and a reconciler pass — and no comparison figure was taken against any other scale-to-zero implementation; see the `vs Knative` note in the main README for why none is quoted.

## Daemon Restart Under Load (2026-07-08)

Because pods are `systemd-nspawn` machines rather than children of the node agent, restarting or upgrading the daemon should not touch running workloads. Measured against a 3-replica nginx Deployment behind a Service, with the daemon restarted mid-flood:

| Metric | Result |
| --- | --- |
| Load at restart | 300 concurrent VUs, ~1,490 req/s steady state |
| Completed iterations | 107,631 |
| Interrupted iterations (real failures) | **0** |
| Pod container restarts | **0** |
| Pod readiness across the restart | stayed Ready throughout |
| Daemon restart window | ~5 s |

Two caveats worth stating plainly. First, this is a single-host result and the run was stopped early (~82 s) once the outcome was unambiguous — it is not a long-soak number. Second, an earlier round of the same test at light load (20 VUs) reported "0 interrupted iterations" and was **misleading**: the restart had silently broken the pods' readiness probes, and the symptom simply hadn't surfaced yet within the short observation window. That bug — a probe path reading a pod IP from an in-memory cache that did not survive a process restart, with no fallback to the pod's recorded status IP — was found, fixed, and unit-tested; the three affected pods recovered on their own within seconds of the fixed binary starting, with no restarts or recreation. The clean result above is from after the fix. The light-load round is kept in the record as a cautionary example of a benchmark that measured the wrong thing.

## Local Artifact To Serving Traffic (2026-07-22)

Because a node can ingest an artifact directly into its library and serve it peer-to-peer, there is a path from a file on disk to a running workload that involves no registry, no image build, and no CI. Measured with a wall clock on each leg, two consecutive runs, for a 3.4 MB WASM component:

| Leg | Run A | Run B |
| --- | --- | --- |
| Ingest (raw component → cluster-available, pinned, P2P-served) | 0.13 s | 0.13 s |
| Manifest apply | 0.22 s | 0.22 s |
| Apply → pod Running | 1.10 s | 1.04 s |
| Running → first HTTP 200 with a real page body | 0.11 s | 0.11 s |
| **Total: local file → serving traffic** | **1.57 s** | **1.51 s** |

This is a small artifact on a warm host, and the response body was verified to be the application's real page rather than a probe artifact. It measures the distribution and launch path, not application startup for a heavyweight service.

## Cross-Architecture Migration (2026-07-09)

A checkpoint-capable WASM component pod was migrated from an amd64 node to an arm64 node with its in-memory state intact and no manual intervention: the target resolved and fetched the component image peer-to-peer, restored the checkpoint, and resumed with its counter continuing rather than resetting. This is a capability result rather than a timing benchmark; no latency distribution was collected.

## Resource Consumption Snapshot (Small Deployment)

A real small-footprint deployment — 1 vCPU (virtualized AMD EPYC-Milan), 1.9 GiB RAM, 29 GiB disk, no swap — measured 2026-07-07 post-deployment, running perigeos with its self-hosted control plane (apiserver, controller-manager, scheduler) plus kube-proxy, CoreDNS, and a sample workload:

| Component | Memory (current) | Memory (peak) | CPU (measured avg) |
| --- | --- | --- | --- |
| perigeos daemon itself | ~102 MiB | ~119 MiB | ~2.7% of one core (30s window) |
| Everything perigeos manages (self-hosted control plane + kube-proxy + CoreDNS + sample workload) | ~421 MiB | — | ~4.3% of one core (20s window) |
| Host total | ~794 MiB / 1.9 GiB used (~42%) | — | — |

perigeos plus its entire managed workload together: ~523 MiB, ~7% of the single core. Disk usage was 20% (5.6 GiB / 29 GiB). This host had substantial headroom left (~1.1 GiB free memory, CPU essentially idle) for additional workloads.

## Extreme Small-End: Raspberry Pi Zero 2 W

The other end of the spectrum from the density runs above: the same `perigeos` binary, unmodified, on a ~$15 board — 467 MiB RAM, arm64, stock Raspberry Pi OS kernel 6.1.21-v8+. It registers as a node, and a normal alpine pod is created, pulls its image (layer cache hit), and runs, with ordinary `kubectl describe` events — measured 2026-07-08:

```text
$ sudo apsis status
Hostname:    engipi
Pawns:       1
Pods:        2
Kernel:      6.1.21-v8+
Arch:        linux/arm64
Memory:      177 / 467 MiB
RSS:         59 MiB

$ free -h
               total        used        free      shared  buff/cache   available
Mem:           467Mi       172Mi        87Mi       240Ki   265Mi       294Mi
```

59 MiB daemon RSS; the whole host with the pod running is ~172 MiB of 467, leaving roughly 294 MiB available. A stock kubelet + containerd pair would not fit alongside a workload on a board this small. Honest caveat: the pod above is on host networking — the multi-pawn CNI path (Constellation) needs kernel 6.6+ for its eBPF datapath, and this board's stock kernel is 6.1; the single-pawn, host-network path is what's verified here. The `go1.26.5` build is a plain cross-compile, nothing arm64-special.

## Density Run Shape

A representative density run uses one physical host split into many scheduler-visible pawns. Kubernetes places pods across those logical nodes, while the host still supervises workloads as native host-managed workloads.

```text
physical host
  -> 30 pawn nodes
  -> hundreds to thousands of lightweight pods
  -> one shared Periapsis daemon process
  -> cgroups v2 resource containment
```

## Benchmark Interpretation

The important comparison is the removed per-container shim and CRI daemon path. Periapsis does not make workload memory disappear; application RSS still dominates for real services. The gain is in the node/runtime tax around many small or short-lived pods.

This makes the benchmark story strongest for:

- CI and batch fleets with many short-lived pods.
- Dense trusted internal services.
- Edge hosts and small VPS deployments where idle overhead matters.
- Homelab and bare-metal machines that hit pod-count limits before hardware limits.
- Local multi-node Kubernetes simulation on one host.

## Public Reproduction Notes

A public reproduction requires a compatible Linux host with cgroups v2, Kubernetes access, compatible CNI setup, and the private Periapsis build. This repository does not publish the build artifacts or source code.

When a public source or binary release exists, this directory can grow into a reproducible benchmark suite. Until then, it remains a public summary of measured behavior and benchmark intent.
