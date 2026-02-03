# Real-time Fleet Management & Telematics Platform - AWS Architecture Documentation
[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws)](https://aws.amazon.com)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-purple?logo=terraform)](https://terraform.io)
[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-blue)](docs/architecture-overview.md)

A production-grade, event-driven AWS architecture for real-time vehicle telemetry processing and fleet management.

---

## Overview
This repository is a **design-first, infrastructure-as-code (IaC) case study** showcasing cloud architecture, system design tradeoffs, Terraform practices, and operational readiness for a large-scale telemetry platform. 
It documents how I would design, secure, deploy, and operate a large-scale fleet management and telematics platform capable of servicing tens of thousands of real-time vehicle events per second.

**The scope of this repository includes:**
- System design & architecture tradeoffs.
- Event-driven, scalable AWS patterns.
- Terraform (IaC) structure & practices.
- Security, reliability, & operational readiness.
  
**Out of the scope**
- Frontend and application code.
- Business logic implementation.
- Live app or data.

**Note:** This is an architectural case study, not a full application codebase. The application showcase and source code will be released in separate repositories in the future.

### The Startup Lens
I'm approaching this architecture from a startup perspective, prioritizing cost efficiency while minimizing performance or reliability degradation. While enterprise-scale solutions might choose differently, every decision here balances capability against budget constraints.

**Target Scale:**
| Metric | Target |
|--------|--------|
| Peak Telemetry Events | 10,000+ events/sec |
| API P99 Latency | < 100ms |
| Connected Vehicles | 1,000 - 10,000 |
| Availability | 99.9% |
| Monthly Cost | $300 - $1,500 (scale dependent) |

## Architecture
This high-level architecture uses the C4 Model (Level 1 - Context) to show system boundaries and external actors. 
