# PR #3144: Separate Workload/Supervisor Topology on OpenShift + Kata

**Result: It works. No blockers.** The new topology runs one Kata VM for the workload and a plain OCI container for the supervisor. Per-sandbox VM count stays at one.

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

**Security inside the VM**: PID 1 is the `openshell-sandbox` binary with `NoNewPrivs=1`, `Seccomp=2` (filter mode), running as UID 1000830000 (OpenShift-assigned). Landlock blocks writes to /etc, allows /tmp. The sandbox operates in LegacyReadOnly mode (kernel lacks WAIT_KILLABLE_RECV; PR #3420 adds the fallback).

**Network policy enforcement**: Tested with curl 8.21.0 inside the sandbox, correlated with supervisor OCSF logs.

| Test | Result | Supervisor Log |
|------|--------|---------------|
| HTTPS GET httpbin.org (allowed by policy) | HTTP 200 | `NET:OPEN ALLOWED; HTTP:GET ALLOWED` |
| HTTPS POST httpbin.org (read-only policy) | HTTP 403 + JSON body | `HTTP:POST DENIED [L7_REQUEST deny]` |
| DNS for api.github.com (not in policy) | curl exit 6: DNS refused | `DENIED api.github.com:53 [policy_dns_ineligible]` |
| TCP to 93.184.216.34 (arbitrary IP) | curl exit 7: Permission denied | `DENIED -> 93.184.216.34:80 [transparent_tcp_policy_denied]` |

L7 enforcement is clean: the supervisor MITMs TLS, inspects HTTP methods, and returns structured JSON errors with policy names. Each denial type produces a distinct error (DNS refused vs. TCP permission denied vs. HTTP 403).

**Lifecycle**: Create-to-first-exec takes 14.3 seconds (one sample, images cached). Deleting a healthy sandbox takes 36.6 seconds and removes both pods, the QEMU VM, virtiofsd, and the Kata shim.

## Issues Found

**OpenShift integration prerequisite**: The `kata` RuntimeClass needs the label `openshell.ai/sandbox-attachable=true` for resource admission (from PR #3538, not PR #3144). Apply with `oc label runtimeclass kata openshell.ai/sandbox-attachable=true`. This label survived OSC operator reconciliation during testing.

**Supervisor loss tears down the workload**: When the supervisor pod dies, the gateway destroys the workload pod and its VM. The sandbox transitions to Stopped. Recovery requires creating a new sandbox. This is an availability tradeoff of the separate-pod design.

**Kata initrd cache invalidation**: After upgrading the kata-containers RPM (same kernel version), the osbuilder service skipped regeneration because it only checks kernel version changes. Manual cache deletion was needed before the new dracut modules (veth, nf_tables) appeared in the initrd.

**Workload pod invisible to kubectl top**: Needs investigation. Kata 3.31 supports container statistics, so this may be a cluster metrics pipeline issue rather than a runtime limitation.

## Resource Cost (Idle, Single Sample)

| Component | RSS |
|-----------|-----|
| QEMU (guest configured at 2048MB) | 486 MB |
| virtiofsd (2 processes) | 26 MB |
| containerd-shim-kata-v2 | 47 MB |
| Supervisor pod | 18 MB |

RSS may double-count shared pages. The 350Mi scheduler overhead is a declared value, not measured consumption.

## Not Tested

- Comparison against the previous combined-pod topology (resource cost, latency, density)
- Two-VM variant (supervisor also under Kata)
- Supervisor interruption with active traffic in progress
- LegacyReadOnly specific call restrictions (getpeername, accept, sendmmsg)
- Warm-pool behavior
- Multiple concurrent sandboxes
