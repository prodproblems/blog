---
layout: article
title: One Slow Database Query Took Down an API
description: How a slow query can exhaust a connection pool and make an API fail.
permalink: /articles/slow-database-query-api/
category: Database Reliability
reading_time: 8 min read
episode: 001
---

## 1. The incident

- A payments API handles about **500 requests per second**.
- A query that normally completes in roughly **50 ms** begins taking **5 seconds** after a release or a change in data volume.
- The API starts returning timeouts and gateway errors.
- Application CPU is around 40%, memory looks normal, and the database is still accepting connections—but the service is unhealthy.

## 2. What engineers observe

Look at correlated signals rather than one dashboard in isolation:

- **API latency:** p95/p99 rises and 504 responses increase.
- **Connection pool:** the pool reaches its maximum and requests wait to acquire a connection.
- **Database:** query latency or wait events reveal the slow path.
- **Application:** CPU may remain moderate because threads are blocked waiting for database work.
- **Endpoint pattern:** failures may concentrate on one endpoint or query shape.
- **Important:** a reachable database is not necessarily healthy. It may be overloaded, waiting on I/O or locks, or executing an inefficient plan.

## 3. Customer and business impact

- Customers may see failed or delayed requests.
- In payment flows, a timeout can leave the outcome unclear: the payment may have succeeded even though the response did not reach the client.
- Retries must be designed around **idempotency and safe status checks**—not assumed to be harmless.

## 4. What could be causing it?

Build and test hypotheses using evidence:

1. A recent release changed the query or its parameters.
2. A query plan changed as table size or data distribution changed.
3. A needed index is missing or unsuitable.
4. Lock contention, storage latency, or database resource pressure increased.
5. The application is leaking connections or holding them longer than expected.

- Compare the first latency increase with deployment events, query statistics, pool metrics, and database wait information.
- Treat each cause as a hypothesis until the evidence confirms it.

## 5. What is actually happening?

A representative query might be:

```sql
SELECT id, amount, status, created_at
FROM payments
WHERE customer_id = ?
ORDER BY created_at DESC
LIMIT 50;
```

- Without a suitable access path, the database may examine and sort far more rows than expected.
- The actual cause must be confirmed from the query plan and runtime evidence. The SQL alone does **not** prove that an index is missing.

**A simplified connection-pool illustration:**

- With 50 connections occupied for 50 ms each, a simple estimate is **50 / 0.05 = 1,000 query executions per second**.
- If each connection is occupied for 5 seconds, the same simplified model gives **50 / 5 = 10 executions per second**.
- This is an intuition aid, not a universal throughput formula. Real capacity also depends on concurrency, query mix, database resources, and queueing.

What happens next:

- All 50 connections can become occupied as connection hold time increases.
- New requests queue while waiting for a connection.
- Requests then exceed application or gateway deadlines.
- The database does not need to be completely down for the API to fail.

## 6. Mitigate the incident

Choose the safest action supported by evidence and the service's operating procedures:

- **Consider rollback** if the issue clearly correlates with a release.
- **Disable or shed the problematic operation** if the business flow safely allows it.
- **Reduce excess load or retry pressure** where possible.
- **Restart only as temporary relief**, if needed and safe. It may release stuck application-side resources, but it does not fix the slow query; saturation can return as traffic resumes.
- **Protect payment flows** with idempotency and reliable status reconciliation. Avoid blindly retrying requests that may already have succeeded.

## 7. Fix the root cause

First identify the actual query, parameters, plan, and waits.

- In PostgreSQL, inspect query statistics and use `EXPLAIN` or `EXPLAIN ANALYZE` appropriately.
- **`EXPLAIN ANALYZE` executes the statement.** Evaluate operational risk before running it on a busy production system; use safe representative data or a controlled window when appropriate.

If evidence shows the filter and sort need a matching index, a candidate for evaluation might be:

```sql
CREATE INDEX CONCURRENTLY idx_payments_customer_created
ON payments (customer_id, created_at DESC);
```

