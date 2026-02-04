# Architecture Overview

## Design Principles

### 1. Serverless-First
Default to managed services (Lambda, DynamoDB, API Gateway) unless there's a compelling reason for containers. This minimizes operational overhead and optimizes costs for variable workloads.

### 2. Event-Driven Architecture
Components communicate through events (Kinesis, EventBridge) rather than synchronous calls. This provides loose coupling, better scalability, and natural replay capabilities.

### 3. Security by Design
Zero-trust model with VIN pseudonymization, least-privilege IAM, and encryption everywhere. Sensitive data (real VINs) is isolated with restricted access.

### 4. Cost Optimization
Pay-per-request billing, intelligent tiering for storage, and right-sized compute. No idle resources.

---

## Infrastructure & Network Architecture

![AWS Infrastructure & Network Diagram](diagrams/aws_infra_network_diagram.jpg)

The diagram above illustrates the complete AWS infrastructure topology:

**Edge Layer:**
- CloudFront distribution serves as the single entry point, routing traffic to appropriate origins based on path patterns
- WAF provides rate limiting and managed rule protection at the edge
- Route 53 handles DNS resolution with health checks

**Network Layer:**
- VPC spans two Availability Zones for high availability
- Public subnets host NAT Instances (cost-optimized alternative to NAT Gateway) and the Network Load Balancer
- Private subnets contain all compute workloads (ECS Fargate tasks) and ElastiCache Redis
- S3 and DynamoDB Gateway Endpoints eliminate NAT costs for AWS service traffic
- Kinesis Interface Endpoint bypasses NAT for better efficiency and NAT cost reduction

**Compute Layer:**
- Fleet Telemetry Server (ECS) terminates vehicle WebSocket connections via NLB
- Go Consumer (ECS Fargate) processes Kinesis streams with KCL
- SSE Server (ECS Fargate Spot) maintains browser connections for real-time updates
- Lambda functions handle API requests and async processing

**Data Layer:**
- Kinesis Data Streams provides ordered, replayable event ingestion
- DynamoDB stores vehicle state, trip data, and user metadata
- ElastiCache Redis enables sub-millisecond pub/sub for live updates
- S3 archives trip data and dashcam media with lifecycle policies

**Security Boundaries:**
- All ECS tasks and Lambda functions run in private subnets with no direct internet access
- mTLS terminated between Vehicles and Telemetry Server
- Security groups enforce least-privilege network access between components
- Signing Proxy Lambda has restricted egress (Tesla API only)

---

## Related Documents

- [Event Flows](02-event-flows.md) — Detailed data flow diagrams
- [Data Model](03-data-model.md) — DynamoDB schema and access patterns
- [Security Design](05-security.md) — Authentication, authorization, encryption
- [ADRs](../adrs/) — Architectural decision records
