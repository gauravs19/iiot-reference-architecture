# Cloud-to-Device (C2D) Command Exchange

C2D is the reverse data flow — commands, configuration changes, and control signals flowing downward from the platform to devices. It is far less volume than D2C but far higher stakes. A telemetry message that is lost or delayed is a data gap. A command that is lost, delivered twice, or executed 10 minutes late can result in incorrect process states, product defects, or in extreme cases, safety incidents.

The gap between how web engineers think about C2D ("just POST to an endpoint") and how industrial IoT requires it to work is wide. A device may be offline for hours when a command is issued. The device may receive the same command twice due to QoS 1 retransmission. The command may have been issued by an operator based on process conditions that no longer exist by the time the device reconnects. None of these failure modes exist in a typical web service context, but all of them are daily reality in industrial IoT.

The patterns in this section — command TTL, idempotency, lifecycle state machine, offline queue management — address each of these failure modes specifically.

### 7.1 Command Lifecycle State Machine

Commands in industrial IoT are not fire-and-forget API calls. They have lifecycle, authorization, and real physical consequences.

```mermaid
stateDiagram-v2
    [*] --> Pending: Operator issues command
    Pending --> Queued: Device online - Command published to MQTT
    Pending --> Expired: Device offline - TTL elapsed
    Queued --> Delivered: Device receives - publishes RECEIVED ack
    Queued --> Expired: TTL elapsed before delivery
    Delivered --> Validating: Device checking preconditions
    Validating --> Rejected: Preconditions not met (e.g. interlock active)
    Validating --> Executing: Preconditions OK - Executing command
    Executing --> Completed: Command successful
    Executing --> Failed: Execution error
    Completed --> [*]
    Rejected --> [*]
    Failed --> [*]: Alert ops
    Expired --> [*]: Alert ops if critical
```

### 7.2 C2D Command Contract

The command contract is more than a message format — it is the authorization and safety boundary for every operator action on a physical asset. Every field below has been added to address a specific failure mode observed in production: `expires_at` prevents stale commands executing after a device reconnects from a long outage, `constraints` gives the device a last line of defense against out-of-range values even if the backend validation had a bug, and `issued_by` provides the audit trail required by most regulated industries. Do not strip fields to save bytes here — command volume is low and the overhead is justified.

```json
{
  "schema_version": "1.0",
  "command_id": "01HX7K3NBVYD5V2Q3BZ8MNRZWK",
  "command": "set_setpoint",
  "device_id": "P-007",
  "params": {
    "tag": "temperature_setpoint_c",
    "value": 75.0,
    "unit": "celsius"
  },
  "constraints": {
    "min_value": 20.0,
    "max_value": 90.0,
    "requires_interlock_clear": true,
    "max_rate_of_change_per_min": 5.0
  },
  "meta": {
    "issued_by": "jsmith@acme.com",
    "issued_at": "2026-03-19T14:00:00Z",
    "expires_at": "2026-03-19T14:02:00Z",
    "priority": "normal",
    "reason": "Batch temperature adjustment per work order WO-4472"
  }
}
```

**Command acknowledgement:**
```json
{
  "command_id": "01HX7K3NBVYD5V2Q3BZ8MNRZWK",
  "status": "completed",
  "device_id": "P-007",
  "ts": "2026-03-19T14:00:00.412Z",
  "previous_value": 70.0,
  "new_value": 75.0,
  "execution_time_ms": 412,
  "message": "Setpoint updated to 75.0°C. Ramp active.",
  "executed_by_fw": "2.3.1"
}
```

### 7.3 C2D Full Message Exchange

The full sequence diagram below shows every hop a command takes from operator action to physical execution and back. The key design points to observe: the command service records state before publishing to the broker (not after), so a broker publish failure is recoverable; the gateway validates the expiry timestamp before touching the PLC, not after; and acknowledgements flow back through the same MQTT channel so the operator UI gets real-time status without polling. This bidirectional status flow is what separates industrial command handling from simple fire-and-forget REST calls.

