# Local SLAs & Telemetry

These metrics are tracked per instance and capability, regardless of the underlying transport protocol (HTTP, SSH, etc.).

- `success_rate_5m` (target ≥ 99%)
- `p95_latency_5m` budgets:
  - control actions: ≤ 2000 ms
  - read/query: ≤ 1000 ms
  - inventory sync: ≤ 5000 ms
- `heartbeat_uptime_24h` (target ≥ 99.5%)
- `error_budget_24h` (default 1%)

Core automatically stores counts of successes/failures, latency histograms, the last N errors, and the last heartbeat timestamp for each instance.