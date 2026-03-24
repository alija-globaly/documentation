# Agentcis Technical Documentation

---

## 1. Introduction

This document provides a technical overview of the Agentcis CRM platform. It is intended for engineers onboarding to the system and covers:

System architecture and infrastructure
URL routing and request flow
Multi-tenancy implementation
Logging standards
Health monitoring
Asynchronous processing

---

## 2. System Overview

Agentcis is a multi-tenant SaaS CRM designed for agencies to manage clients, leads, communications, and workflows. Each tenant is isolated at the database level and accessed via a unique subdomain (tenant.agentcis.com).


### 2.1. Core Stack:

- Backend: PHP 8, Laravel Framework
- Frontend: Vue.js 2.x (Webpack / Laravel Mix)
- Microservices: Node + gRPC (bulk import, onboarding, campaign modules)
- Database: MySQL (1 master, multiple tenant databases)
- Web Server: Nginx + PHP-FPM
- Cache / Queue: DragonFly (cache), LavinMQ (message broker)


### 2.2 Architecture Pattern

Agentcis uses a hybrid architecture:

Modular Monolith: Core Laravel application
Dedicated Microservices: Node.js services for high-load modules
Multi-Tenant: Tenant isolation is maintained at the database and routing layer

---

## 3. URL Structure & Routing

### 3.1. URL Pattern

```bash
Pattern:   {subdomain}.{domain}.{tld}/api/{version}/{resource}?{params}

Example:   demo.agentcis.com/api/v2/email?email=test@gmail.com

```

### 3.2. URL Component Reference

| Component | Example | Description |
| --- | --- | --- |
| subdomain | demo | Identifies the tenant; resolved at the ALB. |
| domain | agentcis | Root product domain. |
| tld | .com | Top-level domain. |
| /api/{version} | /api/v2 | API version prefix. v2 is current stable. |
| {resource} | /email | Target API resource or endpoint group. |
| {?{params}} | ?email=test@gmail.com | Optional query parameters. |



### 3.3. DNS & Traffic Routing

- Wildcard DNS: *.staging.agentcis.com managed via Cloudflare → points to ALB (CNAME).
- ALB Routing: Evaluates host header and listener rules → forwards traffic to target groups (EC2 / pods).

### 3.4. Traffic Flow Diagram

Subdomain resolution is managed via Cloudflare using a wildcard DNS record. CNAME records point to the AWS Application Load Balancer (ALB).


```bash
User Browser <br>
&nbsp;&nbsp;&nbsp;&nbsp;      │
tenant.agentcis.com <br>
&nbsp;&nbsp;&nbsp;&nbsp;      │
Cloudflare DNS (CNAME ➔ ALB DNS) <br>
&nbsp;&nbsp;&nbsp;&nbsp;      │
ALB Listeners (HTTP/HTTPS) ➔ Rules & Priority ➔ Target Groups <br>
&nbsp;&nbsp;&nbsp;&nbsp;      │
EC2 Instances (Frontend + Backend) <br>
&nbsp;&nbsp;&nbsp;&nbsp;      │
Microservice calls ➔ K8s Pods

```



---

## 4. Multi-Tenancy via Subdomain

- Each tenant is assigned a unique subdomain.
- Tenant context is resolved at the ALB or ingress level, eliminating the need to pass tenant IDs in session tokens.
- Downstream routes are automatically tenant-scoped.
- Example tenants: demo.agentcis.com, client1.agentcis.com

---

## 5. System Flow & Architecture

### 5.1. Architecture Layer Reference

| Component | Technology | Responsibility |
| --- | --- | --- |
| Load Balancer | AWS ALB | Entry point; terminates TLS, evaluates routing rules, forwards traffic. |
| Frontend | Vue.js / Nginx | erves UI; proxies API requests to backend. |
| Backend | Laravel / PHP-FPM |Business logic, authentication, tenant resolution, API processing. |
| Cache | DragonFly | Redis-compatible in-memory cache; reduces DB load for repeated reads. |
| Database | MySQL | Master handles writes; replicas handle read-heavy queries. |
| Message Broker | LavinMQ | Queues asynchronous tasks from the backend. |
| Consumer | Worker Process | Subscribes to queues and processes jobs asynchronously. |


### 5.2. Request Lifecycle

| | |
| :--- | :--- |
| **1** | **Client Request**<br>Client sends HTTP/HTTPS request to `{subdomain}.agentcis.com/api/v2/{resource}?params` |
| **2** | **Load Balancer**<br>: TLS terminated; routing rules evaluated; request forwarded to available EC2 / pod. |
| **3** | **Frontend Layer**<br>Nginx serves static assets or proxies API requests to Laravel backend. |
| **4** | **Cache Lookup**<br>Backend queries DragonFly; cache hit → response returned. |
| **5** | **Database Query**<br>Cache miss → query MySQL (writes → Master, reads → Replica). |
| **6** | **Async Offload**<br>Long-running tasks (emails, imports, reports) are published as messages to LavinMQ queues. |
| **6** | **Consumer Processing**<br>Worker processes consume queued jobs asynchronously. |


### 5.3 Database Configuration


| Node  | Role | Operations |
| ------------- | ------------- | ------------- |
| Master  | Primary read/write  | Handles all INSERT, UPDATE, DELETE; source of truth.  |
| Replica  | Read-only  | Read-heavy queries routed here to offload master. |


### 5.4 Cache — DragonFly

- Redis-compatible, in-memory cache.
- Stores frequently accessed data (tenant configs, session info).
- Accessed only by backend; frontend does not query cache directly.

---

## 6. Logging Structure
Agentcis uses structured JSON logging for consistent, machine-readable logs correlated by tenant and timestamp.

### 6.1 Standard Log Schema

| Field  | Description |
| ------------- | ------------- |
| date  | Event date (YYYY-MM-DD)  |
| time  | Timestamp of event  |
| tenant  | Tenant identifier |
| msg   | Human-readable description |


### 6.2. Sample Log Entry


```bash
{
  "date":   "2024-01-15",
  "time":   "14:32:01",
  "tenant": "demo",
  "msg":    "User login successful"
}

```

### 6.3 Log Priority Levels

| Level | Name | Trigger Conditions | Action Required |
| --- | --- | --- | --- |
| P1 | Critical / Error | System failures, unhandled exceptions, data integrity errors | Immediate investigation required |
| P2 | Warning | Slow queries, retries, non-fatal errors, degraded performance | Monitor; investigate |
| P3 | Info / Debug | Standard operational events, request traces, routine actions | No action; used for auditing |




---

## 7. Health Monitoring
Agentcis implements two health check endpoints at the pod (microservice) and application (backend) level. These are polled by AWS ALB and Kubernetes liveness/readiness probes.

### 7.1 Health Check Endpoints

| Endpoint | Method | Purpose | Frequency / Polling | Logging |
| --- | --- | --- | --- | --- |
| /health | GET | Basic liveness/readiness check. Confirms pod or instance is running. | Every 30 s | Suppressed. No log entries generated. |
| /health/detailed | GET | Full diagnostic — validates all sub-components. | Checked once at pod startup | Logged once at startup. Not polled continuously. |



## 8. Asynchronous Processing — LavinMQ
- LavinMQ is the asynchronous message broker used by Agentcis.
- Purpose: Offload long-running tasks from synchronous HTTP requests to keep APIs responsive.
- Usage:
 - Backend publishes events/tasks to queues
 - Worker processes subscribe to queues and process tasks asynchronously
 - Common tasks include: sending emails, bulk data import/export, and report generation
