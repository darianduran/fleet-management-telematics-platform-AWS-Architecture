# Event Flows

This document describes the end-to-end data flows through the platform, from vehicle telemetry ingestion to user-facing dashboards.

---

## Flow 1: Telemetry Ingestion

The primary data path from vehicle to storage.

<p align="center">
  <img src="diagrams/Telemetry_Ingestion_Flow.png" alt="Telemetry Ingestion Flow" width="800"/>
</p>

### Message Types Processed

| Type | Proto | Frequency | Storage |
|------|-------|-----------|---------|
| Vehicle Data | `protos.Payload` | Every 1-5 sec | Redis, DynamoDB |
| Alerts | `protos.VehicleAlerts` | On event | DynamoDB |
| Errors | `protos.VehicleErrors` | On event | DynamoDB |
| Connectivity | `protos.VehicleConnectivity` | On change | DynamoDB |

---

## Flow 2: Real-Time Dashboard Updates

How live data reaches the browser.

<p align="center">
  <img src="diagrams/Realtime_Dashboard_Updates.png" alt="Real-time Dashboard Flow" width="800"/>
</p>

### Latency Budget

| Stage | Target | Actual |
|-------|--------|--------|
| Vehicle → Kinesis | < 100ms | ~50ms |
| Kinesis → Consumer | < 200ms | ~100ms |
| Consumer → Redis | < 10ms | ~5ms |
| Redis → SSE Server | < 10ms | ~2ms |
| SSE → Browser | < 100ms | ~50ms |
| **Total** | **< 500ms** | **~200ms** |

---

## Flow 3: Trip Lifecycle

From trip start to archived replay.

<p align="center">
  <img src="diagrams/Trip_Lifecycle_Flow.png" alt="Trip Recording and Lifecycle Flow" width="800"/>
</p>

---

## Flow 4: Vehicle Command

Sending commands to vehicles via Tesla Fleet API.

<p align="center">
  <img src="diagrams/Vehicle_Command_Flow.png" alt="Vehicle Command Flow" width="800"/>
</p>


---

## Flow 5: Emergency Alert & Dashcam Archive

Triggered when the Go Consumer detects a collision or critical safety event.

<p align="center">
  <img src="diagrams/Alert_and_Dashcam_Archive_Flow.png" alt="Alert & Dashcam Archive Flow" width="800"/>
</p>


### Alert Types That Trigger Dashcam Archive

| Alert Type | Severity | Auto-Archive | Retention |
|------------|----------|--------------|-----------|
| Collision Detected | Critical | ✓ | 1 year |
| Airbag Deployment (Pyrotechnical Devices Deployed) | Critical | ✓ | 1 year |
| Pedestrian Involved Collision | Critical | ✓ | 1 year |
| Sentry Mode Alert (Panicked State) | Critical | ✓ | 1 year |
| Forward or Side Collision Warning System Intervention | Critical | ✓ | 1 year |
| Aggressive Turning (Left/Right Acceleration >0.4g) | High | Optional | 90 days |
| Hard Braking (Backward Acceleration >0.3g) | High | Optional | 90 days |
| Excessive Speeding (VehicleSpeed > 85mph) | High | - | 30 days |
| Unauthorized Movement (Driving Time / Geofence) | High | ✓ | 90 days |

---

## Related Documents

- [Architecture Overview](01-architecture-overview.md)
- [Data Model](03-data-model.md)
- [ADR: Kinesis over SQS](../adrs/001-kinesis-over-sqs.md)
- [ADR: Redis for Real-time](../adrs/007-redis-for-realtime.md)
