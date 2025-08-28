# walNUT Integration Plugin Author GPT Operating Manual

This document is the authoritative guide for creating walNUT integration plugins. It encodes the current enforced truth from orchestrator audit 1a3a090 (2025-08-26) to ensure generated integrations conform without guesswork.

## 1. Overview

**Integration Types vs Instances**: An "integration type" is a plugin package (.int file) defining capabilities, schemas, and driver code. An "instance" is a configured, deployed copy of a type with specific connection details and secrets.

**Lifecycle**: Types are uploaded/validated → instances are created from valid types → instances are tested/monitored → capabilities are invoked on healthy instances.

## 2. Manifest Schema — Current Enforcement

### 2.1 Required Top-Level Fields
Every `plugin.yaml` must contain:
- `id` - Unique identifier (e.g., "com.aruba.aoss", "walnut.proxmox.ve")
- `name` - Human-readable display name
- `version` - Semantic version (e.g., "0.1.0")
- `min_core_version` - Minimum orchestrator version (e.g., "0.10.0")
- `category` - UI grouping hint ("network-device", "host-orchestrator", "generic", etc.)
- `driver` - Driver specification (see 2.2)
- `schema.connection` - JSON Schema for connection configuration
- `capabilities` - Array of capability definitions (see 2.4)

### 2.2 Driver Specification
```yaml
driver:
  entrypoint: "driver:ClassName"  # Format: ^[A-Za-z_][A-Za-z0-9_]*:[A-Za-z_][A-Za-z0-9_]*
  # Implies driver.py file with ClassName class
```

**Requirements**: 
- Must have `driver.py` file in integration root
- Class must implement required methods per Option A mapping (see Section 3)

### 2.3 Connection Schema & Secrets
```yaml
schema:
  connection:
    type: object
    required: [hostname, username, password]
    properties:
      hostname: { type: string }
      username: { type: string }
      password: { type: string, secret: true }    # secret: true = stored in IntegrationSecrets
      api_token: { type: string, secret: true }   # Multiple secrets allowed
```

**Secret Handling**: Any property with `secret: true` is:
- Stored separately in `integration_secrets` table
- Excluded from config validation during updates
- Available to driver via secrets map at runtime

### 2.4 Capabilities
```yaml
capabilities:
  - id: "poe.port"                    # Dots allowed; maps to method poe_port()
    verbs: ["set"]                    # Array of string verbs
    targets: ["poe-port"]             # Array of target types (kebab-case in manifest)
    dry_run: "required"               # required|optional|not_supported
    # Optional fields:
    invertible: 
      set: { inverse: "set" }
    idempotency:
      key_fields: ["verb", "target_id", "params.state"]
```

**Capability Rules**:
- `id` becomes method name via `id.replace('.', '_')`  
- Must have corresponding method in driver class
- `dry_run` enum strictly enforced: `required|optional|not_supported`
- `verbs` and `targets` are free-form strings
- **Target naming**: Use kebab-case in manifest (`stack-member`, `poe-port`) - API normalizes to snake_case for driver

### 2.5 Optional Sections

**Defaults**:
```yaml
defaults:
  transports:
    ssh: { timeout_s: 30, port: 22 }
    snmp: { timeout_s: 5 }
  heartbeat_interval_s: 60
  dry_run_refresh_sla_s: 8          # 1-60 seconds for dry-run freshness
```

**Test Configuration**:
```yaml
test:
  method: "driver"                   # driver|http|tcp
  # If method=http:
  http:
    request: { method: "GET", path: "/version" }
    success_when: "status == 200"
```

**Requirements Section (Optional)**: Available in orchestrator ≥ 0.10.1 PRE-RELEASE (see Section 2.7).

### 2.6 Requirements (Optional)

**Per-Plugin Dependencies** - Available in orchestrator ≥ 0.10.1 PRE-RELEASE:
```yaml
requires:
  python: ">=3.12,<3.13"           # Informational Python version range
  deps:                            # List of dependencies  
    - "requests>=2.28.0"           # Simple string spec
    - name: "paramiko"             # Object spec with extras/markers
      version: ">=2.11.0"
      extras: ["ed25519"]
      markers: "sys_platform != 'win32'"
  wheelhouse: "wheels/"            # Optional offline wheel directory in .int bundle
```

