# API Design & Developer Experience

The IoT platform API is the interface between the data layer and the applications built on top of it — dashboards, mobile apps, ERP integrations, third-party analytics. A well-designed IoT API must handle time-series query patterns, device command issuing, streaming updates, and fleet management operations. These are fundamentally different from typical CRUD REST APIs. The most common design mistakes: offset-based pagination (breaks at 100k+ device lists), polling for live data (creates unnecessary load), and treating telemetry as a simple key-value store (ignores aggregation, quality codes, and time ranges).

### 21.1 REST API Design for IoT

The full API resource model for a production IoT platform:

```
GET  /v1/devices                          # List with pagination + filters
GET  /v1/devices/{id}                     # Device metadata + current state
POST /v1/devices/{id}/commands            # Issue command
GET  /v1/devices/{id}/commands/{cmd_id}   # Command status

GET  /v1/telemetry/{device_id}            # Time-series query
  ?tags=temp_inlet_c,pressure_bar         # Tag selection
  &from=2026-03-01T00:00:00Z             # Time range start
  &to=2026-03-19T23:59:59Z              # Time range end
  &interval=1m                           # Aggregation interval (raw if omitted)
  &agg=avg,min,max                       # Aggregation functions
  &quality=good_only                     # Filter by OPC-UA quality

GET  /v1/events/{device_id}              # Alarms and events
  ?severity=high,critical
  &state=active

GET  /v1/fleet/summary                   # Fleet-level KPIs
GET  /v1/fleet/firmware                  # Firmware version distribution
```

**Pagination:** Device lists at scale require cursor-based pagination — offset pagination (`?page=500&limit=100`) requires the database to scan and discard 50,000 rows to return page 500, which becomes catastrophically slow at 100k+ devices. Cursor-based pagination uses an opaque continuation token that encodes the last-seen position:

```json
{
  "devices": [...],
  "next_cursor": "eyJpZCI6IkdXLTAwNDUiLCJ0cyI6MTcxMDg0NDgwMH0=",
  "has_more": true,
  "total_count": 12847
}
```

The client passes `?cursor=eyJpZCI6...` to fetch the next page. The cursor is a base64-encoded JSON object containing the sort key of the last returned item — the query becomes `WHERE device_id > last_device_id LIMIT 100` which is always an index scan regardless of page depth.

### 21.2 Real-Time Push: WebSocket vs SSE

| Feature | WebSocket | Server-Sent Events (SSE) |
|---|---|---|
| Direction | Bidirectional | Server → client only |
| HTTP/2 support | Separate connection | Native HTTP/2 multiplexing |
| Auto-reconnect | Client must implement | Built into browser EventSource API |
| Proxy compatibility | Some proxies block upgrades | Standard HTTP — no issues |
| Use case fit | Command + telemetry in one connection | Read-only dashboard streaming |

**Recommendation:** Use SSE for read-only dashboard streaming (simpler, HTTP/2 compatible, browser auto-reconnect) and WebSocket only when the same connection needs to carry both live data and bidirectional commands.

SSE endpoint design:

```
GET /v1/stream/devices/{id}/telemetry
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"ts":1710844800123,"temp_inlet_c":72.4,"pressure_bar":4.2}

data: {"ts":1710844801124,"temp_inlet_c":72.5,"pressure_bar":4.2}
```

Include a `retry: 3000` directive in the SSE stream header so the browser reconnects within 3 seconds if the connection drops. Send a `:keepalive` comment every 15 seconds to prevent proxies and load balancers from closing idle connections.

### 21.3 Rate Limiting & Quotas

| Operation Type | Free Tier | Standard | Enterprise |
|---|---|---|---|
| Telemetry write (device → platform) | Exempt | Exempt | Exempt |
| Device list / metadata reads | 100 req/min | 1,000 req/min | 10,000 req/min |
| Time-series queries | 30 req/min | 300 req/min | Custom |
| Command issue | 10 req/min | 100 req/min | Custom |
| SSE stream connections | 5 concurrent | 50 concurrent | Custom |

**Critical principle:** Telemetry write operations (data flowing from device to platform) must never be rate-limited. A device that cannot write its telemetry due to an API rate limit is silently losing data — the device has no buffer for this path in the managed API model. Rate limits apply to read operations only. Sensor data is always accepted.

When a read operation is rate-limited, return HTTP 429 with a `Retry-After` header:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710844812

{"error": "rate_limit_exceeded", "message": "Query limit reached. Retry in 12 seconds."}
```

---
