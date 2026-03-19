# Cost Modeling & FinOps

IoT deployments have a cost structure that is fundamentally different from web applications. Costs scale with device count, message frequency, and data retention — not user count or API calls. Surprises at invoice time are common because the cost model is not intuitive until you have run it at scale. The most common shock: at 10,000+ devices, the broker connection cost dominates everything else on managed IoT services, and teams that sized based on message count alone find their bill 10× higher than expected.

### 24.1 Per-Device Cost Breakdown

The following estimates use 2024/2025 list pricing. Actual costs vary by region, committed use discounts, and negotiated contracts. Use as a starting point for budget models, not as final numbers.

Assumptions: 10 messages/device/minute, 200 bytes/message, 90 days raw data retention, 2 years aggregate retention.

| Cost Component | 1,000 Devices | 10,000 Devices | 100,000 Devices |
|---|---|---|---|
| Managed broker (AWS IoT Core) | ~$0.87/device/month | ~$0.87/device/month | ~$0.87/device/month |
| Self-hosted broker (EMQX on k8s, 3× m5.xlarge) | ~$1.80/device/month | ~$0.18/device/month | ~$0.06/device/month |
| Time-series storage (TimescaleDB on managed Postgres) | ~$0.15/device/month | ~$0.12/device/month | ~$0.10/device/month |
| Data transfer egress (to dashboards/API) | ~$0.05/device/month | ~$0.04/device/month | ~$0.03/device/month |
| Ingestion compute (Kafka + workers) | ~$0.50/device/month | ~$0.08/device/month | ~$0.03/device/month |
| **Total (managed broker)** | ~**$1.57/device/month** | ~**$1.11/device/month** | ~**$1.03/device/month** |
| **Total (self-hosted broker)** | ~**$2.50/device/month** | ~**$0.42/device/month** | ~**$0.22/device/month** |

Self-hosted broker becomes cheaper than managed around 5,000–10,000 devices for the broker component. For storage, self-hosted TimescaleDB on reserved instances becomes cheaper than managed Postgres around 2,000–5,000 devices.

### 24.2 AWS IoT Core vs Self-Hosted EMQX — TCO at 10,000 Devices

Detailed calculation at 10,000 devices, 10 messages/device/minute:

**Monthly message volume:**
10,000 devices × 10 messages/min × 60 min/hr × 24 hr/day × 30 days = **432,000,000 messages/month**

**AWS IoT Core pricing (us-east-1, 2025):**
- Messaging: 432M × $0.08/1M = **$34.56/month**
- Connection: $0.0012/device-hour × 10,000 devices × 720 hours = **$8,640/month**
- Rules engine (if used): additional per-rule-execution cost
- **Total AWS IoT Core: ~$8,674/month (~$0.87/device/month)**

Connection pricing dominates at large device count. AWS IoT Core connection cost alone exceeds the total self-hosted cost at this scale.

**Self-hosted EMQX (3× m5.xlarge, us-east-1):**
- EC2 (3× m5.xlarge reserved 1yr): ~$400/month
- EBS storage: ~$60/month
- Load balancer: ~$50/month
- Operations overhead (estimate 0.1 FTE at $150k/year): ~$1,250/month
- **Total self-hosted EMQX: ~$1,760/month (~$0.18/device/month)**

At 10,000 devices, self-hosted EMQX costs approximately 5× less than AWS IoT Core. The crossover point (where managed cost = self-hosted cost including operations overhead) is approximately 1,500–2,000 devices — below this, managed services are cheaper when operational burden is accounted for.

### 24.3 Storage Cost Optimisation

| Data Tier | Volume (10,000 devices) | Storage Cost | Retention |
|---|---|---|---|
| Raw 1s telemetry (hot, TimescaleDB) | ~50 GB/day → 350 GB/week | ~$35/month (managed Postgres) | 7 days |
| Raw telemetry (cold, Parquet on S3 Standard) | ~50 GB/day → 1.5 TB/month | ~$34.50/month | 12 months |
| Raw telemetry (archive, S3 Glacier) | Historical accumulation | ~$4/TB/month | 7 years (compliance) |
| 1-min aggregates (TimescaleDB, 90 days) | ~3.4 GB/day → 306 GB/90d | ~$7.20/month | 90 days |
| 1-hr aggregates (TimescaleDB, permanent) | ~140 MB/day → 50 GB/year | ~$1.20/month/year | Permanent |

**Recommended tiering strategy:**
1. Raw data: 7 days on hot TimescaleDB storage (dashboards query this for recent views)
2. Raw data: export to Parquet on S3 Standard as chunks close (queryable with Athena)
3. Raw data: transition to S3 Glacier after 90 days (compliance archive)
4. 1-minute aggregates: retain 90 days in TimescaleDB (the primary dashboard query target)
5. 1-hour aggregates: retain permanently (negligible cost; essential for long-term trend analysis)

Applying this tiering, the blended storage cost for 10,000 devices is approximately $80–120/month rather than $3,500+/month if all raw data were kept in TimescaleDB permanently.

### 24.4 FinOps Practices for IoT

**Resource tagging:** Tag every cloud resource with: `tenant_id` (for multi-tenant), `site`, `device_type`, and `environment` (prod/staging/dev). This enables cost allocation reports by customer, by site, and by device type — essential for chargeback models and for identifying cost outliers.

**Per-service budget alerts:** Set separate budget alerts for each major service component (Kafka cluster, TimescaleDB, broker, egress) rather than a single total budget. A Kafka cost spike means a consumer is lagging and replaying messages; a storage spike means a device is sending faster than expected. Aggregate budget alerts mask root causes.

**Cost per device KPI:** Track `total_cloud_cost / active_device_count` monthly. Alert if it rises more than 20% month-over-month without a corresponding increase in device count. Unexpected cost increases are almost always caused by: a device stuck in a fast-publish loop, a consumer replaying a large Kafka partition, or a new device type with a much higher message rate than the fleet average.

**Message loop detection:** A device republishing its own messages is a common firmware bug that can generate millions of messages within hours. Set a broker-side rate limit per `client_id`: maximum 1,000 messages/minute is generous for any industrial sensor — anything above this is either a bug or a security incident. Log and alert when the rate limit is hit; do not silently drop — the device team needs to know.

---
