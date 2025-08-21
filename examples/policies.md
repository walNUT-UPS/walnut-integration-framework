# Policy Example: Mixed-Transport Power Shedding

> Policies are defined in the UI. This example shows how the engine resolves them using capabilities and targets that may span multiple transports.

**Intent:** During a power outage, shed non-essential load by turning off low-priority devices, including PoE-powered access points on switches and MQTT-controlled smart plugs.

### Policy Steps

1.  **Select all low-priority powered devices.**
    -   Target selector: `(type=poe-port AND label=priority=low) OR (type=outlet AND label=priority=low)`

2.  **Turn off all selected devices.**
    -   Capability call: `power.control` verb `off`

3.  **Wait for confirmation.**
    -   Timeout: 15 seconds

### Resolution and Dry-Run

The policy engine resolves the targets and invokes the appropriate integration for each.

-   A `poe-port` target on a network switch is managed by an **SSH-based integration**.
-   An `outlet` target is managed by an **MQTT-based smart plug integration**.

If `dry_run: true` is requested, the engine collects plans from all involved integrations. The combined plan might look like this:

```yaml
# Combined plan from two integrations (SSH switch + MQTT plug)
steps:
  - ssh.commands:
      target: "access-switch-01"
      commands:
        - "configure terminal"
        - "interface 1/0/12"
        - "poe disable"
  - mqtt.publish:
      broker: "mqtt.home.arpa:1883"
      topic: "zigbee2mqtt/office-fan/set"
      payload: '{"state":"OFF"}'
expected_effect:
  - target: { type: poe-port, id: "1/0/12", name: "AP-Office-Ceiling" }
    from: { power: "on" }
    to: { power: "off" }
  - target: { type: outlet, id: "office-fan", name: "Office Fan" }
    from: { state: "ON" }
    to: { state: "OFF" }
risk: low
```