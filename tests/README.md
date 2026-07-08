# Periapsis Test Summary

This directory contains public test-surface notes only. It intentionally does not contain source code, test code, fixtures, private manifests, or internal implementation details.

## Scope

The private test suite exercises Periapsis as a Kubernetes node agent implementation and as a Linux host runtime. Publicly shareable coverage is summarized by behavior below.

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
- PVC-backed mounts through CSI-backed storage classes.
- Service account token projection and rotation behavior.
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
- Networked WASIp2 HTTP and Trail workload behavior.

## End-To-End Checks

Publicly shareable end-to-end scenarios include:

- Single-pawn host running Kubernetes pods.
- Multi-pawn host presenting many scheduler-visible nodes.
- High-replica lightweight workload placement.
- Pod churn with create/delete loops.
- Daemon restart while pods remain running.
- Cross-pawn service connectivity.
- Private or unavailable registry fallback through peer-held image content.

## Disclosure Boundary

The private repository contains the actual Go tests, integration harnesses, manifests, and host-specific scripts. Those are not mirrored here because this public repository is limited to public project information and public result summaries.
