---
title: "Production Experience: A 4-Hour Incident and What It Taught Us"
date: 2026-06-22
tags:
  - SRE
  - PostgreSQL
  - JVM
  - Production
summary: A real production incident where DB failover behavior, DNS caching, and cross-team knowledge gaps extended MTTR to 4 hours.
---

## Why Major Incidents Reveal Knowledge Gaps

Major incidents are rarely caused by one component alone. They expose:

- Hidden dependencies between infrastructure layers
- Operational blind spots across teams
- Weaknesses in runbooks and escalation flow
- Gaps in shared understanding of platform behavior

This incident is a good example.

## Incident Context

A Java application running on multiple VMs started throwing:

> `cannot UPSERT on a read-only DB`

In simple terms, the application was attempting writes on a PostgreSQL node that was not the active primary in a 3-node cluster.

Within 15 minutes, alerts crossed 10K failures in 5-minute intervals and a support bridge was opened.

## What Happened During Investigation

### T+0 to T+1 hour

- DB team validated cluster health and reported no active DB-side changes
- Application team noted they had seen similar transient errors during primary switches
- DB team observed open connections on secondary/remote servers
- Clearing those connections did not solve the issue

Core question remained:

**Why was the application suddenly connecting to non-primary nodes?**

### T+1 to T+2 hours

- Additional teams joined: Load Balancer, Infra Ops, downstream application owners
- Load balancer team observed VIP pool instability and intermittent health-check failures
- Similar incidents were reported by other applications
- System returned to partial BAU at times, then regressed

At this stage, this was treated as an enterprise-level DB incident without a clear root cause.

### T+2 to T+3 hours

- Errors crossed 3M at application layer
- DB logs showed `Postgres process shutting down` around incident start on master nodes
- Engineering suggested restarting one impacted app instance
- Restart worked immediately on that instance

A temporary resolution was identified:

**Restart impacted application instances**

## Key Root Cause Insights

DB SME analysis clarified the sequence:

1. When load balancer health checks failed, DNS lookups for DB VIP began round-robin behavior across cluster nodes.
2. Application connections landed on read-only nodes, causing UPSERT failures.
3. Even after health checks recovered, JVM processes reused existing TCP connections and did not force fresh DNS resolution.
4. Patroni leader checks (30s interval) and LB health checks (5s interval) had a timing mismatch during transient restart windows.

## Why Resolution Time Matters in Production

This issue lasted ~4 hours. A focused restart/runbook action could have reduced MTTR significantly.

Longer incident duration caused:

- Large-scale user-facing errors
- Multi-team paging and escalation fatigue
- Communication overhead (50-100 people on bridge)
- Delayed root-cause convergence due to parallel assumptions

## My Technical Takeaways

- **Connection reuse + DNS caching** in JVM can prolong failures after backend topology changes.
- **Health-check interval mismatch** (Patroni vs LB) can create instability windows.
- **Cross-team knowledge gaps** can dominate MTTR more than pure technical failure.
- **Runbooks should include validated fast mitigations** (e.g., controlled app restart) while root cause analysis continues.

.