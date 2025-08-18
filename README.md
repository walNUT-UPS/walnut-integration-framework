# walNUT Integrations Framework

A **manifest-first** framework for defining external integrations in walNUT.

**What this is**
- A schema and vocabulary for **integration types** (e.g., Proxmox, AOS-CX).
- A model for **instances** (e.g., `pve-01`, `pve-02`) with their own creds, health, and metrics.
- A controlled set of **capabilities** (actions) with typed **targets**.
- Rules for **test connection**, **heartbeats**, **dry-run plans**, **errors**, and **SLAs**.

**What this is not**
- It is **not** vendor code. Manifests and drivers live elsewhere. This repo only defines the contract and templates.

## Concepts

- **Integration Type**: Declares connection schema, capabilities, tests, defaults. Distributed as a `plugin.yaml`.
- **Instance**: A concrete connection of a type with creds, endpoint, and health settings.
- **Capability**: A named, portable action surface (e.g., `vm.lifecycle: shutdown`).
- **Target**: The thing actions operate on (e.g., `vm`, `host`, `poe-port`).

See: `SPEC.md`, `CAPABILITIES.md`, `TARGETS.md`, `ERRORS.md`, `SLA.md`, `DRYRUN.md`.

## Quick start

- Use `templates/integration-template/` to scaffold a new integration type (no vendor specifics baked in).
- Validate manifests with `schema/manifest.schema.json`.
- Keep NUT/UPS handling in core; integrations are for “other stuff” (VM hosts, smart-home, switches, webhooks).