# Periapsis Test Summary

This directory contains public test-surface notes only. It intentionally does not contain source code, test code, fixtures, private manifests, or internal implementation details.

## Scope

The private test suite exercises Periapsis as a Kubernetes node agent implementation and as a Linux host runtime. Publicly shareable coverage is summarized by behavior below.

Coverage comes in tiers: unit tests (no root, no systemd, no cluster), integration tests (root plus a live systemd), an end-to-end suite against a real cluster and pawn node, and the upstream Kubernetes conformance suite described next.

## Upstream Conformance Suite

Beyond Periapsis's own tests, pod behavior is validated against the **upstream Kubernetes end-to-end node suite** — the same Ginkgo specs the Kubernetes project ships, run unmodified with the version-matched `v1.36` e2e binary against a real cluster, pointed at pawn nodes.

As of the 2026-07-25 snapshot, across all specs run to date:

| | |
| --- | --- |
| Distinct specs run | 752 |
| Passing | 657 |
| Actionable failures | 0 |
| Deferred (in scope, postponed) | 3 |
| Ignored (non-actionable) | 92 |

Every non-passing spec carries a written reason. The three deferred specs are a single CSI feature (per-pod service-account-token plumbing through `NodePublishVolume`) that is in scope and simply not on the current worklist. The 92 ignored specs fall into three groups: off-by-default feature gates the cluster under test does not enable; control-plane concerns that are not a node's responsibility (kubeadm bootstrap tokens, API-server storage semantics); and identity divergences — specs asserting the node is *literally* a kubelet, such as kubelet-branded metric names or kubelet-internal cgroup layout. Periapsis does not pretend to be kubelet, so those diverge by design.

Verdicts are regenerated from each run's JUnit output into a per-spec table (most recent run wins), so the tracker reflects the current build rather than a one-off screenshot. Run-level interrupts, suite timeouts, and flake retries are explicitly not scored as failures.

Notable spec families passing in full include in-place pod resize (KEP-1287) at 9/9 — app containers, native sidecars, running init containers, and rollback.

## Kubernetes Pod Lifecycle

Covered behavior:

- Pod create, update, delete, and terminal-state handling.
- Restart policies and CrashLoopBackOff behavior.
- Init containers and native sidecar behavior.
- Liveness, readiness, startup, TCP, HTTP, exec, and gRPC probe paths.
- postStart and preStop lifecycle hooks.
- Grace-period handling and forced termination at grace expiry.
- Pod phase, conditions, container statuses, exit codes, and termination messages.
- OOMKilled detection and reporting.
- Daemon restart recovery without stopping already-running pods.
- Ephemeral debug containers, including rejection of the unsupported PID-namespace targeting variant.
- Static pod handling: launch, structural and resources-only edits applied in place, stop/hold and resume, and exemption from reaping and idling.
- Readiness reported by the workload itself over the systemd notification socket.

## Resize And Scaling

Covered behavior:

- In-place CPU and memory resize of a running container without a restart.
- Restart-policy resize where a resource is marked as requiring one, including mixed pods that resize one resource in place while restarting for another.
- Resize of native sidecars and of running init containers.
- Pod-level resources and pod-level resize.
- Enacted-value reporting in pod status and resize completion events.
- Rejection and rollback paths for invalid resize requests.
- Live pawn add, resize, removal with drain, and reconcile-to-count scaling.

## Idle, Wake, And Freeze

Covered behavior:

- Idling a pod so its memory is reclaimed while its network namespace, address, and rootfs are retained.
- Waking an idled pod, both on an incoming connection and on explicit operator command.
- Freezing and thawing a pod's existing processes, as distinct from a cold start.
- Wall-time and compute budgets that force a pod idle once consumed.
- Safety rules that keep control-plane and other exempt pods from being idled.

## Kubelet API Compatibility

Covered behavior:

