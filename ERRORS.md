# Error Taxonomy

## Base Error Codes
- `auth_error`: Authentication failure (e.g., invalid credentials, expired token).
- `config_error`: Invalid configuration in the instance schema.
- `network_error`: Connectivity issue (e.g., DNS failure, connection refused).
- `tls_error`: TLS/SSL handshake or certificate validation error.
- `rate_limited`: The request was denied due to a rate limit.
- `not_found`: The requested resource does not exist.
- `validation_error`: The input provided by the user or policy is invalid.
- `server_error`: The remote server encountered an internal error.
- `unknown`: An unspecified error occurred.

Integrations may extend these with subcodes (e.g., `auth_error:insufficient_scope`), but **must** map to a base code.

## Transport Error Mapping

Here are some examples of how typical transport-level failures should be mapped to the base error codes:

| Transport | Failure Scenario | Recommended Code |
| :--- | :--- | :--- |
| **HTTP** | `401 Unauthorized` | `auth_error` |
| | `403 Forbidden` | `auth_error` |
| | `404 Not Found` | `not_found` |
| | `500 Internal Server Error`| `server_error` |
| **SSH** | Authentication failure | `auth_error` |
| | Host key mismatch | `tls_error` |
| | Connection timeout | `network_error` |
| **MQTT** | Broker unavailable | `network_error` |
| | Not authorized to publish/subscribe | `auth_error` |
| **WebSocket**| TLS handshake failure | `tls_error` |
| | Connection refused | `network_error` |


## Circuit Breaking (defaults)
- Trip after 3 consecutive failures within 60s.
- Cooldown: 120s.
- Per-instance overrides: `failures`, `window_s`, `cooldown_s`.