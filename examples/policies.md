# Policy Binding Example (Framework-Level)

> Policies are defined in the UI. This shows how the engine resolves them using capability + targets.

**Intent:** During outage, shut down low-priority PoE ports, then stop noncritical VMs.

Steps:
1. Select targets where:
   - type = `poe-port`
   - label `priority=low`
2. For each, call capability:
   - `network.poe` verb `disable`
3. Wait for confirmation events or 10s timeout.
4. Select targets where:
   - type = `vm`
   - labels include `tier=low`
5. Call `vm.lifecycle` verb `shutdown` with concurrency 5.
6. If dry-run is enabled, show plan diffs before execute.

The engine resolves each target to its owning **instance** and checks the instance advertises the capability with the verb for that target type.