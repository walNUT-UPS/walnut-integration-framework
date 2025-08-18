# Target Model (v1)

Targets are the things actions operate on.

Types (v1): `vm, host, poe-port, webhook, outlet, service`

Minimal shape:
```yaml
type: vm | host | poe-port | webhook | outlet | service
id: <vendor-local id>     # e.g., VMID, interface name, webhook slug
name: <human label>
instance_id: <walnut instance id>
attrs:                    # freeform, vendor-specific hints
  node: pve-01
  port: '1/0/24'
  priority: low
labels:                   # user-defined selection keys
  tier: low
  shutdown: graceful
'''

## Discovery
	- If the integration implements inventory, it can upsert targets with attrs/labels.
	- Manual target creation is not included in v1.