- This example is PostgreSQL-specific.
- Check whether an equivalent index already exists and validate the query plan and write workload.
- `CREATE INDEX CONCURRENTLY` has operational caveats and **cannot run inside a transaction block**.
- Do not add an index solely because this example contains one.
- Test with production-like data volume and distribution, realistic concurrency, and representative parameters before deploying.

## 8. Engineering trade-offs

- **Increase pool size:** may help if the pool is undersized, but can increase database contention when queries are already slow.
- **Add an index:** may improve reads, but consumes storage and adds write and maintenance overhead.
- **Tighten timeouts:** can fail requests sooner and free resources, but does not make the underlying query faster.
- **Cache results:** may reduce database work, but introduces freshness, invalidation, and correctness considerations.
- **Set a coherent timeout budget:** account for database execution, pool acquisition, application processing, and gateway deadlines. The exact ordering depends on the architecture.
- **Know what the timeout controls:** HikariCP's `connectionTimeout` controls how long callers wait to acquire a pooled connection; it is not the SQL execution timeout.

## 9. Prevent recurrence

- Track query latency and pool-acquisition wait separately.
- Alert on pool saturation, queue depth, query regressions, and endpoint p95/p99.
- Review query plans against representative data volumes before risky releases.
- Test under realistic concurrency, including the effect of retries.
- Set clear end-to-end timeout budgets and bound retries with backoff and jitter.
- Use idempotency and reconciliation where a timeout can obscure an operation's outcome.
- Document rollback and traffic-shedding actions in a runbook.

## 10. FDE challenge

> **You are on call:** an API starts returning 504s. CPU is at 40%, memory is normal, the database pool is 50/50, and one query's latency has jumped from 50 ms to 5 seconds.

Answer these questions:

- What do you investigate first?
- Which signal would confirm your hypothesis?
- How do you reduce customer impact without creating duplicate payment operations?

A strong answer separates **mitigation** from **root-cause correction**:

- Correlate pool-acquisition wait with the slow query.
- Reduce the immediate blast radius.
- Validate and fix the query plan or underlying database bottleneck.

---

## Watch the episode

Watch the Day 1 walkthrough here:

<div style="position: relative; width: 100%; aspect-ratio: 16 / 9; overflow: hidden; border-radius: 12px; margin: 1rem 0;">
  <iframe
    src="https://www.youtube-nocookie.com/embed/LaUlmYq-KrY"
    title="One Slow Database Query Took Down an API | Real Production Problems"
    loading="lazy"
    style="position: absolute; inset: 0; width: 100%; height: 100%; border: 0;"
    referrerpolicy="strict-origin-when-cross-origin"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>

- [Watch directly on YouTube](https://youtu.be/LaUlmYq-KrY) ↗
- This article is the technical companion to the episode, with deeper reasoning, caveats, and operational checks.
- [Visit the YouTube channel](https://www.youtube.com/@RealProductionProblems) ↗


---

## Visual summary

These diagrams summarize the incident, failure mechanism, and recovery workflow.

### 1. The production incident

![Production incident: API Gateway, backend service, and PostgreSQL showing slow query symptoms]({{ '/assets/images/day-1-production-incident.svg' | relative_url }})

- A slow query increases request latency.
- The connection pool fills up and the API begins returning 504 errors.

### 2. Why the API fails

![Connection pool saturation: slow SQL occupies all connections and requests queue]({{ '/assets/images/day-1-connection-pool-failure.svg' | relative_url }})

- Long-running queries hold connections for longer.
- New requests wait for a free connection and can time out.

### 3. Mitigation, fix, and prevention

![Incident response workflow covering mitigation, diagnosis, and long-term prevention]({{ '/assets/images/day-1-mitigation-fix.svg' | relative_url }})

- Mitigate customer impact first.
- Confirm the root cause before changing queries or indexes.
- Add monitoring, alerts, and load tests to reduce recurrence.
