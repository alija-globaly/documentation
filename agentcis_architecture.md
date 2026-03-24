# Agentcis Technical Documentation

---

## 1. Introduction

This doc provides a comprehensive technical overview of the Agentcis CRM platform. It is intended for engineers onboarding to the platform and covers system architecture, infrastructure, URL routing, request flow, multi-tenancy, logging, health monitoring, and asynchronous processing.

---

## 2. System Overview

Agentcis is a multi-tenant CRM designed for agencies managing clients, leads, communications, and workflows. Each tenant is isolated at the database level and accessed via a unique subdomain (tenant.agentcis.com)

### 2.1. Core Stack:

- PHP (Laravel Framework)
- Frontend powered by Vue.js, bundled with Webpack/Laravel Mix
- Microservices (Node.js + gRPC) 
- MySQL (one master and multiple tenant databases)
- Nginx + PHP-FPM
- LavinMQ, Dragonfly


### 2.2 Architecture Pattern

Architecture Pattern: uses a hybrid architecture, specifically a Modular Monolith (Laravel/PHP) that orchestrates dedicated Microservices. The entire application is Multi-Tenant.
Agentcis follows a Modular Monolith pattern for its core Laravel application, with discrete microservices (Node.js + gRPC) handling specialised modules such as bulk import, onboarding, and campaign processing. This approach balances deployment simplicity with the scalability benefits of isolated services for high-load workloads.

---

## 3. URL Structure & Routing

### 3.1. URL Pattern

All incoming traffic follows a standardised URL convention that encodes tenant identity and API versioning directly within the URL path:
```bash
Pattern:   {subdomain}.{domain}.{tld}/api/{version}/{resource}?{params}

Example:   demo.agentcis.com/api/v2/email?email=test@gmail.com

```

### 3.2. URL Component Reference

Agentcis uses a hybrid architecture, specifically a Modular Monolith (Laravel/PHP) that orchestrates dedicated Microservices. The entire application is Multi-Tenant.

| Component | Example | Description |
| --- | --- | --- |
| subdomain | demo | Uniquely identifies the tenant. Resolved at the load balancer. |
| domain | agentcis | Root product domain. |
| tld | .com | Top-level domain. |
| /api/{version} | /api/v2 | API version prefix. Current stable: v2. Older versions maintained during deprecation grace period. |
| {resource} | /email | The targeted API resource or endpoint group. |
| {?{params}} | ?email=test@gmail.com | Optional query parameters passed to the route handler. |



### 3.3. DNS & Traffic Routing

Subdomain resolution is managed via Cloudflare using a wildcard DNS record. CNAME records point to the AWS Application Load Balancer (ALB).

```bash
Wildcard DNS record:   *.staging.agentcis.com  →  CNAME  →  ALB DNS

```

Traffic routing sequence:

- PHP (Laravel Framework)
- Frontend powered by Vue.js 2.x, bundled with Webpack/Laravel Mix
- Microservices (Node.js + gRPC) 
- MySQL (one master and multiple tenant databases)
- Nginx + PHP-FPM
- LavinMQ, Dragonfly


### 3.4. Traffic Flow Diagram

Subdomain resolution is managed via Cloudflare using a wildcard DNS record. CNAME records point to the AWS Application Load Balancer (ALB).

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

---

## 4. Multi-Tenancy via Subdomain

Each tenant (agency) is assigned a unique subdomain. 
Tenant resolved at the ALB layer.
Downstream routes automatically scoped by tenant context.
Example subdomains: demo.agentcis.com, client1.agentcis.com

---

## 5. System Flow & Architecture

### 5.1. Architecture Layer Reference

