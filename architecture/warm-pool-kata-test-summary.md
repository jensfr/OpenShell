# Warm-pool functionality with Kata — PASS

Tested on 25 September 2026 using OpenShift 4.22.13 and Kata 3.31.0.

## Configuration

New OpenShell topology: workload Pod uses `runtimeClassName: kata`; supervisor runs as a regular OCI Pod.

Kata guest initrd includes the required networking modules, including veth and nftables support.

PR #3508 (`3086d27a4`) integrated with main (`924486805`), with admission fix `f06aa684d`.

## Checks performed

Created a cold sandbox and successfully executed a command.

Created template `kata-warm` with `--ready-within 1s --max-burst 2`. Two prepared pairs reached `openshell.ai/warm-pool-state=ready`.

Created a sandbox from the template. It became Ready, command execution worked, and the workload and supervisor Pod UIDs remained unchanged.

Confirmed automatic replenishment back to two ready, unassigned pairs.

## Result

Basic warm-pool preparation, claiming, command execution, Pod reuse, and replenishment work with Kata and the new topology.

The integrated build needed an admission adjustment because unclaimed warm pairs do not yet have a logical sandbox ID. The result applies to that patched build. Pod reuse was verified; continuity of the underlying Kata VM was not independently checked.
