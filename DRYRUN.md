# Dry-Run / Plan Contract

If a capability declares `dry_run: required|optional`, actions must support `dry_run: true`.

Return a **plan** with:

- `will_call`: list of API calls (method, path, masked params)
- `expected_effect`: structured diffs per target:
  - `target`: { type, id, name }
  - `from`: { state subset }
  - `to`: { state subset }
- `assumptions`: list of preconditions checked
- `risk`: `low | medium | high`
- `revert_hint`: optional human guidance
- `notes`: optional free text

On execute, include `trace_id`, `duration_ms`, `evidence` (request IDs/response codes), and final `result_state?`.