**Runtime Behavior**:
- **With requires**: Orchestrator creates per-plugin virtualenv and imports driver within that environment
- **Without requires**: Legacy import rules apply (host Python environment)
- **Wheelhouse**: If present, orchestrator prefers offline install from bundled wheels
- **Online install**: Allowed when no wheelhouse provided

**Validation Error Keys**:
- `deps_missing` - Required dependencies not found
- `deps_install_error` - Dependency installation failed  
- `deps_unsupported_platform` - Platform constraints not met
- `deps_wheelhouse_not_found` - Declared wheelhouse directory missing from bundle

### 2.7 Minimal Complete Example

**Basic example (legacy compatibility)**:
```yaml
id: walnut.proxmox.ve
name: Proxmox VE
version: 1.0.1
min_core_version: 0.1.0
category: host-orchestrator
driver:
  entrypoint: driver:ProxmoxVeDriver
schema:
  connection:
    type: object
    required: [host, port, node, api_token]
    properties:
      host: { type: string }
      port: { type: integer, default: 8006 }
      node: { type: string }
      api_token: { type: string, secret: true }
defaults:
  heartbeat_interval_s: 120
  dry_run_refresh_sla_s: 5
test:
  method: http
  http:
    request: { method: GET, path: /version }
    success_when: "status == 200"
capabilities:
  - id: vm.lifecycle
    verbs: [shutdown, start, stop, suspend, resume, reset]
    targets: [vm]
    dry_run: required
  - id: inventory.list
    verbs: [list]
    targets: [vm, host]
    dry_run: optional
```

**Example with dependencies (orchestrator ≥ 0.10.1)**:
```yaml
id: example.network.device
name: Network Device Integration
version: 1.0.0
min_core_version: 0.10.1
category: network-device
driver:
  entrypoint: driver:NetworkDeviceDriver
requires:
  python: ">=3.8"
  deps:
    - "requests>=2.28.0"
    - "netmiko>=4.1.0"
    - name: "cryptography"
      version: ">=40.0.0"
      markers: "platform_system != 'Windows'"
  wheelhouse: "wheels/"     # Optional: bundle wheels in integration
schema:
  connection:
    type: object
    required: [hostname, username, password]
    properties:
      hostname: { type: string }
      username: { type: string }
      password: { type: string, secret: true }
test:
  method: driver
capabilities:
  - id: device.config
    verbs: [read, backup]
    targets: [device]
    dry_run: optional
```

## 3. Driver Contract (Option A - Current Enforcement)

### 3.1 Capability → Method Mapping
**Rule**: For each manifest capability `id`, driver must expose method named `id.replace('.', '_')`.

Examples:
- `vm.lifecycle` → `vm_lifecycle()`
- `poe.port` → `poe_port()`
- `inventory.list` → `inventory_list()`

**Validation**: Method presence checked at install time; signatures not currently validated (see Known Gaps).

### 3.2 Required Base Methods

**test_connection** (mandatory):
```python
def test_connection(self) -> dict:
    """Test connectivity and return status.
    
    Returns:
        dict: {
            "status": "connected"|"degraded"|"error",
            "latency_ms": int,
            "details": str  # optional error details
        }
    """
```

Maps to instance states: `connected|degraded|error`.

### 3.3 Inventory Contract

**inventory_list** (if inventory.list capability declared):
```python
def inventory_list(self, target_type: str, active_only: bool = True, options: dict = None) -> list:
    """List inventory items of specified type.
    
    Args:
        target_type: "vm"|"stack_member"|"port" (normalized from API)
        active_only: Filter to active items only
        options: Additional parameters (reserved)
        
    Returns:
        list: Items matching target type specification
    """
```

**Target Normalization**: 
- **In manifests**: Use kebab-case target names (`stack-member`, `poe-port`)
- **API processing**: Normalizes to snake_case before driver calls (`stack-member`→`stack_member`, `poe-port`→`port`)
- **Driver methods**: Receive normalized snake_case target types

### 3.4 Action Method Signatures

**vm.lifecycle example**:
```python
def vm_lifecycle(self, verb: str, target: dict, dry_run: bool = False, **params) -> dict:
    """Execute VM lifecycle action.
    
    Args:
        verb: "start"|"shutdown"|"stop"|"suspend"|"resume"|"reset"
        target: { "type": "vm", "external_id": str, "name": str, ... }
        dry_run: Return plan instead of executing
        **params: Action-specific parameters
        
    Returns:
        dict: Result with status/plan/error details
    """
```

