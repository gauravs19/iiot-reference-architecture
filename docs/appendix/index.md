# Appendices

## Appendix A: Protocol Quick Reference

| Protocol | Port | Transport | Auth | Encryption | Industrial Use | Maturity |
|---|---|---|---|---|---|---|
| MQTT 3.1.1 | 1883/8883 | TCP | User+Pass / Cert | TLS | Primary telemetry | Production |
| MQTT 5.0 | 1883/8883 | TCP | User+Pass / Cert | TLS | Advanced features | Growing |
| CoAP | 5683/5684 | UDP | None/DTLS | DTLS | Constrained devices | Niche |
| AMQP 1.0 | 5671/5672 | TCP | SASL | TLS | Enterprise bus | Production |
| OPC-UA | 4840/4843 | TCP | Cert/User | TLS | PLC integration | Production |
| Modbus TCP | 502 | TCP | None | None (add VPN) | Legacy PLC | Legacy |
| Modbus RTU | — | RS-485 | None | None | Field devices | Legacy |
| DNP3 | 20000 | TCP/Serial | None | TLSv5 | Utilities/SCADA | Production |
| EtherNet/IP | 44818 | TCP/UDP | None | None | AB/Rockwell PLCs | Production |
| PROFINET | — | Ethernet | None | None | Siemens PLCs | Production |
| EtherCAT | — | Ethernet | None | None | Motion control | Production |
| LoRaWAN | — | RF | AES-128 | Built-in | Low-power WAN | Production |
| WirelessHART | 2.4GHz | RF | AES-128 | Built-in | Process sensor retrofit | Production |

## Appendix B: OPC-UA Quality Codes Reference

| Code (Dec) | Hex | Meaning | Action |
|---|---|---|---|
| 192 | 0xC0 | Good | Use value |
| 216 | 0xD8 | Good, local override | Use value, flag override |
| 0 | 0x00 | Bad | Alarm, do not use |
| 24 | 0x18 | Bad — device failure | Alarm, check device |
| 28 | 0x1C | Bad — sensor failure | Alarm, check sensor |
| 32 | 0x20 | Bad — last known value | Use with caution |
| 64 | 0x40 | Uncertain | Use with caution |
| 68 | 0x44 | Uncertain — sensor not accurate | Maintenance needed |
| 80 | 0x50 | Uncertain — last known value | Data may be stale |

## Appendix C: Schema Version Compatibility Matrix

| Change Type | Backward Compatible | Forward Compatible | Action Required |
|---|---|---|---|
| Add optional field | ✅ Yes | ✅ Yes | None |
| Add required field | ❌ No | ✅ Yes | Deploy consumer first |
| Remove unused field | ✅ Yes | ❌ No | Audit all consumers first |
| Remove used field | ❌ No | ❌ No | Deprecate → dual-publish → remove |
| Rename field | ❌ No | ❌ No | Add new → dual-publish → remove old |
| Change type (widening) | ✅ Usually | ✅ Usually | Test consumer behavior |
| Change type (narrowing) | ❌ No | ❌ No | New field + version bump |
| Change enum values (add) | ✅ Yes | ❌ No | Consumers must handle unknown |
| Change units/semantics | ❌ No | ❌ No | New field name + version bump |

---

## Appendix D: What's Missing — Extension Roadmap

This document covers the core integration and configuration patterns. The following areas are intentionally deferred and should be added to extend this into a complete reference architecture guide.

### D.1 Gaps to Address Next

The following gaps from the original roadmap have been addressed in this document:

| Gap | Now covered in |
|---|---|
| Digital Twin & Asset Modeling | §17 |
| Edge ML / Inference | §18 |
| Fleet Management at Scale | §19 |
| Multi-Site / Multi-Tenant Architecture | §20 |
| API Design & Developer Experience | §21 |
| Disaster Recovery & Business Continuity | §22 |
| Regulatory Compliance | §23 |
| Cost Modeling & FinOps | §24 |

### D.1 Remaining Gaps

The following areas remain as future extensions:

**Edge-to-Edge Communication**
Peer-to-peer communication between gateways on the same OT network, without routing through cloud. Relevant for: coordinated load shedding across multiple PLCs, site-local command routing with zero cloud dependency.

**Digital Thread / Traceability**
End-to-end traceability from raw material to finished product using IoT data. Required for: automotive (IATF 16949), aerospace (AS9100), pharma (serialisation). Links batch records, machine telemetry, and quality inspection data into a single traceable chain.

**5G Private Network Integration**
Private 5G networks are being deployed in large factories and ports as an alternative to Wi-Fi. Architecture implications: URLLC for deterministic latency, network slicing for OT/IT separation, SIM-based device identity.

### D.2 Reference Architecture Variants to Add

| Variant | Status |
|---|---|
| Smart Building / BMS | ✅ §15.5 |
| Predictive Maintenance Platform | ✅ §15.4 |
| Fleet / Mobile Asset Tracking | ✅ §15.6 |
| Water Utilities / SCADA | ✅ §15.7 |
| Multi-Site SaaS Platform | ✅ §15.8 |
| Pharmaceutical Manufacturing | Deferred — see §23.2 for compliance patterns |

### D.3 Open Questions for Architecture Decisions

These are decisions that need to be resolved per deployment — there is no single right answer, and teams should document their choice and rationale:

1. **Broker topology:** Single global cluster vs. per-region vs. per-site? Implications for latency, data sovereignty, and failure isolation.
2. **Edge compute boundary:** Where does "cloud" start? Can the edge gateway reach back into cloud services during normal operation, or only during scheduled sync windows?
3. **Command authorization model:** Device-level ACLs (device can only receive its own commands) vs. zone-level (line controller commands any device on the line)?
4. **Data residency:** Can telemetry leave the country? EU plants with GDPR or operational data sovereignty requirements may mandate on-premises or regional cloud deployments.
5. **OT network exposure:** Should the MQTT broker be in the DMZ (accessible from OT and IT) or purely in the IT zone with only the edge gateway in the DMZ?

---

*This document is a living reference. Update the protocol table, version compatibility matrix, and reference architectures as your platform stack evolves. The patterns here are platform-agnostic by design — the underlying principles apply regardless of whether you are running on AWS IoT, Azure IoT Hub, HiveMQ, EMQX, or a fully custom stack.*
