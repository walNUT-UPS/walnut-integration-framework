# Capability Vocabulary

> Policies call instance actions via a `capability` + `verb` + `target`.

This document lists the core vocabulary. Integrations may define their own vendor-specific capabilities using a reverse-DNS naming convention (e.g., `com.aruba.switch.poe.priority`).

## Core Capabilities

### Power / Lifecycle
- `power.control` → targets: `host | outlet | poe-port`
  - verbs: `on | off | cycle`

### Virtualization
- `vm.lifecycle` → target: `vm`
  - verbs: `start | shutdown | stop | restart | suspend | resume`

### Network (PoE)
- `network.poe` → target: `poe-port`
  - verbs: `enable | disable`

### Notifications
- `notify.send` → target: `event | text`
  - verbs: `post`

## Guidance for Integrations
The framework does not define complex semantics such as PoE power priority or VLAN tagging. Integrations **may** implement these features through vendor-specific capabilities or by exposing rich attributes on targets. For example, an integration could expose a `priority` attribute on a `poe-port` target to enable "power off low-priority" policies.