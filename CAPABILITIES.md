# Controlled Capability Vocabulary (v0)

> Policies call instance actions via capability + verb + target.

## Power / lifecycle
- `power.control` → targets: `host | outlet | poe-port`
  - verbs: `on | off | cycle`

## Virtualization
- `vm.lifecycle` → target: `vm`
  - verbs: `start | shutdown | stop | restart | suspend | resume`

## Network (PoE)
- `network.poe` → target: `poe-port`
  - verbs: `enable | disable`
  - notes: integrations **should** expose port priority (`high|normal|low`) via discovery or attributes to enable “power off low-priority” policies.

## Notifications
- `notify.send` → target: `event | text`
  - verbs: `post`

## Generic HTTP (optional for power users)
- `http.request` → target: `webhook`
  - verbs: `get | post | put | patch | delete`

> Anything not listed here is out-of-scope for v1 (e.g., VLAN ops, storage arrays/datasets).