**Other capability patterns**: Each capability's method signature varies but generally follows `(verb, target, dry_run, **params)` pattern.

## 4. Inventory Targets & "Active" Definition

### 4.1 Current Target Types & Naming

**Target Naming Convention**:
- **Manifest**: Use kebab-case (`stack-member`, `poe-port`)
- **Driver receives**: snake_case after API normalization (`stack_member`, `port`)

**Common target types**:
- **vm**: Virtual machines
- **stack-member**: Switch stack members (driver receives `stack_member`)  
- **port**: Network ports/interfaces
- **poe-port**: PoE-enabled ports (driver receives `port`)
- **host**: Physical hosts/servers
- **switch**: Network switches
- **interface**: Generic network interfaces

### 4.2 Target Object Shapes

**VM**:
```python
{
    "type": "vm",
    "external_id": "100",
    "name": "web-server-01", 
    "attrs": {
        "status": "running",
        "node": "pve-node-1",
        "memory_mb": 2048,
        "cpu_count": 2
    },
    "labels": {}
}
```

**Stack Member**:
```python
{
    "type": "stack_member",
    "external_id": "1",
    "name": "Switch-1",
    "attrs": {
        "model": "Aruba 2930F-24G-4SFP+",
        "status": "active",
        "role": "commander"
    },
    "labels": {}
}
```

**Port**:
```python
{
    "type": "port", 
    "external_id": "1/0/1",
    "name": "GigabitEthernet1/0/1",
    "attrs": {
        "link": "up",              # Required for active calculation
        "media": "1000T",
        "speed_mbps": 1000,
        "poe_class": "3",
        "poe_power_w": 15.2,       # Used in active calculation
        "poe_status": "delivering"  # Used in active calculation
    },
    "labels": {}
}
```

### 4.3 Active Port Definition
A port is considered "active" if:
```python
link == "up" OR poe_power_w > 0 OR poe_status == "delivering"
```

API applies this filter when `active_only=True` requested.

## 5. Packaging & Upload (.int Files)

### 5.1 Package Requirements
- **Extension**: Must be `.int` file
- **Format**: ZIP archive
- **Size**: ≤ 10MB
- **Contents**: Must contain `plugin.yaml` at root
- **Security**: No path traversal (no `..` or absolute paths)

### 5.2 Required Files
- `plugin.yaml` - Manifest (mandatory)
- `driver.py` - Driver implementation (if `driver.entrypoint` specified)
- `README.md` - Documentation (recommended)

### 5.3 Upload Process & WebSocket Events
Streaming upload emits WebSocket events:
- `integration_job.event` - Progress updates with `{job_id, ts, phase, level, message, meta}`
- `integration_job.done` - Final result with `{success, type_id, installed_path, result/error}`

**Phases**: upload → unpack → manifest → install → registry → final

### 5.4 Installation
- Extracted to `./integrations/{id}/`
- Previous folder removed if exists
- Type record created with status=checking
- Validation triggered automatically

## 6. Validation Pipeline

### 6.1 Upload Validation (Phase-by-Phase)
1. **Extension check**: Reject non-.int files → 400 "File must have .int extension"
2. **Size check**: Reject >10MB → 413 "File too large (max 10MB)" 
3. **ZIP validity**: Must pass `zipfile.is_zipfile()` → 400 "Invalid ZIP file"
4. **Zip-slip guard**: No `..` or absolute paths → 400 "Unsafe file path in archive"
5. **Manifest presence**: Must contain `plugin.yaml` → 400 "No plugin.yaml found in package"
6. **Manifest parsing**: Must be valid YAML dict → 400 "Invalid plugin.yaml content"
7. **ID presence**: Must have `id` field → 400 "Missing 'id' field in plugin.yaml"
8. **Duplicate check**: Conflict on existing ID → 409 with removal guidance

### 6.2 Type Validation (Load/Run Phase)
Validation errors and their keys:

| Error Key | Cause | Status |
|-----------|--------|---------|
| `schema_error` | Manifest fails JSON Schema validation | invalid |
| `driver_missing` | No `driver.py` file found | invalid |
| `import_error` | Cannot import driver class | invalid |  
| `capability_mismatch` | Capability method missing from driver | invalid |
| `missing_test_connection` | Driver lacks `test_connection()` method | invalid |
| `core_version_incompatible` | `min_core_version` > orchestrator version | invalid |
| `deps_missing` | Required dependencies not found | invalid |
| `deps_install_error` | Dependency installation failed | invalid |
| `deps_unsupported_platform` | Platform constraints not met | invalid |
| `deps_wheelhouse_not_found` | Declared wheelhouse directory missing | invalid |
| `validation_error` | General validation failure | invalid |