| Component | Technology | Responsibility |
| --- | --- | --- |
| Load Balancer | AWS ALB | Single entry point. Distributes traffic; enforces TLS termination. |
| Frontend | Vue.js / Nginx | Serves the UI. Proxies API calls to the backend. |
| Backend | Laravel / PHP-FPM | Business logic, authentication, tenant resolution, API processing. |
| Cache | DragonFly | Redis-compatible in-memory cache. Reduces DB load for repeated reads. |
| Database | MySQL | Persistent storage. Master handles writes; Replica handles reads. |
| Message Broker | LavinMQ | Receives async tasks published by the backend. |
| Consumer | Worker Process | Receives async tasks published by the backend. |


### 5.2. Request Lifecycle

| | |
| :--- | :--- |
| **1** | **Client Request**<br>Browser sends HTTP/HTTPS request to `{subdomain}.agentcis.com/api/v2/{resource}?params` |
| **2** | **Load Balancer**<br>AWS ALB terminates TLS, evaluates routing rules, and forwards to an available EC2 instance. |
| **3** | **Frontend Layer**<br>Nginx serves static assets or proxies the API request to PHP-FPM / Laravel backend. |
| **4** | **Cache Lookup**<br>Backend queries DragonFly cache. On a cache hit, the response is returned immediately. |
| **5** | **Database Query**<br>On a cache miss, backend queries MySQL. Writes go to Master; reads go to Replica. |
| **6** | **Async Offload**<br>Long-running tasks (emails, imports, reports) are published as messages to LavinMQ queues. |
| **6** | **Async Offload**<br>Worker processes consume queued jobs asynchronously, independent of the HTTP request cycle. |


### 5.3 Database Configuration


| Node  | Role | Operations |
| ------------- | ------------- | ------------- |
| Master  | Primary read/write  | All writes: INSERT, UPDATE, DELETE. Source of truth.  |
| Replica  | Read-only replica  | Read-heavy queries routed here to offload the master node.  |


### 5.4 Cache — DragonFly

DragonFly: Redis-compatible in-memory cache.
Stores frequently accessed data (tenant configs, session data).
Backend integrated; frontend does not access cache directly.


---

## 6. Logging Structure
Agentcis uses structured JSON logging to provide consistent, machine-parseable log output across all services. Logs are correlated by tenant and timestamp to support efficient debugging and observability.

`Log Fields:` date, time, tenant, message



### 6.1 Standard Log Schema

| Field  | Description |
| ------------- | ------------- |
| date  | CDate of the log event. Format: YYYY-MM-DD.  |
| time  | Timestamp of the log event.  |
| msg   | Human-readable description of the event.  |


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

Log verbosity is configured per route. High-traffic or low-risk routes use lower priorities to reduce noise; critical paths are set to P1.

| Level | Name | Trigger Conditions | Action Required |
| --- | --- | --- | --- |
| P1 | Critical / Error | System failures, unhandled exceptions, data integrity errors. | Immediate investigation required. |
| P2 | Warning | Slow queries, retries, non-fatal errors, degraded performance. | Monitor; investigate if sustained. |
| P3 | Info / Debug | Standard operational events, request traces, routine actions. | No action. Used for audit trails. |




---

## 7. Health Monitoring
Agentcis implements two health check endpoints at both the pod (microservice) and application (backend) level. These endpoints are used by AWS ALB and Kubernetes liveness/readiness probes to ensure that only healthy instances receive traffic.


### 7.1 Health Check Endpoints

| Endpoint | Method | Purpose | Frequency / Polling | Logging |
| --- | --- | --- | --- | --- |
| /health | GET | Basic liveness check. Confirms pod or instance is up. | Every 30 s | Suppressed. No log entries generated. |
| /health/detailed | GET | Full diagnostic — validates all sub-components. | Once at startup | Logged once at startup. Not polled continuously. |



## 8. Asynchronous Processing — LavinMQ
`LavinMQ:` Asynchronous message broker
Keeps APIs responsive even under high load.

`Purpose:` Handle background tasks without blocking API responses
`Usage:`
- Backend publishes events/tasks to queues.
- Consumer service picks up and processes them asynchronously.
- Common tasks: sending emails, bulk data import/export, report generation.
