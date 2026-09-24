# PR #3144 Topology Evaluation Findings

## Blockers

### F1: WAIT_KILLABLE_RECV Startup Blocker

- **Affects**: Original PR #3144 pin (dc47f155)
- **Resolved by**: PR #3420 (293fab75d)
- **Classification**: Upstream kernel gap
- **Impact**: Sandbox fails at startup on kernels < 5.19 without the fallback

## Integration Prerequisites

### F3: Resource Admission Label on RuntimeClass

- **Affects**: Any OpenShift cluster using OSC-managed RuntimeClasses
- **Attribution**: PR #3538 (resource admission feature)
- **Action**: `oc label runtimeclass kata openshell.ai/sandbox-attachable=true`
- **Risk**: Label may be removed by OSC operator upgrades (survived reconciliation in this test)

## Architecture Tradeoffs

### F5: Supervisor Loss Destroys Workload

- **Observed behavior**: Supervisor pod deletion triggers gateway to tear down the workload pod and Kata VM
- **Gateway log**: "sandbox-runtime supervisor is unavailable; suspending workload"
- **Phase transition**: Running -> Provisioning -> Stopped
- **Impact**: Running processes and in-memory state lost; workspace filesystem (PVC-backed /sandbox) survives
- **Recovery**: `sandbox start` restores with a new VM in ~9.6 seconds; workspace preserved, ephemeral state gone
- **Traffic behavior**: With active HTTPS requests, 9 requests succeeded (HTTP 200), relay closed ~1 second after supervisor kill

## Configuration Baseline

### F6: disable_guest_seccomp=true

- **Location**: `/etc/kata-containers/configuration.toml`
- **Effect**: Kata agent does not enforce container seccomp profiles inside the VM
- **OpenShell impact**: Sandboxed processes are still filtered (sandbox runtime applies its own seccomp at PID 1). Processes started via `oc exec` bypass the sandbox runtime and run without the Kata agent's seccomp layer. Whether this matters depends on the threat model for `oc exec` access to Kata workload pods.