```mermaid
sequenceDiagram
    participant OPS as Operator / SCADA
    participant API as Command Service
    participant REG as Device Registry
    participant BRK as MQTT Broker
    participant GW as Edge Gateway
    participant PLC as PLC / Device

    OPS->>API: POST /devices/P-007/commands {command:set_setpoint, value:75.0}
    API->>REG: Check device status
    REG->>API: {online: true, fw: 2.3.1, last_seen: 2s ago}

    API->>API: Validate params, check authorization (RBAC), generate command_id + expiry
    API->>BRK: PUBLISH QoS1 acme/.../P-007/commands/{cmd_id} {full command payload}
    BRK->>API: PUBACK
    API->>OPS: 202 Accepted {command_id, status: queued}

    Note over BRK,GW: Delivery to device
    BRK->>GW: Command message
    GW->>BRK: PUBLISH QoS1 .../commands/{cmd_id}/ack {status:delivered, ts:...}

    GW->>GW: Check expiry (reject if past)
    GW->>GW: Route to appropriate handler
    GW->>PLC: Write setpoint via OPC-UA / Modbus
    PLC->>GW: Write confirmation

    GW->>BRK: PUBLISH QoS1 .../commands/{cmd_id}/ack {status:completed, new_value:75.0}

    API->>API: Update command state in DB
    OPS->>API: GET /commands/{cmd_id} → {status: completed}

    Note over API,OPS: Webhook / SSE notification to operator
    API->>OPS: PUSH notification: command completed
```

### 7.4 Idempotency — Critical for C2D

Idempotency in C2D is not a theoretical concern — QoS 1 will deliver duplicate commands in normal operation, not just edge cases. The critical distinction is between absolute commands (naturally idempotent) and relative commands (dangerous when duplicated). This distinction should be enforced at the command contract level, not handled case-by-case in device firmware. Device-side deduplication using the `command_id` as a key provides the second layer of protection when command design is correct, and the only protection when it is not.

```
Problem: QoS 1 can deliver command twice. If command is "open valve":
  First delivery:  valve opens ✓
  Second delivery: valve opens again (already open — no harm)
  Third delivery:  fine

If command is "increase setpoint by 5°C":
  First delivery:  75°C → 80°C ✓
  Second delivery: 80°C → 85°C ✗ WRONG

Rule: Absolute commands are naturally idempotent. Relative commands are not.

Solution:
  1. Prefer absolute commands: {set_to: 80.0} not {increase_by: 5.0}
  2. Track command_id on device, reject duplicates:

     # Device-side deduplication
     def handle_command(cmd):
         if db.exists(f"cmd:{cmd['command_id']}"):
             # Already processed — re-send last ack
             ack = db.get(f"cmd:{cmd['command_id']}:ack")
             publish_ack(ack)
             return

         result = execute_command(cmd)
         ack = build_ack(cmd, result)
         db.set(f"cmd:{cmd['command_id']}", "processed", ttl=86400)
         db.set(f"cmd:{cmd['command_id']}:ack", ack, ttl=86400)
         publish_ack(ack)
```

### 7.5 Command Queue Management for Offline Devices

Industrial devices go offline regularly — planned maintenance windows, network outages, power cycles. Commands issued during these windows must be managed carefully, not simply dropped or blindly queued. The flowchart below encodes the decisions that depend on command type: time-critical commands (emergency stops, interlocks) should never wait — set a short TTL and alert immediately if the device is unreachable. Configuration and maintenance commands can safely queue for hours or days. The most important rule: when a device reconnects, do not deliver a burst of queued commands simultaneously. Deliver them in order, waiting for acknowledgement before sending the next.

```mermaid
flowchart TD
    CMD[Command Issued] --> Q1{Device Online?}
    Q1 -->|Yes| PUB[Publish immediately<br/>QoS 1 retained=false]
    Q1 -->|No| Q2{Command Type?}
    Q2 -->|Time-critical<br/>e.g. emergency stop| EXP[Set short TTL<br/>60 seconds<br/>Alert ops immediately]
    Q2 -->|Operational<br/>e.g. config change| QUEUE[Queue in DB<br/>with expiry]
    Q2 -->|Maintenance<br/>e.g. OTA, log pull| QUEUE

    QUEUE --> WATCH[Connectivity Watcher]
    WATCH --> Q3{Device comes<br/>online?}
    Q3 -->|Yes, within TTL| DELIVER[Deliver queued<br/>commands in order]
    Q3 -->|TTL expired| EXPIRE[Mark expired<br/>Alert if priority high]

    DELIVER --> Q4{Multiple commands<br/>for same device?}
    Q4 -->|Yes| ORDER[Deliver in issued_at<br/>order, wait for ACK<br/>before next]
    Q4 -->|No| DELIVER2[Deliver single command]
```

**Stale command prevention — always enforce TTL on device:**
```c
// Device firmware command handler (C pseudocode)
void handle_command(Command* cmd) {
    uint64_t now_ms = get_epoch_ms();
    uint64_t expires_ms = parse_iso8601_ms(cmd->expires_at);

    if (now_ms > expires_ms) {
        LOG_WARN("Command %s expired %lums ago — rejecting",
                 cmd->command_id, now_ms - expires_ms);
        publish_ack(cmd->command_id, STATUS_EXPIRED,
                    "Command expired before execution");
        return;
    }

    // Execute...
}
```

---
