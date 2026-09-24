# PR #3144: Separate Workload/Supervisor Topology on OpenShift + Kata

**Result: It works. No blockers found in this configuration.** The new topology runs one Kata VM for the workload and a plain OCI container for the supervisor. Per-sandbox VM count stays at one.

**Tested configuration**: revision a8f98ec09 (main HEAD), custom initrd with veth/nftables modules, PR #3420 WAIT_KILLABLE_RECV fallback active, single-node OpenShift 4.22.13, Kata 3.31.0 via OSC 1.13.1.

**Date**: 2026-09-24
**Cluster**: virtlab2400, single-node OpenShift 4.22.13
**Kata**: OSC 1.13.1, kata-containers-3.31.0-7 with custom initrd (veth/nftables modules)
**Evaluation revision**: a8f98ec09 (main HEAD; the original PR #3144 pin dc47f155 lacks a kernel compatibility fix added by PR #3420)

---

## What Was Tested

PR #3144 changed the Kubernetes driver from a single combined pod (workload + supervisor in one container) to two separate pods. On this cluster, `defaultRuntimeClassName: kata` puts the workload pod into a Kata VM. The supervisor pod has no RuntimeClass set, so it runs under the cluster's default OCI runtime (crun).

## Placement Proof

Both pods inspected via `kubectl get pod -o json`:

| Pod | Runtime | SCC | Overhead |
|-----|---------|-----|----------|
| Workload (`default--placement-test`) | `runtimeClassName: kata` | restricted-v2 | 250m CPU / 350Mi memory |
| Supervisor (`os-supervisor-...`) | Not set (OCI) | restricted-v2 | None |

Confirmed on the host: one QEMU VM in `/run/vc/vm/`, mapped to the workload pod. `crictl pods` shows the supervisor running under the default runtime. The supervisor does not start a VM.

## What Works

**Command execution**: `openshell sandbox exec -- echo "exec-ok"` returns output and exit code correctly.

**Security inside the VM**: PID 1 is the `openshell-sandbox` binary with `NoNewPrivs=1`, `Seccomp=2` (filter mode), running as UID 1000830000 (OpenShift-assigned). Writing to /etc is blocked and writing to /tmp is allowed. The sandbox operates in LegacyReadOnly mode (kernel lacks WAIT_KILLABLE_RECV; PR #3420 adds the fallback).

Note: The write-to-/etc denial could be filesystem permissions (restricted-v2 SCC assigns a non-root UID), Landlock enforcement, or both. This test did not isolate which mechanism produced the denial.

**Network policy enforcement**: Tested with curl 8.21.0 (from `nicolaka/netshoot:latest`) inside the sandbox, correlated with supervisor OCSF logs.

| Test | Result | Supervisor Log |
|------|--------|---------------|
| HTTPS GET httpbin.org (allowed by policy) | HTTP 200 | `NET:OPEN ALLOWED; HTTP:GET ALLOWED` |
| HTTPS POST httpbin.org (read-only policy) | HTTP 403 + JSON body | `HTTP:POST DENIED [L7_REQUEST deny]` |
| DNS for api.github.com (not in policy) | curl exit 6: DNS refused | `DENIED api.github.com:53 [policy_dns_ineligible]` |
| TCP to 93.184.216.34 (arbitrary IP) | curl exit 7: Permission denied | `DENIED -> 93.184.216.34:80 [transparent_tcp_policy_denied]` |

L7 enforcement is clean: the supervisor MITMs TLS, inspects HTTP methods, and returns structured JSON errors with policy names.

**Lifecycle**: Create-to-first-exec takes 14.3 seconds (one sample, images cached). Deleting a healthy sandbox takes 36.6 seconds and removes both pods, the QEMU VM, virtiofsd, and the Kata shim.

**Supervisor loss with active traffic**: Ran a loop of HTTPS GET requests against httpbin.org. Requests 1 through 9 returned HTTP 200. The supervisor pod was killed at request ~10. The exec relay closed approximately 1 second later ("exec relay closed before the command reported an exit status"). The workload pod was removed and the sandbox transitioned to Stopped.

**Recovery after supervisor loss**: Running `sandbox start` on the Stopped sandbox brought it back to Ready in 9.6 seconds with a new workload pod and new supervisor pod. The workspace file `/sandbox/persist.txt` (written before supervisor loss) survived across the restart, confirming PVC-backed workspace persistence. Ephemeral state (`/tmp/marker.txt`) was gone as expected (new VM). HTTPS traffic restored immediately (HTTP 200).

## Issues Found

**OpenShift integration prerequisite**: The `kata` RuntimeClass needs the label `openshell.ai/sandbox-attachable=true` for resource admission (from PR #3538, not PR #3144). Apply with `oc label runtimeclass kata openshell.ai/sandbox-attachable=true`. This label survived OSC operator reconciliation during testing.

**Supervisor loss tears down the workload**: When the supervisor pod dies, the gateway destroys the workload pod and its VM. The sandbox transitions to Stopped. Running `sandbox start` restores the sandbox with a new VM while preserving the PVC-backed workspace. In-memory state and running processes are lost.

## Resource Cost (Idle, Single Sample)

All values measured via `/proc/<pid>/status` VmRSS on the host:

| Component | RSS (host /proc VmRSS) |
|-----------|------------------------|
| QEMU (guest configured at 2048MB) | 486 MB |
| virtiofsd (2 processes) | 26 MB |
| containerd-shim-kata-v2 | 47 MB |
| Supervisor pod | 18 MB |

RSS may double-count shared pages. The 350Mi scheduler overhead is a declared value, not measured consumption.

## Configuration Baseline

`disable_guest_seccomp=true` is set in `/etc/kata-containers/configuration.toml`. This means the Kata agent does not enforce container seccomp profiles inside the VM. The OpenShell sandbox runtime applies its own seccomp filters at PID 1, so sandboxed processes are still filtered. Processes started via `oc exec` bypass the sandbox runtime and run without the Kata agent's seccomp layer. Whether this matters depends on the threat model for `oc exec` access to Kata workload pods.

## Not Tested

- Comparison against the previous combined-pod topology (resource cost, latency, density)
- Two-VM variant (supervisor also under Kata)
- LegacyReadOnly specific call restrictions (getpeername, accept, sendmmsg)
- Warm-pool behavior
- Multiple concurrent sandboxes
- OpenShell e2e test suite with Kata runtime
