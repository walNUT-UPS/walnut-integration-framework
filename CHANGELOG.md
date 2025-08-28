# Changelog

All notable changes to the walNUT Integration Framework documentation and schemas will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Comprehensive GPT.md authoring guide aligned with orchestrator audit 1a3a090
- Updated manifest schema with current enforcement rules from orchestrator
- Driver skeleton template with Option A method signatures and documentation
- Example plugin.yaml template based on Proxmox VE integration pattern

### Changed
- GPT.md completely rewritten to encode current enforcement truth from orchestrator
- Manifest schema updated to reflect required fields: id, name, version, min_core_version, category, driver, schema, capabilities
- Schema now enforces driver.entrypoint regex pattern and dry_run enum values
- Updated capability definitions to document Option A mapping (id.replace('.', '_'))

### Documented
- Active port definition: link=="up" OR poe_power_w>0 OR poe_status=="delivering"
- Inventory target shapes for vm, stack_member, and port types
- Upload validation pipeline with exact error messages and HTTP status codes
- Type and instance status taxonomy with state transitions
- Secret handling: properties with secret:true stored in integration_secrets table
- Packaging requirements: .int extension, ZIP format, ≤10MB, no path traversal
- WebSocket event streams for upload jobs (integration_job.event, integration_job.done)

### Fixed
- Schema now correctly requires driver field for all integrations
- Removed test field from required array (test is optional)
- Added proper descriptions and examples throughout schema
- Corrected capability dry_run enum to match enforcement (required|optional|not_supported)

### Known Gaps
- Method signature validation not yet implemented (only method presence checked)
- Manifest requires section not defined or enforced
- Type removal does not propagate type_unavailable to instances
- Capability verb/target validation against driver method parameters not implemented

## Notes

This release aligns the integration framework documentation with the current orchestrator implementation as audited on commit 1a3a090 (2025-08-26). The documentation now serves as the authoritative source of truth for generating compliant integrations.

Integration authors can now reference GPT.md for complete guidance on manifest requirements, driver contracts, validation pipeline, and error handling without guesswork about current enforcement.