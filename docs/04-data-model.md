# Data Model

This document describes the DynamoDB schema, access patterns, and data lifecycle for the fleet telemetry platform.  
The full table structures and relationships are visualized in the diagram above.

---

## Design Philosophy

We use **purpose-specific tables** rather than a single-table design to achieve:

- Clear separation of concerns  
- Independent scaling per table  
- Easier team onboarding  
- Granular backup and restore  
- Reduced blast radius for schema evolution  

See [ADR-002](../adrs/002-dynamodb-design.md) for the full rationale.

---

## Table Overview

| Category | Tables | Purpose |
|----------|--------|---------|
| Vehicle State | 4 | Real-time vehicle data |
| Trips | 2 | Trip tracking and history |
| Security | 4 | Alerts, geofences, audit |
| Fleet Management | 8 | Organizations, users, drivers |
| Auth | 2 | Tokens, VIN mapping |
| Telemetry | 4 | Alerts, errors, metrics |

**Total: 28 tables**

**Note:** The diagram shows a concise subset of fields for readability. Some tables (e.g., `vehicle-state`) contain additional telemetry attributes in production.

---

## Core Vehicle Tables

<p align="center">
  <img src="/diagrams/data-model-diagram.png" alt="Visual diagram of core DynamoDB Tables" width="100%"/>
</p>

### vehicle-state

**Purpose:**  
Stores the latest real-time snapshot of each vehicle’s state.

**Notable Details:**
- One item per vehicle (single-row snapshot model)
- Overwritten on every telemetry update (no history)
- Optimized for ultra-low-latency reads by dashboard and APIs
- Frequently updated; hot partition risk mitigated by pVIN hashing
- Schema is intentionally extensible and evolves as OEMs expose new telemetry fields

**Primary Access Patterns:**
- Get current state by VIN  
- Replace full state on each update  

---

### trip-state

**Purpose:**  
Tracks the currently active trip for each vehicle.

**Notable Details:**
- One active trip per vehicle at a time
- Used for real-time trip progress, breadcrumb aggregation, and orphan detection
- Updated frequently during driving sessions
- Periodically scanned to detect orphaned trips (e.g., crash, disconnect)

**Primary Access Patterns:**
- Get active trip by VIN  
- Update breadcrumb count + lastUpdate  
- Scan for orphaned trips  

---

### trips-index

**Purpose:**  
Stores metadata for completed trips and supports historical queries.

**Notable Details:**
- Append-only records for completed trips  
- Optimized for both:
  - Per-vehicle trip history
  - Cross-fleet trip views by date  
- Stores only metadata and S3 object keys (not raw telemetry blobs)
- AI summaries are optional and backfilled asynchronously

**Primary Access Patterns:**
- List trips by vehicle  
- List trips by date (GSI)  
- Get specific trip by ID  

---

## Fleet Management Tables

### organizations

**Purpose:**  
Stores fleet-level metadata and configuration.

**Notable Details:**
- One row per organization (tenant)
- Settings are schemaless and evolve over time
- Used as the root entity for RBAC and tenant isolation
- Rarely updated compared to telemetry tables

**Primary Access Patterns:**
- Get organization by ID  
- Update organization settings  

---

### organization-users

**Purpose:**  
Maps users to organizations with roles and permissions.

**Notable Details:**
- Supports many-to-many user ↔ organization membership  
- Role-based access control (owner/admin/manager/driver)  
- Driver accounts may be scoped to assigned vehicles  
- GSI enables reverse lookup (find orgs for a user)

**Primary Access Patterns:**
- List users in an organization  
- Find organizations for a user (GSI)  
- Get membership record  

---

## Security Tables

### vin-mapping

**Purpose:**  
Maps pseudoVINs to real VINs. This is the only place real VINs exist.

**Notable Details:**
- **Admin-only access** with CloudTrail auditing  
- Real VINs stored encrypted at rest  
- Supports user → vehicle lookup via GSI  
- Used at ingestion and command execution boundaries only  
- Application databases never store real VINs

**Primary Access Patterns:**
- Resolve pseudoVIN → real VIN (admin workflows)  
- List vehicles owned by a user  

---

### command-audit

**Purpose:**  
Immutable audit trail of all vehicle commands issued by users.

**Notable Details:**
- Append-only  
- TTL-based retention (e.g., 90 days)  
- Queryable by:
  - User (for security review)
  - Vehicle (for incident investigation)  
- Required for compliance, forensics, and abuse detection

**Primary Access Patterns:**
- List commands by user  
- List commands by vehicle (GSI)  

---

## Data Lifecycle

### Hot Data (DynamoDB)

| Table | Retention | Notes |
|------|-----------|------|
| vehicle-state | Forever | Overwritten on each update |
| trip-state | Until trip ends | Deleted after archival |
| trips-index | 90 days | TTL-based cleanup |
| live-breadcrumbs | 24 hours | TTL-based cleanup |

---

### Warm Data (S3 Standard)

| Prefix | Retention | Transition |
|--------|-----------|------------|
| trips/ | 90 days | → Intelligent Tiering |
| dashcam/webview/ | 90 days | → Intelligent Tiering |

---

### Cold Data (S3 Glacier IR)

| Prefix | Retention | Notes |
|--------|-----------|------|
| trips/ (>90d) | 1 year | Glacier Instant Retrieval |
| dashcam/archive/ | 180 days | Then deleted |

---

## Access Pattern Summary

| Pattern | Table | Key Condition |
|--------|-------|--------------|
| Get vehicle state | vehicle-state | PK=vin |
| Get active trip | trip-state | PK=vin |
| List vehicle trips | trips-index | PK=vin |
| List trips by date | trips-index (GSI) | PK=tripDate |
| Get user's orgs | organization-users (GSI) | PK=userId |
| List org members | organization-users | PK=organizationId |
| Audit commands by user | command-audit | PK=userId |
| Audit commands by vehicle | command-audit (GSI) | PK=vin |

---

## Related Documents

- [ADR: DynamoDB Design](../adrs/002-dynamodb-design.md)  
- [Security Design](05-security.md)  
- [Event Flows](02-event-flows.md)
