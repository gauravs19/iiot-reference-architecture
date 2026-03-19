# Device Provisioning & Identity

### 8.1 PKI Hierarchy

The PKI hierarchy below is the trust foundation for the entire platform. Every device certificate, gateway certificate, and service certificate chains up to a single offline root CA. The separation into manufacturing and operations intermediate CAs is deliberate: the manufacturing CA is only online during device production, limiting the blast radius of a compromise. The operations CA issues shorter-lived certificates with automatic renewal, accepting that some operational complexity is the price of reducing the impact of a compromised certificate.

```mermaid
graph TB
    ROOT["🔑 Root CA<br/>(Offline, HSM-protected, air-gapped)<br/>Expiry: 20 years"]
    INT_MFG["Intermediate CA — Manufacturing<br/>(Online during production only)<br/>Expiry: 10 years"]
    INT_OPS["Intermediate CA — Operations<br/>(Online, per-region)<br/>Expiry: 5 years"]

    DEV["Device Certificate<br/>(Unique per device)<br/>CN = device_id<br/>Expiry: 10 years"]
    GW["Gateway Certificate<br/>(Unique per gateway)<br/>Expiry: 3 years"]
    SVC["Service Certificate<br/>(Backend services)<br/>Expiry: 1 year, auto-renew"]

    ROOT --> INT_MFG
    ROOT --> INT_OPS
    INT_MFG --> DEV
    INT_OPS --> GW
    INT_OPS --> SVC
```

### 8.2 Zero-Touch Provisioning Flow

Zero-touch provisioning means a device can go from factory floor to operational state without a technician manually entering credentials, IP addresses, or certificates on site. This matters at scale: configuring 5,000 devices manually at deployment is a months-long project with high error rates. The sequence below splits provisioning into two phases: manufacturing time (key generation, certificate installation, pre-registration) and first-power-on (automated enrollment using the manufacturing certificate as the proof of identity). The site claim token is a one-time secret that links the device to the correct customer and site during enrollment.

```mermaid
sequenceDiagram
    participant MFG as Factory / Manufacturer
    participant DEV as Device (new)
    participant PROV as Provisioning Service
    participant REG as Device Registry
    participant BROKER as Operational Broker

    Note over MFG,DEV: At manufacturing time
    MFG->>DEV: Generate key pair in secure element
    MFG->>DEV: Install device cert (signed by Mfg CA)
    MFG->>DEV: Install provisioning endpoint config
    MFG->>REG: Pre-register device {serial, hw_type, customer}

    Note over DEV,PROV: First power-on at site
    DEV->>PROV: TLS-mTLS connect with device cert, GET /provision {serial, hw_id, site_claim_token}
    PROV->>PROV: Verify cert chain → Mfg CA → Root CA
    PROV->>REG: Lookup serial → get customer, site assignment
    PROV->>PROV: Generate operational config
    PROV->>DEV: 200 OK {broker_endpoint, mqtt_topic_prefix, telemetry_interval, initial_config}
    DEV->>DEV: Store config in persistent storage
    DEV->>REG: Register {status: provisioned, fw_version, hw_revision}
    DEV->>BROKER: Connect with device cert (mTLS)
    DEV->>BROKER: PUBLISH .../status {online: true, provisioned_at: ...}
```

---
