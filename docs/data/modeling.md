# Data Modeling for IoT

### 10.1 Time-Series Schema — Wide vs. Narrow

The wide vs. narrow schema decision is one of the highest-leverage choices in your IoT data model, and the wrong choice is expensive to reverse at scale. Narrow tables are schema-flexible but query-expensive: getting temperature and pressure for a pump at the same instant requires a self-join or pivot. Wide tables are faster for cross-tag analysis (the common dashboard query) but require a schema migration for every new sensor type. The recommendation for most production deployments is a hybrid: a wide table for the stable core tags of each device type, and a narrow overflow table for extended diagnostic or optional tags.

```sql
-- NARROW table: one row per tag per timestamp
-- Pros: simple schema, new tags require no schema change
-- Cons: 15x more rows, JOINs for cross-tag analysis, poor columnar compression

CREATE TABLE telemetry_narrow (
    time       TIMESTAMPTZ     NOT NULL,
    device_id  TEXT            NOT NULL,
    tag        TEXT            NOT NULL,
    value      DOUBLE PRECISION,
    quality    SMALLINT        DEFAULT 192
);
SELECT create_hypertable('telemetry_narrow', 'time',
    chunk_time_interval => INTERVAL '1 day');
CREATE INDEX ON telemetry_narrow (device_id, tag, time DESC);

-- WIDE table: one row per device per timestamp (all tags as columns)
-- Pros: columnar compression, fast cross-tag queries, natural schema
-- Cons: schema change for new tags, NULL-heavy for sparse devices

CREATE TABLE telemetry_pump (
    time            TIMESTAMPTZ     NOT NULL,
    device_id       TEXT            NOT NULL,
    temp_inlet_c    DOUBLE PRECISION,
    temp_outlet_c   DOUBLE PRECISION,
    pressure_bar    DOUBLE PRECISION,
    flow_m3h        DOUBLE PRECISION,
    vibration_rms   DOUBLE PRECISION,
    power_kw        DOUBLE PRECISION,
    q_temp_inlet    SMALLINT DEFAULT 192,
    q_pressure      SMALLINT DEFAULT 192
    -- quality codes per tag that matter
);
SELECT create_hypertable('telemetry_pump', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- RECOMMENDATION:
-- Wide table per device type — better query performance
-- Narrow table for dynamic/unknown tags — more flexible
-- Hybrid: wide for core tags, narrow for extended/diagnostic tags
```

### 10.2 Continuous Aggregates — Production Query Performance

Raw data queries at 1s resolution over 90 days are slow. Use continuous aggregates. A dashboard querying 90 days of 1Hz data for a single device touches 7.8 million rows — and most dashboards query multiple devices and tags simultaneously. Continuous aggregates pre-compute these roll-ups incrementally as new data arrives, reducing a 30-second query to sub-second response time. The critical configuration detail is the `start_offset`: set it to cover your maximum expected late-arriving data (devices reconnecting after outages can deliver data that is hours old). Setting it too small means roll-ups are computed before all data has arrived, producing incorrect aggregates.

```sql
-- 1-minute aggregate (materialized, auto-updated)
CREATE MATERIALIZED VIEW telemetry_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    device_id,
    AVG(temp_inlet_c)   AS temp_inlet_avg,
    MIN(temp_inlet_c)   AS temp_inlet_min,
    MAX(temp_inlet_c)   AS temp_inlet_max,
    STDDEV(temp_inlet_c) AS temp_inlet_stddev,
    AVG(pressure_bar)   AS pressure_avg,
    AVG(flow_m3h)       AS flow_avg,
    AVG(power_kw)       AS power_avg,
    COUNT(*)            AS sample_count,
    -- Quality: only aggregate Good readings (q=192)
    COUNT(*) FILTER (WHERE q_temp_inlet = 192) AS good_quality_count
FROM telemetry_pump
GROUP BY bucket, device_id
WITH NO DATA;

-- Auto-refresh policy
SELECT add_continuous_aggregate_policy('telemetry_1min',
    start_offset => INTERVAL '2 hours',   -- reprocess last 2h (late data)
    end_offset   => INTERVAL '1 minute',  -- don't process very recent
    schedule_interval => INTERVAL '1 minute');

-- 1-hour aggregate (built on top of 1-minute, not raw)
CREATE MATERIALIZED VIEW telemetry_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS hour,
    device_id,
    AVG(temp_inlet_avg)   AS temp_inlet_avg,
    MIN(temp_inlet_min)   AS temp_inlet_min,
    MAX(temp_inlet_max)   AS temp_inlet_max,
    SUM(sample_count)     AS sample_count
FROM telemetry_1min
GROUP BY hour, device_id
WITH NO DATA;

-- Retention: raw data 7 days, 1-min 90 days, 1-hour forever
SELECT add_retention_policy('telemetry_pump',  INTERVAL '7 days');
SELECT add_retention_policy('telemetry_1min',  INTERVAL '90 days');
-- telemetry_1hour: no retention policy (keep forever)
```

### 10.3 Alarm & Event Model

Alarm management is a safety-critical function in industrial environments, not just a notification feature. The schema below implements a proper alarm lifecycle (active → acknowledged → cleared → shelved) that matches ISA-18.2, the industrial alarm management standard. Without acknowledgement tracking, operators cannot tell whether an alarm has been seen. Without shelving, nuisance alarms that cannot be immediately fixed will be ignored — and real alarms will be buried. The flood detection query at the bottom is particularly important: a single failing sensor can generate thousands of alarms per hour, masking real incidents and overwhelming operators.

```sql
CREATE TABLE alarms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       TEXT NOT NULL,
    tag             TEXT NOT NULL,
    alarm_type      TEXT NOT NULL,          -- 'process', 'device', 'comms'
    severity        TEXT NOT NULL,          -- 'critical','high','medium','low'
    condition       TEXT NOT NULL,          -- 'gt','lt','eq','rate_of_change'
    threshold       DOUBLE PRECISION,
    value_at_trigger DOUBLE PRECISION,
    activated_at    TIMESTAMPTZ NOT NULL,
    cleared_at      TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by TEXT,
    shelved_until   TIMESTAMPTZ,            -- operator can shelve noisy alarms
    state           TEXT NOT NULL           -- 'active','cleared','acked','shelved'
        CHECK (state IN ('active','cleared','acked','shelved')),
    message         TEXT,
    norm_value      DOUBLE PRECISION        -- value when cleared
);

CREATE INDEX ON alarms (device_id, activated_at DESC);
CREATE INDEX ON alarms (state, severity) WHERE state = 'active';

-- Alarm flood detection query
SELECT device_id, COUNT(*) as alarm_count
FROM alarms
WHERE activated_at > NOW() - INTERVAL '10 minutes'
GROUP BY device_id
HAVING COUNT(*) > 10
ORDER BY alarm_count DESC;
-- Devices with > 10 alarms in 10 min = flooding → suppress + alert ops
```

---
