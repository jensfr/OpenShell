# Conflict Resolution Log

Integration: Gordon's warm pool commit (`3086d27a4`) cherry-picked onto main (`924486805`)
Result: `warm-pool-integrated` branch, commit `657483522`
Compilation: passes `cargo check` with 1 warning (unused `generations` parameter)

## Conflicts (14 files)

### 1. `architecture/gateway.md`
**Conflict:** Main updated JWT terminology; Gordon added `RegisterSupervisor` and runtime-assignment language.
**Resolution:** Merged both: main's generation-bound JWT text + Gordon's supervisor registration descriptions.

### 2. `architecture/sandbox.md`
**Conflict:** Main added HTTP/2 flow control content; Gordon added boundary connection timeout content.
**Resolution:** Included both sections.

### 3. `crates/openshell-core/src/jwt.rs`
**Conflict:** Gordon added `mint_bound`/`mint_bound_sandbox_token` methods; main changed `ttl` to `Option<Duration>`.
**Resolution:** Gordon's new methods adapted to use main's `Option<Duration>` ttl type.

### 4. `crates/openshell-driver-kubernetes/src/driver.rs` (most complex)
**Conflict:** 6+ conflict regions. Main refactored resource admission, added `generations` parameter to `delete_sandbox_runtime_supervisor` and `reconcile_sandbox_runtime_supervisor`, renamed `ANNOTATION_SANDBOX_RUNTIME_NETWORK_POLICY_VERSION` to `..._GENERATION`. Gordon added warm pool controller, pair management, template handling.
**Resolution:**
- Kept main's refactored function signatures (3-arg `delete_sandbox_runtime_supervisor` with `generations: &[&str]`, 4-arg `reconcile_sandbox_runtime_supervisor`)
- Gordon's warm pool call sites updated to pass `&[]` for `generations` (warm pool pairs don't use generation-based secret cleanup)
- Renamed constant: Gordon's `ANNOTATION_SANDBOX_RUNTIME_NETWORK_POLICY_VERSION` -> `..._GENERATION` to match main
- Added missing imports: `SANDBOX_SECRET_COMPONENT`, `SUPERVISOR_SECRET_COMPONENT` from `sandbox_runtime`
- Kept Gordon's `create_sandbox_inner(sandbox, None)` warm pool parameter pattern
- Kept Gordon's label-based cleanup via `warm_pool::create_owned()` wrapper

### 5. `crates/openshell-sandbox-backend/src/runtime.rs`
**Conflict:** Main added connection probe constants; Gordon added `CONNECT_ATTEMPT_TIMEOUT`.
**Resolution:** Kept both sets of constants.

### 6. `crates/openshell-server/src/auth/principal.rs`
**Conflict:** Gordon added `SupervisorRegistration` principal variant; main had surrounding changes.
**Resolution:** Added Gordon's variant into main's updated enum.

### 7. `crates/openshell-server/src/compute/mod.rs`
**Conflict:** Both sides added test functions in the same region.
**Resolution:** Kept both: main's `remote_compute_driver_rejects_policy_changes_before_forwarding` and Gordon's `warm_candidate_is_durable_before_claim_and_survives_uncertain_results`.

### 8. `crates/openshell-server/src/storage_proto.rs`
**Conflict:** Schema fingerprint hash changed on both sides (different proto changes).
**Resolution:** Used HEAD's (main's) `PUBLIC_RPC_SCHEMA_SHA256` hash. This will need regeneration if proto changes from Gordon's branch affect the hash. The proto schema test will catch this.

### 9-13. Helm files (`README.md`, `clusterrole.yaml`, `role.yaml`, `clusterrole_test.yaml`, `sandbox_namespace_test.yaml`)
**Conflict:** Main added resource admission RBAC rules; Gordon added warm pool CRD RBAC (sandboxtemplates, sandboxwarmpools), leases/coordination, extended secrets verbs.
**Resolution:** Included both sets of RBAC rules. Gordon's warm pool CRD verbs merged alongside main's existing rules.

### 14. `docs/how-it-works/gateways/configuration.mdx`
**Conflict:** File was renamed on main (from `docs/reference/gateway-config.mdx`). Gordon added warm pool configuration docs.
**Resolution:** Gordon's warm pool config section added to main's renamed file path.

## Post-merge compile fixes
- Added `SANDBOX_SECRET_COMPONENT`, `SUPERVISOR_SECRET_COMPONENT` to import list (line 15)
- Changed `ANNOTATION_SANDBOX_RUNTIME_NETWORK_POLICY_VERSION` -> `..._GENERATION` (line 2643)
- Added `&[]` generations argument to 4 warm pool call sites of `delete_sandbox_runtime_supervisor` and `reconcile_sandbox_runtime_supervisor` (lines 3235, 3427, 4115, 4176)

## Post-test fix (commit f06aa684d)
- `admit_stored_resources()` at line 1787: skipped `sandbox_id_from_object()` check for `warm_pool::is_pool_pair()` objects. Warm pool pairs have no logical sandbox ID until claimed; main's admission check rejected them during preparation.

## Code review fixes (found by source review after test pass)
- `delete_sandbox_runtime_generation_secrets()`: restored main's by-name deletion when `generations` is non-empty. The initial merge used Gordon's label-based cleanup unconditionally, ignoring the `generations` parameter. Now: non-empty `generations` deletes by exact name (main's behavior), empty `generations` falls back to label-based cleanup (warm pool path).
- `PUBLIC_RPC_SCHEMA_SHA256`: updated to `48f83cf1...` to reflect `RegisterSupervisor` and warm pool RPCs added by the integration. Durable and overlap hashes unchanged. Message/enum counts unchanged.
- Test compile fixes: `from_pem` TTL argument wrapped in `Some()` at 3 call sites (warm_pool.rs, mod.rs, supervisor_registration.rs). Added `warm_pair: None` to `CreateSandboxRequest` initializer in mod.rs.
