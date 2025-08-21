# Dry-Run Plan Contract

If a capability declares `dry_run: required` or `optional`, its actions must support returning a **plan** when called with `dry_run: true`.

## Plan Structure

- `steps` (array, required): A list of the operations the action will perform. This field was previously named `will_call`.
- `expected_effect` (array, optional): A list of structured diffs describing the anticipated changes to targets.
- `assumptions` (array, optional): A list of preconditions checked by the integration.
- `risk` (enum, optional): `low | medium | high`.
- `revert_hint` (string, optional): Human-readable guidance on how to undo the action.
- `notes` (string, optional): Free text for additional context.

## Plan Steps

The `steps` array can contain objects describing operations across different transports.

### `http.request`
```yaml
- http.request:
    method: POST
    url: "https://api.vendor.com/v1/devices/123/shutdown"
    headers: { "Authorization": "Bearer <masked>" }
    body: { "force": true }
```

### `ssh.commands`
```yaml
- ssh.commands:
    target: "switch-01.example.com"
    commands:
      - "configure terminal"
      - "interface 1/0/24"
      - "shutdown"
      - "end"
```

### `mqtt.publish`
```yaml
- mqtt.publish:
    broker: "mqtt.home.arpa:1883"
    topic: "zigbee2mqtt/desk/plug/set"
    payload: '{"state":"OFF"}'
    retain: false
```

### `websocket.sendrecv`
```yaml
- websocket.sendrecv:
    url: "wss://api.vendor.com/v1/streaming"
    send: { "command": "ping" }
    expect: { "contains": "pong" }
```

## Example: Mixed-Transport Plan

This plan involves running commands on an SSH target and then sending an MQTT message.
```yaml
steps:
  - ssh.commands:
      target: "core-switch-01"
      commands:
        - "poe-port 24 disable"
  - mqtt.publish:
      broker: "mqtt.local:1883"
      topic: "notifications/power"
      payload: '{"message":"PoE port 24 on core-switch-01 disabled due to power policy."}'
expected_effect:
  - target: { type: "poe-port", id: "24", name: "AP-Lobby" }
    from: { state: "enabled" }
    to: { state: "disabled" }
risk: low
revert_hint: "Run 'poe-port 24 enable' on core-switch-01."
```