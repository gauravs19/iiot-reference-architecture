<div align="center">

# Industrial IoT Platform Engineering

### The production-grade reference for engineers building connected industrial systems

[![Live Site](https://img.shields.io/badge/Live%20Site-gauravs19.github.io-blue?style=for-the-badge&logo=github)](https://gauravs19.github.io/iiot-reference-architecture/)
[![Deploy](https://img.shields.io/github/actions/workflow/status/gauravs19/iiot-reference-architecture/deploy.yml?style=for-the-badge&label=Deploy)](https://github.com/gauravs19/iiot-reference-architecture/actions/workflows/deploy.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)
[![Stars](https://img.shields.io/github/stars/gauravs19/iiot-reference-architecture?style=for-the-badge)](https://github.com/gauravs19/iiot-reference-architecture/stargazers)

**24 sections · Full stack coverage · Production-derived patterns · Platform-agnostic**

[**Read the Guide**](https://gauravs19.github.io/iiot-reference-architecture/) · [Browse Sections](#whats-inside) · [Run Locally](#running-locally) · [Contribute](#contributing)

</div>

---

## Why This Exists

Most IIoT guides are either vendor documentation or theoretical whitepapers. This is neither.

Every pattern here was derived from **real production deployments** — factory floors, remote oil & gas sites, water utilities, and cold-chain logistics. The failures are real. The tradeoffs are real. The recommendations are opinionated because wishy-washy option lists don't help when you're on call at 2am debugging why 300 devices went dark.

**If you're designing, building, or operating an industrial IoT platform, this is the guide you'll wish existed when you started.**

---

## The Full Stack at a Glance

```mermaid
graph TB
    subgraph L5["L5 · Application"]
        A1[Dashboards] --- A2[ML / Analytics]
        A2 --- A3[ERP / MES]
        A3 --- A4[Alerting]
    end

    subgraph L4["L4 · Platform / Cloud"]
        B1[MQTT Broker] --> B2[Kafka / Stream Processor]
        B2 --> B3[TimescaleDB / InfluxDB]
        B4[Device Registry + OTA Service]
    end

    subgraph L3["L3 · Edge / Gateway"]
        C1[Protocol Translator] --> C2[Store & Forward Buffer]
        C3[Edge Rules + ML] --> C4[OTA Agent]
    end

    subgraph L2["L2 · Field Network"]
        D1[Modbus / RS-485]
        D2[PROFINET / EtherCAT]
        D3[WirelessHART / LoRa]
    end

    subgraph L1["L1 · Device & Sensor"]
        E1[PLCs / DCS]
        E2[RTUs]
        E3[Smart Sensors / Drives]
    end

    L5 <-->|"REST APIs / WebSocket / SSE"| L4
    L4 <-->|"MQTT over mTLS"| L3
    L3 <-->|"OT Network / DMZ"| L2
    L2 <-->|"Fieldbus / Industrial Ethernet"| L1
```

> **The most common architectural mistake:** collapsing the edge and cloud layers into one "IoT platform" — until the factory loses internet for 4 hours and all sensor data disappears.

---

## What's Inside

### Hardware & Protocols
| Section | What It Covers |
|---|---|
| **Hardware Layer** | PLCs, RTUs, smart sensors, analog vs. digital signals, top hardware choices by industry |
| **Edge Layer & Gateways** | Store-and-forward design, OSS (Balena, WasmEdge) vs. managed (Greengrass, IoT Edge) |
| **Communication Protocols** | MQTT deep dive, OPC-UA security model, Modbus RTU/TCP, WirelessHART, LoRaWAN, BACnet |

### Data & Contracts
| Section | What It Covers |
|---|---|
| **Contract Design** | Schema versioning, Protobuf evolution rules, backward compatibility guarantees |
| **D2C Data Exchange** | Telemetry payload design, OPC-UA quality codes, waveform/burst data, compression |
| **C2D Commands** | Command lifecycle, idempotency tokens, offline queue management, ack patterns |
| **Device Provisioning** | PKI hierarchy, X.509 zero-touch provisioning, device registry design |
| **Data Ingestion** | Broker → Kafka → TSDB pipeline, throughput sizing, backpressure handling |
| **Data Modeling** | TimescaleDB schema, continuous aggregates, alarm state machine, asset hierarchy |

### Platform Engineering
| Section | What It Covers |
|---|---|
| **Integration Patterns** | Unified Namespace (UNS), ISA-95 hierarchy, OT/IT DMZ, historian bridge patterns |
| **OTA Firmware** | A/B partition scheme, signing pipeline, staged rollout campaigns, delta OTA, brick recovery |
| **Security Architecture** | IEC 62443 zones, mTLS everywhere, topic ACLs, certificate lifecycle automation |
| **Observability** | Four golden signals for IoT, fleet health scoring, anomaly detection, alerting runbooks |

### Operations & Advanced
| Section | What It Covers |
|---|---|
| **Reference Architectures** | Greenfield, brownfield, remote assets, PdM, BMS, water utility, SaaS multi-tenant |
| **Operational Runbooks** | Device offline diagnosis, data gap investigation, cert expiry, broker overload response |
| **Digital Twin** | ISA-95 asset hierarchy, twin state model, AWS IoT TwinMaker vs. Azure DT vs. open source |
| **Edge ML & Inference** | Anomaly detection at edge, model packaging, OTA model updates, runtime comparison |
| **Fleet Management** | Config drift detection, remote diagnostics, bulk operations, device lifecycle states |
| **Multi-Tenant Architecture** | Isolation levels, per-tenant broker ACLs, TimescaleDB RLS, SaaS onboarding patterns |
| **API Design** | REST resource model, cursor pagination, WebSocket vs. SSE for live telemetry, rate limits |
| **Disaster Recovery** | RTO/RPO targets per component, multi-region failover, edge-buffered recovery |
| **Regulatory Compliance** | IEC 62443, FDA 21 CFR Part 11, NERC CIP, GDPR for device data |
| **Cost Modeling & FinOps** | Per-device cost breakdown, AWS IoT vs. self-hosted TCO, storage tier strategy |

---

## Reference Architecture — Full System View

```mermaid
graph TB
    subgraph FIELD["Physical & Field Layer  ·  §2 Hardware  ·  §4 Protocols"]
        PLC["PLCs / DCS / RTUs"]
        SEN["Smart Sensors & Drives"]
        FN["Field Network<br/>Modbus · OPC-UA · PROFINET · WirelessHART · LoRa"]
        PLC --- FN
        SEN --- FN
    end

    subgraph EDGE["Edge Layer  ·  §3 Gateways  ·  §8 Provisioning  ·  §12 OTA  ·  §18 Edge ML"]
        PROTO["Protocol Translator"]
        SAF["Store & Forward Buffer"]
        ERULES["Edge Rules Engine"]
        EDGEML["Edge ML Inference"]
        OTAA["OTA Agent"]
        CERT["X.509 Identity"]
        PROTO --> SAF
        ERULES --> SAF
        EDGEML --> SAF
    end

    subgraph PLATFORM["Cloud Platform  ·  §9 Ingestion  ·  §10 Modeling  ·  §13 Security  ·  §14 Observability"]
        BROKER["MQTT Broker<br/>mTLS · Topic ACLs · §13"]
        STREAM["Stream Processor<br/>Kafka / Kinesis · §9"]
        TSDB["Time-Series DB<br/>TimescaleDB · §10"]
        DREG["Device Registry<br/>Lifecycle · §8"]
        OTAS["OTA Service<br/>Campaigns · Rollback · §12"]
        SEC["Security Plane<br/>PKI · IEC 62443 · §13"]
        OBS["Observability<br/>Metrics · Logs · Traces · §14"]
        BROKER --> STREAM --> TSDB
        DREG --- BROKER
        OTAS --- OTAA
    end

    subgraph APP["Application Layer  ·  §15 Ref Archs  ·  §17 Digital Twin  ·  §19 Fleet  ·  §21 API"]
        DASH["Dashboards &<br/>Visualization"]
        TWIN["Digital Twin<br/>ISA-95 Asset Model · §17"]
        MLPLAT["ML Platform<br/>PdM · Anomaly · §18"]
        FLEET["Fleet Management<br/>Config Drift · §19"]
        APIGW["API Gateway<br/>REST · WebSocket · SSE · §21"]
        ERPINT["ERP / MES<br/>Integration · §11"]
    end

    subgraph XCUT["Cross-Cutting Concerns  ·  §20–24"]
        MT["Multi-Tenant<br/>Isolation · RLS · §20"]
        DR["Disaster Recovery<br/>RTO/RPO · §22"]
        COMP["Regulatory Compliance<br/>IEC 62443 · FDA · NERC · §23"]
        FINOPS["FinOps<br/>Per-device TCO · §24"]
    end

    FN -->|"fieldbus / industrial ethernet"| PROTO
    SAF -->|"MQTT over mTLS"| BROKER
    CERT -->|"zero-touch enroll"| DREG
    TSDB -->|"query / subscribe"| APP
    SEC -.->|"governs"| EDGE
    SEC -.->|"governs"| PLATFORM
    OBS -.->|"monitors"| EDGE
    OBS -.->|"monitors"| PLATFORM
    PLATFORM -.->|"shapes"| XCUT
```

## Guide Coverage Map

```mermaid
mindmap
  root((IIoT Platform<br/>Engineering))
    Hardware & Protocols
      §2 PLCs · RTUs · Sensors
      §3 Edge Gateways
      §4 MQTT · OPC-UA · Modbus
    Data Pipeline
      §5 Contract Design
      §6 D2C Telemetry
      §7 C2D Commands
      §8 Device Provisioning
      §9 Data Ingestion
      §10 Data Modeling
    Platform Engineering
      §11 Integration · UNS · ISA-95
      §12 OTA Firmware Updates
      §13 Security · IEC 62443
      §14 Observability
    Reference Architectures
      §15 Greenfield · Brownfield
      §15 PdM · BMS · Water · SaaS
    Operations
      §16 Runbooks
      §17 Digital Twin
      §18 Edge ML & Inference
      §19 Fleet Management
    Advanced Topics
      §20 Multi-Tenant Architecture
      §21 API Design
      §22 Disaster Recovery
      §23 Regulatory Compliance
      §24 FinOps & Cost Modeling
```

---

## Design Principles

| Principle | What It Means in Practice |
|---|---|
| **Platform-agnostic** | Patterns apply to AWS IoT, Azure IoT Hub, EMQX, or a fully self-hosted stack |
| **Production-proven** | Every pattern reflects real deployment experience — failures included |
| **Brownfield-first** | Assumes legacy PLCs, Modbus RTUs, and 1990s historians — because that's the real world |
| **Opinionated** | Specific recommendations with reasoning, not just option lists |
| **Operability over elegance** | If you can't monitor it and recover from failures, it doesn't belong in the design |

---

## Who This Is For

- **Platform engineers** designing the cloud-side ingestion, storage, and API layer
- **Edge/embedded engineers** building gateway firmware, OTA agents, and protocol drivers
- **Architects** evaluating build vs. buy, platform choices, and long-term IIoT strategy
- **OT/IT integration teams** bridging SCADA/historian systems into modern cloud platforms
- **Anyone inheriting** an existing IIoT deployment and trying to understand what they have

---

## Running Locally

```bash
pip install mkdocs-material mkdocs-minify-plugin
git clone https://github.com/gauravs19/iiot-reference-architecture.git
cd iiot-reference-architecture
mkdocs serve
```

Open [localhost:8000](http://localhost:8000). Full search, dark mode, and Mermaid diagrams work locally.

---

## Contributing

Real-world corrections are the most valuable contributions to this guide.

If you have production experience that **contradicts, extends, or adds nuance** to a pattern here — open an issue. Vendor experience, deployment failures, protocol edge cases, and compliance gotchas are all welcome.

**What makes a good contribution:**
- A pattern that failed in production and why
- A better tradeoff analysis for a specific industry vertical
- A missing section (check the [extension roadmap](docs/appendix/index.md))
- Corrections to protocol details, schema examples, or cost figures

PRs welcome. Issues are even more welcome — discussion is how the guide gets better.

---

## License

MIT — use freely in your own architecture work, internal docs, or presentations. Attribution appreciated.
