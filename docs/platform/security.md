# Security Architecture

### 13.1 Threat Model

```mermaid
graph LR
    subgraph Threats
        T1[Unauthenticated<br/>broker connections]
        T2[MITM on device<br/>communication]
        T3[Malicious firmware<br/>pushed via OTA]
        T4[Stolen device<br/>certificate]
        T5[Replay attack<br/>on commands]
        T6[Lateral movement<br/>from compromised gateway]
        T7[Data exfiltration<br/>via device]
    end
    subgraph Controls
        C1[mTLS + cert pinning]
        C2[TLS 1.3 everywhere]
        C3[ECDSA firmware signing<br/>+ secure element]
        C4[Short-lived certs<br/>+ CRL / OCSP]
        C5[Command TTL<br/>+ nonce + timestamp]
        C6[Network segmentation<br/>IEC 62443 zones]
        C7[Topic-level ACLs<br/>per device scope]
    end
    T1 --> C1
    T2 --> C2
    T3 --> C3
    T4 --> C4
    T5 --> C5
    T6 --> C6
    T7 --> C7
```

### 13.2 Network Zone Architecture (IEC 62443)

IEC 62443 defines the zone-and-conduit model that is the basis for industrial cybersecurity architecture. The key principle is that every zone boundary is a trust boundary: traffic crossing a zone boundary must be explicitly authorized, and the direction of trust matters — higher-trust zones (cloud) must never be able to initiate connections into lower-trust zones (OT). The diagram below shows the standard four-zone model; pay attention to Zone 0 (safety systems), which must be physically isolated — no network path exists to or from it, by design. Any architecture that touches Zone 0 from a network connection is non-compliant with IEC 61511 safety requirements.

```mermaid
graph TB
    subgraph Z0["Zone 0 — Safety Layer (isolated, SIL-rated)"]
        SIS[Safety Instrumented System<br/>Burner Management, ESD]
    end
    subgraph Z1["Zone 1 — Control (OT Network, no internet)"]
        PLC[PLCs / DCS]
        HMI[Local HMI]
    end
    subgraph Z2["Zone 2 — Operations DMZ"]
        HIST[OT Historian]
        GW[Edge Gateways]
        MQTT_LOCAL[Local MQTT Broker]
    end
    subgraph Z3["Zone 3 — Business IT / Cloud"]
        CLOUD_PLAT[Cloud IoT Platform]
        ERP[ERP / Business Systems]
    end
    subgraph Z4["Zone 4 — Internet (untrusted)"]
        REMOTE[Remote Access<br/>VPN only]
    end

    Z0 -.->|"Physical isolation - no network path"| Z1
    Z1 -->|"OPC-UA / Modbus - OT-to-DMZ only"| Z2
    Z2 -.->|"Blocked: no inbound to control zone"| Z1
    Z2 -->|"MQTT TLS outbound or HTTPS"| Z3
    Z3 -.->|"No direct inbound to DMZ"| Z2
    Z4 -->|"VPN only - Jump host required"| Z3
```

### 13.3 Certificate Lifecycle Management

Certificate expiry is the most common operational security incident in IoT fleets — more common than actual attacks. A device with an expired certificate cannot authenticate to the broker, which means it silently stops sending data. At fleet scale with certificates expiring on different schedules, this creates a rolling pattern of unexplained device offline events that consumes significant operations time. The workflow below implements a 90-day advance warning with automated renewal, which means no device should ever expire as a surprise. The overlap period (old and new certs both valid) is essential: it allows rollback if the new cert has a problem, and accommodates devices that are offline during the renewal window.

```
Certificate states and operations:

  ISSUED → ACTIVE → EXPIRING (30 days) → EXPIRED → REVOKED
                                ↓
                          AUTO-RENEW
                          (if device online)

Rotation workflow:
  1. Monitor cert expiry: daily job checks all devices
     SELECT device_id, cert_expiry, (cert_expiry - NOW()) as days_left
     FROM devices
     WHERE cert_expiry < NOW() + INTERVAL '90 days'
     ORDER BY cert_expiry ASC;

  2. 90 days before expiry: issue new cert, push via secure channel
  3. Device stores new cert alongside old cert
  4. Device tests new cert (connects to test endpoint)
  5. On success: device switches to new cert, reports cert_id
  6. Old cert revoked after 7-day overlap

  Revocation for compromised cert:
    CRL: update CRL, distribute to all brokers (max 24h propagation)
    OCSP: real-time revocation check (preferred, adds latency)

    Industrial reality: many devices cannot do OCSP (no internet, constrained MCU)
    Solution: CRL distribution to edge gateway (gateway validates on behalf of device)
```

---
