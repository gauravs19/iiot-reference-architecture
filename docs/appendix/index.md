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

*This document is a living reference. Update the protocol table, version compatibility matrix, and reference architectures as your platform stack evolves. The patterns here are platform-agnostic by design — the underlying principles apply regardless of whether you are running on AWS IoT, Azure IoT Hub, HiveMQ, EMQX, or a fully custom stack.*
