# Integration Manifest Spec

**File:** `plugin.yaml` (YAML 1.2)

## Top-level

- `id` (string, required): globally unique (reverse DNS style recommended).
- `name` (string, required)
- `version` (semver, required)
- `category` (enum): `host-orchestrator | network | smart-device | notifications | generic`
- `min_core_version` (semver range, required)
- `schema.connection` (JSON-Schema object, required): user-editable fields. Mark secrets with `secret: true`.
- `defaults` (object): 
  - `http`: `{ timeout_s, retries, backoff_ms_start, verify_tls, user_agent_suffix }`
  - `heartbeat_interval_s`
- `test` (object, required): how to validate connectivity & permissions.
  - `method`: `http` (v1)
  - `http`: `{ request: { method, path, headers?, body? }, success_when: <jsonpath or status code set> }`
- `capabilities` (array, required): capability declarations (see `CAPABILITIES.md`)
  - item: 
    - `id`: e.g., `vm.lifecycle`
    - `verbs`: list of supported verbs (e.g., `[shutdown, restart]`)
    - `targets`: list of supported target types (e.g., `[vm]`)
    - `dry_run`: `required | optional | unsupported`
    - `rate_limit`: optional `{ rps, burst }`
- `discovery` (object, optional): inventory listing.
  - `implements`: e.g., `inventory.list`
  - `interval_s`: default 600
- `errors` (optional): extend base taxonomy with subcodes.

### Instance model (runtime)

Instances are persisted by core, validated against `schema.connection`, and enriched with:
- `state`: `connected | degraded | error | unknown`
- `last_test`, `latency_ms`
- `overrides`: `{ http, heartbeat_interval_s, circuit_breaker }`
- `audit`: created/updated by, timestamps

See `instance.schema.json`.

## HTTP Standardization (v1)

- Retries: default 2, exponential backoff starting at 250ms (jittered), idempotent GET/HEAD by default; POST retry enabled only if manifest sets `retry_on_post: true`.
- Timeouts: default 5s.
- TLS: verify by default; instance may override.
- Headers: core injects `User-Agent: walnut/<core-version> (+<integration-id>)`.

## Heartbeats

- Core schedules heartbeats at `heartbeat_interval_s` (default 120).
- Result must map to `connected|degraded|error` with `latency_ms` and `error_code?`.

## Dry-run

- If `dry_run: required|optional`, actions **must** return a plan (see `DRYRUN.md`).
- Policies can require dry-run before execute.

## Circuit breaking

- Default: trip after 3 consecutive failures in 60s; cooldown 120s. Instance overrides allowed.