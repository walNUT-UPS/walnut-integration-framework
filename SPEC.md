# Integration Manifest Spec

**File:** `plugin.yaml` (YAML 1.2)

## Top-level Fields

- `id` (string, required): A globally unique identifier for the integration, typically in reverse-DNS format (e.g., `com.vendor.integration`).
- `name` (string, required): A human-readable name for the integration.
- `version` (semver, required): The version of the integration manifest.
- `min_core_version` (semver range, required): The minimum version of the core framework required to run this integration.
- `schema.connection` (JSON-Schema object, required): Defines the configuration fields required for an instance of this integration (e.g., hostname, credentials). These fields are defined by the integration developer and presented to the user. Mark sensitive fields with `secret: true`.
- `defaults` (object, optional): Default values for transport configurations and other behaviors.
  - `transports`: A container for transport-specific default settings. See the "Transports" section for details.
  - `heartbeat_interval_s`: Default interval in seconds for health checks.
- `test` (object, required): A definition for a connection test used to validate an instance's configuration and connectivity.
  - `method` (enum, required): The transport protocol to use for the test. Allowed values: `http` | `ssh` | `mqtt` | `websocket`.
  - `http` (object, optional): HTTP test configuration.
    - `request`: `{ method, path, headers?, body? }`
    - `success_when`: JSONPath expression or status code to evaluate success.
  - `ssh` (object, optional): SSH test configuration.
    - `commands`: `{ commands: [string], prompt_hint?, enable_password_ref? }`
  - `mqtt` (object, optional): MQTT test configuration.
    - `ping`: `{ broker, topic, payload?, expect_topic?, timeout_s? }`
  - `websocket` (object, optional): WebSocket test configuration.
    - `probe`: `{ url, send?, expect? }`
- `capabilities` (array, required): A list of capabilities this integration provides. See `CAPABILITIES.md`.
- `discovery` (object, optional): Configuration for automated inventory discovery.
- `errors` (object, optional): Extensions to the base error taxonomy. See `ERRORS.md`.

## Transports

Integrations can define default connection parameters for various transports. These can be overridden at the instance level.

### `defaults.transports.http`
- `timeout_s`, `retries`, `backoff_ms_start`, `verify_tls`, `user_agent_suffix`
- **Note**: The framework provides standard HTTP client features like retries (idempotent GET/HEAD by default), timeouts (default 5s), and TLS verification. Core injects a `User-Agent` header.

### `defaults.transports.ssh`
- `port`, `timeout_s`, `pty_required`, `key_exchange_algorithms`

### `defaults.transports.mqtt`
- `port`, `client_id_prefix`, `timeout_s`, `use_tls`

### `defaults.transports.websocket`
- `timeout_s`, `max_payload_size_kb`, `handshake_headers`

## Heartbeats

- Core schedules heartbeats at the configured `heartbeat_interval_s` (default 120).
- Heartbeat operations should be lightweight and read-only where possible.
- The result must map to a `state` (`connected`|`degraded`|`error`) and include `latency_ms` and an optional `error_code`.

## Dry-run

- If a capability has `dry_run: required` or `optional`, its actions must support returning a plan.
- The plan may contain steps across different transports (e.g., an SSH command and an HTTP request).
- For the detailed plan structure, see `DRYRUN.md`.

## Circuit Breaking

- Default: trip after 3 consecutive failures in 60s; cooldown 120s. Instance overrides allowed.