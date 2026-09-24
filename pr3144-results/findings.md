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
- **Impact**: Running processes and in-memory state lost; workspace filesystem (kataShared mount) survives if backed by PVC

## Operational

### F2: Kata Initrd Cache Invalidation

- **Trigger**: RPM upgrade within same kernel version
- **Symptom**: `kata-osbuilder-generate.service` reports "Nothing to do"
- **Workaround**: Delete `/var/cache/kata-containers/osbuilder-images/<kernel-version>/` before running the service
- **Scope**: Affects any dracut config or module change that doesn't also change the kernel version

### F4: Workload Metrics via kubectl top

- **Observed**: `kubectl top pod` returns NotFound for Kata workload pods
- **Status**: Requires investigation (Kata 3.31 supports container statistics; may be a cluster metrics pipeline issue)

## Configuration Baseline

### F6: disable_guest_seccomp=true

- **Location**: `/etc/kata-containers/configuration.toml`
- **Effect**: Kata agent does not enforce container seccomp profiles inside the VM
- **OpenShell impact**: None (sandbox runtime applies its own seccomp filters at PID 1)
- **oc exec impact**: Processes started via oc exec bypass the sandbox runtime and run without seccomp or NoNewPrivs