**Success**: Sets status=valid, persists capabilities/schema/defaults.

### 6.3 Type Statuses
- `checking` - Discovery/validation in progress
- `valid` - Validation passed, ready for instances
- `invalid` - Validation failed (see error keys above)
- `superseded` - Replaced by newer version
- `unavailable` - Integration folder missing

### 6.4 Instance Statuses  
- `configured` - Created but not tested
- `connected` - test_connection() succeeded
- `degraded` - test_connection() succeeded with warnings
- `error` - test_connection() failed
- `unknown` - State unclear
- `needs_review` - Config changed, requires attention
- `type_unavailable` - Type was removed (blocks operations)

## 7. Instances & Health

### 7.1 Instance Creation
- Requires type with status ∈ {valid, checking}
- Instance `name` must be unique
- Config validated against non-secret subset of connection schema  
- Secrets stored in separate `integration_secrets` table
- Initial state: `configured`, `last_test` set to creation timestamp

### 7.2 Health Testing
**test_connection() flow**:
1. Load driver dynamically
2. Build secrets map from stored secrets
3. Instantiate driver with config + secrets
4. Call `driver.test_connection()`
5. Update instance: `last_test`, `latency_ms`, `state` per result

**State mapping**: 
- `status: "connected"` → `state = connected`
- `status: "degraded"` → `state = degraded`  
- `status: "error"` → `state = error`

### 7.3 Configuration Updates
- Optional name change and `config` updates
- Config validated against schema (secrets excluded)
- On config change: `state` → `needs_review`

## 8. Inventory API & Caching

### 8.1 Endpoints
- `GET /integrations/instances/{id}/inventory?type=vm&active_only=true&page=1&page_size=50`
- `GET /integrations/instances/{id}/inventory/summary` - Returns counts by type

### 8.2 Caching Policy
- **Default TTL**: 180 seconds
- **Fresh data**: Returned if age < TTL
- **Stale fallback**: If `cached_only=true` or refresh in progress
- **Active fallback**: Can serve inactive cache for active_only requests (filtered)

### 8.3 Cache Refresh
Background refresh triggered when serving stale data. Driver called with:
```python
driver.inventory_list(target_type="vm", active_only=True, options={})
```

## 9. Error/Status Taxonomy

### 9.1 Upload Errors
| HTTP | Error | Message |
|------|-------|---------|
| 400 | Extension | "File must have .int extension" |
| 413 | Size | "File too large (max 10MB)" |
| 400 | ZIP format | "Invalid ZIP file" |
| 400 | Path safety | "Unsafe file path in archive" |
| 400 | Missing manifest | "No plugin.yaml found in package" |
| 400 | Manifest format | "Invalid plugin.yaml content" |
| 400 | Missing ID | "Missing 'id' field in plugin.yaml" |
| 409 | Duplicate | Guidance to remove existing type |

### 9.2 Validation Error Messages
Provide clear remediation guidance in integration README for each potential failure mode.

**Recommended error handling in manifest**:
- Include comprehensive README with troubleshooting
- Test connection schema thoroughly  
- Verify all capability methods exist
- Document required Python dependencies

### 9.3 Common Schema Validation Issues

**Target naming errors**: Use kebab-case in manifest targets:
```yaml
# ✓ Correct
capabilities:
  - id: inventory.list
    targets: [stack-member, poe-port, vm]

# ✗ Incorrect (snake_case in manifest)
capabilities:
  - id: inventory.list
    targets: [stack_member, poe_port, vm]
```

The API normalizes kebab-case to snake_case before calling driver methods.

## 10. Examples

### 10.1 Proxmox VE Integration

**Manifest** (trimmed):
```yaml
id: walnut.proxmox.ve
name: Proxmox VE
driver:
  entrypoint: driver:ProxmoxVeDriver
capabilities:
  - id: vm.lifecycle
    verbs: [shutdown, start, stop, suspend, resume, reset]
    targets: [vm]
    dry_run: required
  - id: power.control  
    verbs: [shutdown, cycle]
    targets: [host]
    dry_run: required
  - id: inventory.list
    verbs: [list]
    targets: [vm, host]
    dry_run: optional
```

