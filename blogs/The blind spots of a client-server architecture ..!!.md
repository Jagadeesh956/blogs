---
title: "The Blind Spots of a Client-Server Architecture"
date: 2026-06-22
tags: [SRE, Observability, Microservices, Reliability, Kubernetes]
summary: "Why client-side 503s can happen even when app logs look healthy, and how to close observability gaps across LB, ingress, sidecar, and app layers."
---

## Overview

A server exposing REST APIs and a client consuming them sounds simple.  
In real production systems, the request path is not just client-to-app.

In many microservice environments, it looks like this:

**Client -> Load Balancer -> Ingress Gateway -> Sidecar Proxy -> App Container**

That path introduces multiple failure points *before* traffic reaches application business logic.

---

## The Blind Spot Problem

Many teams measure request health only from application logs and controller metrics.  
That creates blind spots when failures happen upstream.

Common scenarios:

1. Request fails at security/filter layer, but logging starts only inside the controller.
2. No distributed tracing; only raw app logs exist, so dropped requests are invisible.
3. LB/Ingress/Sidecar errors are not correlated with app metrics in one timeline.

Result: the client receives `503 Service Unavailable`, but the app team says:

> "We do not even return 503 in our code."

Both statements can be true.

---

## Questions That Matter in Production

1. Where exactly did the 503 originate?
2. How do teams count requests that never reached the app container?
3. How accurate are availability metrics if upstream failures are excluded?
4. How can reliability metrics become realistic and decision-worthy?

---

## Why 503 Can Happen Without App Code Returning It

A 503 can be produced by:

- Load balancer target health failures
- Ingress gateway upstream timeout or retry exhaustion
- Sidecar proxy circuit-breaking due to unhealthy upstream endpoints
- Infrastructure/network churn before app code execution

So if app logs show no 503, it does **not** mean no 503 occurred.

---

## What to Instrument End-to-End

To eliminate blind spots, instrument each hop in the chain:

### 1) Load Balancer Layer
- Target health state transitions
- Target registration/deregistration events
- Upstream connect timeouts and reset counts
- 4xx/5xx split by backend target group

### 2) Ingress Gateway Layer
- Request count by host/path/status
- Upstream latency and retry counts
- Saturation metrics (connections, queue depth)
- Access logs with trace IDs

### 3) Sidecar / Service Mesh Layer
- Per-service success rate, retries, resets
- Circuit-breaker open/close events
- TLS/mTLS handshake failures
- Upstream endpoint membership changes

### 4) Application Layer
- Request timing from earliest filter/middleware entry point
- Error classification (auth, dependency timeout, validation, unknown)
- Structured logs enriched with trace and span IDs

---

## Practical Reliability Improvements

- Start request timing before controller methods.
- Propagate correlation IDs from edge to app and back to client.
- Build dashboards combining LB + ingress + mesh + app metrics.
- Alert on full-path error budget burn, not app-only error rates.
- Add synthetic probes from client perspective for true user-facing health.

---

## Final Takeaway

Reliability is not an application-only concern.  
If observability starts too late in the request lifecycle, teams undercount failures and overestimate availability.

The key principle:

> Measure the whole request path, not just the code you own.
