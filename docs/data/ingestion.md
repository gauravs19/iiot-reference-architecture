# Data Ingestion Pipelines

The ingestion pipeline is the path between "message arrives at the broker" and "data is queryable in the time-series database." It sounds like a simple write operation but at production scale it is a distributed system with its own failure modes, bottlenecks, and tuning surface.

The critical design decision is whether to write directly from MQTT to your database or to introduce a message bus (Kafka, Kinesis, Pulsar) in between. The message bus adds operational complexity but provides: consumer fan-out (multiple services processing the same stream independently), replay capability (re-process historical data after a bug fix), and decoupling (database can be down without losing messages). For deployments above ~5,000 devices or requiring multiple downstream consumers, the message bus is almost always worth it.

A second key decision is **schema validation at ingestion**. You can validate early (at the broker via auth hooks, or at the ingestion worker) or late (at query time). Validate early: bad data caught late costs more to remediate and may have already flowed into dashboards, alerts, and ML models.

### 9.1 Scalable Ingestion Architecture

```mermaid
graph LR
    subgraph DEVICES["10,000 Devices"]
        D1[Device Group A]
        D2[Device Group B]
        D3[Device Group C]
    end

    subgraph BROKER["MQTT Broker Cluster<br/>(EMQX / HiveMQ)"]
        B1[Node 1]
        B2[Node 2]
        B3[Node 3]
    end

    subgraph KAFKA["Message Bus<br/>(Kafka / Kinesis)"]
        K1[Topic: raw-telemetry<br/>32 partitions]
        K2[Topic: commands-ack]
        K3[Topic: device-events]
    end

    subgraph CONSUMERS["Stream Consumers"]
        C1[Ingestion Workers<br/>x8 instances<br/>validate+normalize]
        C2[Rules Engine<br/>Flink / Spark Streaming]
        C3[Alert Processor]
        C4[ML Feature Pipeline]
    end

    subgraph STORAGE["Storage Layer"]
        S1[TimescaleDB<br/>Hot: 90 days]
        S2[Parquet / S3<br/>Cold: unlimited]
        S3[Redis<br/>Last-known-value cache]
        S4[ElasticSearch<br/>Events + Alarms]
    end

    DEVICES -->|MQTT TLS| BROKER
    BROKER -->|Bridge / Rule| KAFKA
    KAFKA --> CONSUMERS
    C1 --> S1
    C1 --> S2
    C1 --> S3
    C2 --> C3
    C3 --> S4
    C4 --> S2
```

### 9.2 Throughput Math — Sizing Your Pipeline

Pipeline sizing is one of the most common causes of production incidents in IoT platforms — not because engineers do not care, but because they use the wrong input numbers. The calculation below walks through a realistic mid-size plant deployment with all the factors that are typically forgotten: deadband filtering (which typically reduces traffic 60–80% from the raw poll rate), payload format choice (Protobuf vs JSON matters at this scale), and the distinction between broker capacity and database capacity. Run this calculation before committing to infrastructure, and add at least 3× headroom for burst traffic during connectivity restoration events.

```
Real-world calculation for a mid-size plant deployment:

  Devices:          2,000
  Tags per device:  15 (mix of fast and slow)
  Fast tags (temp, pressure, flow): 10 tags, 1s interval
  Slow tags (totals, setpoints):     5 tags, 60s interval

  Raw message rate:
    Fast: 2,000 × 10 × 1 msg/s  = 20,000 msg/s
    Slow: 2,000 × 5 / 60 msg/s  =    167 msg/s
    Total: ≈ 20,167 msg/s

  After deadband filtering (assume 70% reduction in steady state):
    Effective: ≈ 6,050 msg/s to broker

  Payload size:
    JSON (verbose): 350 bytes avg → 2.1 MB/s ingress
    JSON (compact):  200 bytes avg → 1.2 MB/s ingress
    Protobuf:         80 bytes avg → 0.48 MB/s ingress

  Daily volume (compact JSON):
    1.2 MB/s × 86,400s = 103 GB/day uncompressed
    With LZ4 compression: ≈ 25-35 GB/day on disk

  Kafka partition sizing:
    Target: 50,000 msg/s per partition (conservative)
    Partitions needed: ceil(6,050 / 50,000) = 2 → use 8 (for consumer parallelism)

  TimescaleDB sizing:
    6,050 rows/s × 86,400s = 522M rows/day
    At 50 bytes/row (compressed): ≈ 26 GB/day
    90-day hot storage: ≈ 2.3 TB

  Chunk interval for hypertable:
    time_bucket = '1 day' for 90-day retention
    chunk_time_interval = INTERVAL '1 day'
```

---
