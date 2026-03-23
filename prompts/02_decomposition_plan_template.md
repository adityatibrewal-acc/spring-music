# Monolith Decomposition Plan

## 1. Executive Summary
**Objective:**  
Explain why the monolith is being decomposed (speed, scale, reliability, cost, compliance, team autonomy).

**Non‑Goals:**  
List what is explicitly out of scope.

**Success Metrics:**  
- Deployment frequency:
- Lead time for change:
- MTTR:
- Incident rate:
- Cost / performance targets:

**Constraints:**  
- Regulatory / Compliance:
- Hosting / Infrastructure:
- Release windows:
- Security requirements:

---

## 2. Current State Overview

### 2.1 Business Capability Map
List the major business capabilities (not technical layers).

- Capability 1:
- Capability 2:
- Capability 3:
- …

### 2.2 Application Architecture
**Tech Stack:**  
- Language / Framework:
- Deployment model:
- Databases:
- Messaging / Batch:

**High-Level Diagram:**  
_(link or reference if available)_

### 2.3 Dependency & Coupling Summary
- Shared libraries:
- Cyclic dependencies:
- Shared database tables:
- Cross-module transactions:

### 2.4 Operational & Change Hotspots
- High incident modules:
- High churn modules:
- Performance bottlenecks:

---

## 3. Target State Principles
- Services aligned to **business capabilities (bounded contexts)**
- Independent deployment and scaling
- Database per service (end-state)
- Backward compatibility during migration
- No distributed transactions by default
- Strong observability (logs, metrics, traces)