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
- WASIp2 HTTP workload throughput when using the Trail host runtime.

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