- `kubectl logs`, including previous container logs where applicable.
- `kubectl exec`, including stdin-capable commands.
- `kubectl attach`.
- `kubectl port-forward`.
- Metrics endpoint behavior used by Kubernetes tooling.
- Event publication for runtime and configuration failures.

## Security And Admission Behavior

Covered behavior:

- `runAsUser`, `runAsGroup`, `runAsNonRoot`, `fsGroup`, and supplemental groups.
- Capability add/drop behavior.
- No-new-privileges behavior.
- Seccomp handling and explicit rejection for unsupported profile modes.
- Read-only root filesystem behavior.
- Privileged-mode handling.
- Sysctl acceptance/rejection according to host namespace support.
- Fail-closed rejection where a Kubernetes field cannot be implemented faithfully.

## Resources And Isolation

Covered behavior:

- cgroups v2 memory, CPU, IO, PID, etc controls.
- Per-pod resource limits.
- Per-pawn capacity budgets.
- Host-level budget protection for co-located services.
- Node-pressure detection and eviction ordering.
- Disk-pressure image and layer cleanup paths.

## Networking

Covered behavior:

- Pod IP allocation and network namespace setup.
- Standard CNI operation for one-host/one-node deployments.
- Multi-pawn pod networking.
- Pod-to-pod connectivity on the same host (multi-pawn) and across hosts.
- ClusterIP service connectivity.
- DNS policy and DNS config behavior.
- Managed `/etc/hosts` and host aliases.
- NetworkPolicy behavior in policy-capable deployments.

## Storage And Volumes

Covered behavior:

- ConfigMap, Secret, projected, downward API, emptyDir, memory emptyDir, and hostPath volumes.
- PVC-backed mounts through CSI-backed storage classes, including node volume expansion.
- Service account token projection and rotation behavior, including CSI-requested tokens.
- OCI images mounted as read-only volumes rather than run as containers.
- Volume ownership and filesystem-group handling.
- Cleanup of pod-scoped mount state.

## Image Handling

Covered behavior:

- OCI image resolution and pull behavior.
- Pull policy behavior.
- Digest verification.
- ImagePullSecret handling.
- Shared layer storage and overlay mounting.
- Local image ingestion.
- Peer-assisted manifest and layer distribution.
- Cache cleanup and pinned-image behavior.

## WebAssembly

Covered behavior:

- Runtime-class selection for WASM (Trail) workloads.
- Host-runtime launch behavior when a supported runtime is installed.
- Fail-closed behavior when a requested runtime is unavailable.
- Networked WASIp2/p3 HTTP and Trail workload behavior.
- Capability profiles: deny-by-default host access, per-capability grants, node-wide defaults and ceilings, and fail-closed behavior for an un-granted import.
- Component composition tiers — fused at launch, plugged in-process with hot-swap, or bound to a provider on another node — including transitions between them.
- Cooperative checkpoint and restore, including state surviving a coordinated restart.
- Cross-node and cross-architecture pod migration with in-memory state preserved.

## End-To-End Checks

Publicly shareable end-to-end scenarios include:

- Single-pawn host running Kubernetes pods.
- Multi-pawn host presenting many scheduler-visible nodes.
- High-replica lightweight workload placement.
- Pod churn with create/delete loops.
- Daemon restart while pods remain running, including under sustained HTTP load.
- Cross-pawn service connectivity.
- Private or unavailable registry fallback through peer-held image content.
- Cold bootstrap of a self-hosted control plane on an unprepared host, warm start against an already-running control plane, and recovery across a host reboot.
- Control-plane outage and cutover handling, including recovery once the API server returns.
- Migration of a stateful component pod between nodes of different CPU architectures.
- Loading a local artifact into the cluster and serving traffic from it with no registry involved.

## Disclosure Boundary

The private repository contains the actual Go tests, integration harnesses, manifests, and host-specific scripts. Those are not mirrored here because this public repository is limited to public project information and public result summaries.
