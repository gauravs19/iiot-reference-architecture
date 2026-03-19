# Regulatory Compliance

Industrial IoT systems in regulated industries must satisfy auditors as well as engineers. Compliance is not a feature added at the end — it shapes fundamental architecture decisions: audit trail immutability, access control granularity, data retention policies, and change management processes. The frameworks below are the most commonly encountered in industrial IoT. Understanding which framework applies to your deployment determines which architecture patterns are mandatory versus recommended.

### 23.1 IEC 62443 — Industrial Cybersecurity

IEC 62443 is the primary cybersecurity standard for industrial automation and control systems. It defines Security Levels (SL 1–4) applied to both individual components and complete systems, and uses a zone-and-conduit model to partition the network. It is referenced by critical infrastructure regulators globally and is becoming a requirement for equipment sold into the EU under the NIS2 Directive.

| Security Level | Requirements | Architecture Mapping |
|---|---|---|
| **SL1** | Password authentication, basic network segmentation | Minimum for any IoT deployment; no mTLS required at this level |
| **SL2** | mTLS, role-based access control, complete audit trail, software integrity verification | Recommended baseline — maps to §13 security architecture and §8 provisioning |
| **SL3** | Multi-factor authentication, physical security controls, formal risk assessment (IEC 62443-3-2) | Critical infrastructure (water, energy, transport) |
| **SL4** | Highest assurance — government, military, nuclear | Full formal verification; outside scope of this document |

**Mapping IEC 62443 requirements to this document:**
- Zone and conduit model → §13.2 network segmentation architecture
- Device identity and authentication → §8 device provisioning and mTLS
- Software update integrity → §12.2 firmware signing (Ed25519 signature verification)
- Audit trail → §9 ingestion pipeline (immutable Kafka log as audit record)
- Access control → §7 command authorization and §21.3 API rate limiting and quota design

### 23.2 FDA 21 CFR Part 11 — Pharmaceutical

FDA 21 CFR Part 11 governs electronic records and electronic signatures in pharmaceutical manufacturing. Any IoT system that generates or stores data used in batch records, quality decisions, or regulatory submissions falls under Part 11. Non-compliance is a Form 483 observation and can result in import alerts or consent decrees.

**Key requirements and IoT architecture implications:**

| Part 11 Requirement | IoT Architecture Implementation |
|---|---|
| Electronic records must be trustworthy and reliable | Immutable append-only storage: write-once S3 for raw data; TimescaleDB with audit trigger table that cannot be modified |
| Complete audit trail of data creation and modification | Every telemetry write, config change, and command logged with: user identity (or device ID for automated writes), timestamp (UTC, NTP-synchronized), previous value |
| Electronic signatures for critical operations | Commands to control systems require operator e-signature via TOTP code or PKI-based signature; signature recorded in command log |
| System validation (IQ/OQ/PQ) | Architecture must be fully documented and version-controlled; all changes go through formal change control; validation package maintained |
| Access controls reviewed periodically | Role-based access with quarterly access review; departed employees de-provisioned within 24 hours |

**Audit table pattern (TimescaleDB):**
```sql
CREATE TABLE config_audit_log (
    id          BIGSERIAL PRIMARY KEY,
    device_id   TEXT NOT NULL,
    changed_by  TEXT NOT NULL,   -- user_id or 'system'
    change_type TEXT NOT NULL,   -- 'config_update', 'command_issued', 'fw_update'
    old_value   JSONB,
    new_value   JSONB NOT NULL,
    reason      TEXT,
    ts          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- This table has no UPDATE or DELETE grants — insert only.
```

### 23.3 NERC CIP — Power Utilities

NERC CIP (Critical Infrastructure Protection) standards apply to entities operating the Bulk Electric System (BES) in North America. IoT systems connected to or monitoring BES assets must comply with the applicable CIP standards.

| CIP Standard | Subject | IoT Architecture Impact |
|---|---|---|
| **CIP-002** | BES Cyber System identification | Classify each IoT component: does it affect BES reliability? Classifies what CIP standards apply |
| **CIP-005** | Electronic Security Perimeter (ESP) | Maps directly to zone architecture (§13.2) — IoT gateway sits in the ESP boundary; cloud connection is through a "routable protocol" Electronic Access Point |
| **CIP-007** | Systems Security Management | Patch management (§12 OTA), port and service control (disable unused services on gateway), security event monitoring (§14 observability) |
| **CIP-010** | Configuration Change Management | Device configuration management (§9), change request process before any configuration modification, baseline documentation |
| **CIP-013** | Supply Chain Risk Management | Firmware signing (§12.2) — verify firmware integrity before deployment; vendor security assessment for hardware and software components |

### 23.4 GDPR / Data Residency

For IoT systems in the EU or processing data related to EU individuals, GDPR applies in ways that are not immediately obvious. Industrial IoT data is often assumed to be non-personal — but this is not always correct.

**Personal data in IoT context:**
- Location tracking data (GPS fleet tracking) if vehicles are assigned to individuals
- Building occupancy data from badge readers or BLE beacons linked to employee identifiers
- Operator identity in audit trails (command issued by `john.smith@plant-detroit.acme.com`)
- Wearable safety device data (worker location in factory, physiological monitoring)

**Architecture implications:**

**Data minimisation:** Collect only what is needed for the stated purpose. Do not forward operator names in every telemetry message — use an operator ID that maps to a name only in a separate identity system. Strip personal identifiers from the telemetry stream before it leaves the factory if they are not required for cloud-side processing.

**Right to erasure:** Device data tied to a specific person (wearable safety device, GPS tracker assigned to a named driver) must be deletable on request. TimescaleDB does not efficiently support row-level deletion across large compressed hypertables. Design for this: store person-linked data in a separate table (not co-mingled with equipment telemetry), use a `subject_id` column, and run `DELETE WHERE subject_id = ?` when erasure is requested. Verify deletion has propagated to all replicas and backups (or document that backups are excluded from erasure scope and will expire within the retention period).

**Cross-border transfer:** Telemetry from EU factories cannot be transferred to US cloud services without either Standard Contractual Clauses (SCCs), an adequacy decision for the destination country, or explicit consent (impractical for machine telemetry). Practical options: deploy cloud infrastructure in EU regions (AWS eu-west-1, Azure westeurope), use on-premises TimescaleDB with cloud used only for analytics, or establish SCCs with your cloud provider (all major cloud providers have SCCs in place — verify your data processing agreement).

---
