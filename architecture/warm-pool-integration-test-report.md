# Warm Pool Integration Test Report

PR #3508 integrated with main at 924486805.

## Test environment
- **Cluster:** virtlab2400 (OpenShift, RHEL 9.8, kernel 5.14.0-687.44.1.el9_8)
- **Branch:** `warm-pool-integrated` (PR #3508 cherry-picked onto main@924486805)
- **Integration commit:** `657483522`, fix commit `f06aa684d`
- **Gateway image:** `quay.io/jensfr/openshell-gateway:integrated-657483-fix1` (with admission fix)
- **Sandbox/supervisor images:** `quay.io/jensfr/openshell-sandbox:integrated-657483`, `quay.io/jensfr/openshell-supervisor:integrated-657483`
- **RuntimeClass:** `kata`
- **Namespace:** `pr3144-topology`
- **Helm release:** `openshell-pr3144` revision 15

## Integration bug found and fixed

**Problem:** Main's `admit_stored_resources()` (driver.rs:1787) calls `sandbox_id_from_object()`, which requires a logical sandbox ID. Gordon's warm pool pairs intentionally have no logical ID until claimed. The integration merged main's resource-admission call into the bootstrap path, causing warm pair preparation to reject every pair with "unassigned pool pair" and leaving all pods SchedulingGated.

**Fix:** Skip the `sandbox_id_from_object` check for warm pool pairs in `admit_stored_resources`. Pool pairs use a physical pair ID (`openshell.ai/warm-pair-id`); the logical sandbox ID is assigned on claim. All other admission checks (workspace, namespace, bootstrap secret, resource admission identities) still run.

**Attribution:** This bug was introduced by the integration, not by Gordon's PR or main individually. Gordon's branch does not have `admit_stored_resources` at all; main's version is correct for assigned sandboxes.

## Cold Kata baseline

- Created sandbox `cold-baseline` in ~11 seconds (image pull + init containers + startup)
- Exec works: `uname -a` returns RHEL 9.8 kernel
- WAIT_KILLABLE_RECV fallback (PR #3420) operational on this kernel. This fallback was needed on the tested kernel (5.14.0, which lacks the flag); integrating with main was the chosen way to obtain it.
- Sandbox pod runs under `restricted-v2` SCC with RuntimeDefault seccomp and all capabilities dropped

## Warm pool test

### Template creation
- Template `kata-warm` with `--ready-within 1s --max-burst 2`
- Two warm pairs created and reached `warm-pool-state=ready` within ~50 seconds
- Both sandbox and supervisor pods scheduled and running (supervisor intentionally not Kubernetes-Ready)

### Warm pair claim
- Claimed one warm pair: `openshell sandbox create --detach --name warm-claim --template kata-warm -- sh -c 'exec sleep infinity'`
- Warm create command completed in **1.073 seconds**, one sample
- Sandbox immediately Ready

### Pod reuse verification

**Pre-claim Pod UIDs:**
- Sandbox pod `warm-e0b7c3bde16211bd97f01b3b`: `82f7a048-7a70-4442-b757-ce4b1d74b696`
- Supervisor pod `os-supervisor-e0b7c3bde16211bd97f01b3b`: `c3b6cebd-c5b2-4885-8d9e-5a3768d5a45a`

**Post-claim Pod UIDs:**
- Sandbox pod: `82f7a048-7a70-4442-b757-ce4b1d74b696` (identical)
- Supervisor pod: `c3b6cebd-c5b2-4885-8d9e-5a3768d5a45a` (identical)

Hostname inside claimed sandbox: `warm-e0b7c3bde16211bd97f01b3b` (matches original pod name).

**Pod reuse confirmed; VM continuity not independently verified.** Neither CRI sandbox creation timestamps nor QEMU process start times were checked.

### Spare capacity replenishment
After claiming one pair, the warm pool controller created a new spare pair `warm-68e65b1c253af8110b032782`, which reached `pool-state=ready`. Two unassigned ready pairs restored as configured (`maxPairs: 2`).

## Summary

| Metric | Cold | Warm (1 sample) |
|--------|------|------------------|
| Create command wall time | ~11s | 1.073s |
| Pod reuse | N/A | Confirmed (UIDs identical) |
| VM continuity | N/A | Not independently verified |
| Spare replenishment | N/A | Confirmed |

Gordon's warm-pool design works in this Kata deployment after resolving the admission incompatibility introduced by integration with main.
