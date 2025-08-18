# Local SLAs & Telemetry (tracked in SQLite)

Per instance and capability:

- `success_rate_5m` (target ≥ 99%)
- `p95_latency_5m` budgets:
  - control actions: ≤ 2000 ms
  - read/query: ≤ 1000 ms
  - inventory sync: ≤ 5000 ms
- `heartbeat_uptime_24h` (target ≥ 99.5%)
- `error_budget_24h` (default 1%)

Store: counts of success/fail, latencies, last N errors, last heartbeat timestamp.