**Driver methods required**:
```python
class ProxmoxVeDriver:
    def test_connection(self) -> dict: ...
    def vm_lifecycle(self, verb, target, dry_run=False, **params) -> dict: ...
    def power_control(self, verb, target, dry_run=False, **params) -> dict: ...
    def inventory_list(self, target_type, active_only=True, options=None) -> list: ...
```

### 10.2 ArubaOS-S Integration

**Manifest** (trimmed):
```yaml
id: com.aruba.aoss
name: ArubaOS-S Switches
capabilities:
  - id: poe.port
    verbs: [set]
    targets: [poe-port]
    dry_run: required
  - id: poe.priority
    verbs: [set] 
    targets: [poe-port]
    dry_run: required
  - id: inventory.list
    verbs: [list]
    targets: [stack-member, port]
    dry_run: optional
```

**Driver methods required**:
```python
class ArubaOSSwitchDriver:
    def test_connection(self) -> dict: ...
    def poe_port(self, verb, target, dry_run=False, **params) -> dict: ...
    def poe_priority(self, verb, target, dry_run=False, **params) -> dict: ...
    def inventory_list(self, target_type, active_only=True, options=None) -> list: ...
```

**Inventory handling**: Returns stack_member items for switches, port items for interfaces. Port names follow pattern from switch aliases/descriptions to external_id mapping.

## 11. Known Gaps vs. Planned Target

### 11.1 Method Signature Validation
**Gap**: Validation only checks method presence, not signatures. Runtime assumes specific signatures without validation.

**Impact**: Integration may install successfully but fail at runtime due to signature mismatch.

**Recommendation**: Add `inspect.signature()` validation during type validation.

### 11.2 Type Removal → Instance State Propagation
**Gap**: When integration type removed, instances not automatically marked `type_unavailable`.

**Current**: Type marked `unavailable`, folder deleted, but instance states unchanged.

**Impact**: API blocks operations but instance state doesn't reflect unavailability.

### 11.3 Capability Verb/Target Validation
**Gap**: No enforcement that declared verbs/targets are handled by driver methods.

**Recommendation**: Extend validation to verify method parameter handling per declared capability scope.

## 12. Directory & File Structure

### 12.1 Driver-Based Integration
```
com.aruba.aoss/
├── plugin.yaml
├── driver.py
├── README.md
├── requirements.txt  # Python dependencies (optional)
└── tests/           # Unit tests (recommended)
    └── test_driver.py
```

### 12.2 Transport-Only Integration  
```
example.http.pinger/
├── plugin.yaml     # test.method: http
└── README.md
```

**Key Guidelines**:
- Directory name must match `id` field
- `plugin.yaml` always required at root
- `driver.py` required if `driver.entrypoint` specified
- README.md strongly recommended for troubleshooting guidance

## 13. Authoring Checklist

Before submitting integration:
- [ ] ID unique and follows naming convention
- [ ] All required manifest fields present
- [ ] Connection schema minimal (only required fields)
- [ ] Secret fields flagged with `secret: true`
- [ ] Test method appropriate for integration type
- [ ] Capabilities minimal and verbs consistent
- [ ] Driver methods exist for all capability IDs
- [ ] `test_connection()` method implemented
- [ ] Destructive operations require confirmation parameters
- [ ] Dry-run support implemented per declaration
- [ ] README includes troubleshooting guidance
- [ ] YAML validates against schema

## 14. GPT Collaboration Protocol

1. **Scoping**: Ask 5-8 focused questions about target system, capabilities needed, authentication, and operational constraints.

2. **Capability Design**: Propose minimal capability set in table format. Justify exclusions to avoid feature creep.

3. **Manifest Generation**: Produce valid `plugin.yaml` following current enforcement rules. Include recommended directory structure.

4. **Iteration**: One focused change per round. Keep manifest churn minimal.

5. **Validation**: Ensure generated integration validates against current schema and includes proper error handling.

6. **Documentation**: Include clear README with connection requirements, capability descriptions, and troubleshooting guidance.

**Safety Requirements**: 
- Include `params.confirm: boolean` for destructive operations
- Specify explicit revert procedures for risky capabilities  
- Implement comprehensive dry-run support with clear execution plans