# Error Taxonomy (base)

- `auth_error`
- `config_error`
- `network_error`
- `tls_error`
- `rate_limited`
- `not_found`
- `validation_error`
- `server_error`
- `unknown`

Integrations may extend with subcodes (e.g., `auth_error:insufficient_scope`), but **must** map to a base code.

## Circuit Breaking (defaults)
- Trip after 3 consecutive failures within 60s.
- Cooldown: 120s.
- Per-instance overrides: `failures`, `window_s`, `cooldown_s`.