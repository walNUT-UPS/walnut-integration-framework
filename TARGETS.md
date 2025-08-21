# Target Model

A **Target** is the resource that an action operates on. Each target is owned by a single integration instance.

## Target Shape

- `type` (string, required): The category of the target.
- `id` (string, required): The integration-local identifier (e.g., a VM ID, interface name, or webhook slug).
- `name` (string, required): A human-readable label for the target.
- `instance_id` (string, required): The ID of the integration instance that owns this target.
- `attrs` (object, optional): A set of key-value pairs with vendor-specific hints and read-only state (e.g., `port: '1/0/24'`, `power_state: 'ON'`).
- `labels` (object, optional): User-defined key-value pairs for selection and grouping (e.g., `tier: critical`, `priority: low`).

## Target Types

The framework defines a set of base target types:
- `vm`, `host`, `poe-port`, `webhook`, `outlet`, `service`

Integrations are encouraged to define their own, more specific target types when appropriate. These types do not need to be registered in advance.

### Example Vendor-Defined Targets

A network switch integration might define:
- `type: switch`, `attrs: { model: 'Aruba CX 6300', firmware: '10.09' }`
- `type: ap`, `attrs: { ip_address: '10.1.1.5', clients: 25 }`
- `type: vlan`, `attrs: { vlan_id: 100, name: 'guest-wifi' }`

A security camera integration might define:
- `type: camera`, `attrs: { recording: true, motion_detected: false }`

## Discovery

Integrations that implement discovery can automatically create and update targets, populating their attributes and labels from the source system.