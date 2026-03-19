# Industrial IoT Platform Engineering
### Reference Architecture for Production-Grade Connected Systems

[![Deploy MkDocs](https://github.com/gauravs19/iiot-reference-architecture/actions/workflows/deploy.yml/badge.svg)](https://github.com/gauravs19/iiot-reference-architecture/actions/workflows/deploy.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**Live site:** [gauravs19.github.io/iiot-reference-architecture](https://gauravs19.github.io/iiot-reference-architecture/)

---

A production-grade reference architecture guide for engineers and architects building industrial IoT systems. Covers the full stack from physical sensors to cloud platform, with real-world patterns, working code, and operational runbooks derived from production deployments.

## What's Inside

| # | Section | What it covers |
|---|---|---|
| 1 | IoT Architecture | 5-layer model, failure domains, architectural tensions |
| 2 | Hardware Layer | PLCs, RTUs, sensors, signal types, top hardware choices |
| 3 | Edge Layer | Gateways, store-and-forward, OSS vs managed platform choices |
| 4 | Communication Protocols | MQTT deep dive, OPC-UA, Modbus, wireless protocols |
| 5 | Contract Design | Schema versioning, backward compatibility, Protobuf evolution |
| 6 | D2C Data Exchange | Telemetry flow, payload design, quality codes, waveform data |
| 7 | C2D Command Exchange | Command lifecycle, idempotency, offline queue management |
| 8 | Device Provisioning | PKI hierarchy, zero-touch provisioning, device registry |
| 9 | Data Ingestion | Broker → Kafka → TSDB pipeline, throughput sizing |
| 10 | Data Modeling | Time-series schema, continuous aggregates, alarm model |
| 11 | Integration Patterns | UNS/ISA-95, OT/IT bridge, API design |
| 12 | OTA Firmware | A/B partitions, signing, rollout campaigns, delta OTA, failure recovery |
| 13 | Security | IEC 62443 zones, mTLS, topic ACLs, certificate lifecycle |
| 14 | Observability | Four signals, fleet health scoring, alerting runbooks |
| 15 | Reference Architectures | Greenfield, brownfield, remote assets, PdM, BMS, fleet, water, SaaS |
| 16 | Operational Runbooks | Device offline diagnosis, data gap investigation, cert expiry response |
| 17 | Digital Twin | ISA-95 hierarchy, twin state model, tool comparison |
| 18 | Edge ML & Inference | Use cases, deployment pipeline, runtime choices, model versioning |
| 19 | Fleet Management | Config drift, remote diagnostics, device lifecycle, bulk ops |
| 20 | Multi-Tenant Architecture | Isolation levels, broker ACLs, TimescaleDB RLS, SaaS patterns |
| 21 | API Design | REST resource model, cursor pagination, WebSocket vs SSE, rate limits |
| 22 | Disaster Recovery | RTO/RPO per component, multi-region failover, backup strategy |
| 23 | Regulatory Compliance | IEC 62443, FDA 21 CFR Part 11, NERC CIP, GDPR |
| 24 | Cost Modeling & FinOps | Per-device cost breakdown, AWS IoT vs self-hosted TCO, storage tiers |

## Key Design Principles

- **Platform-agnostic** — patterns apply whether you use AWS IoT, Azure IoT Hub, EMQX, or a fully custom stack
- **Production-proven** — every pattern reflects real deployment experience, not theoretical best practice
- **Industrial-grade** — addresses OT/IT integration, brownfield constraints, safety requirements, and regulatory compliance
- **Opinionated** — includes specific recommendations, not just option lists

## Running Locally

```bash
pip install mkdocs-material mkdocs-minify-plugin
mkdocs serve
```

Then open [localhost:8000](http://localhost:8000).

## Contributing

Issues and PRs welcome. If you have production experience that contradicts or extends a pattern here, open an issue — real-world corrections are the most valuable contributions.

## License

MIT — use freely, attribution appreciated.
