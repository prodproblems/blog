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

Imagine a payments API handling about 500 requests per second. A query that normally completes in roughly 50 milliseconds begins taking 5 seconds after a release or a change in data volume.

The API starts returning timeouts and gateway errors. Application CPU is around 40%, memory looks normal, and the database is still accepting connections—but the service is unhealthy.

## 2. What engineers observe

Look at correlated signals rather than one dashboard in isolation:

- API p95/p99 latency rises and 504 responses increase.
- The database connection pool is at its maximum; requests wait to acquire a connection.
- Database query latency or wait events show the slow path.
- Application CPU may remain moderate because threads are blocked waiting for database work.
- Failures may concentrate on one endpoint or query shape.

A reachable database does not mean the database is healthy. It may be overloaded, waiting on I/O or locks, or executing an inefficient plan.

## 3. Customer and business impact

Customers may see failed or delayed requests. In payment flows, a timeout can leave the outcome unclear: the payment may have succeeded even though the response did not reach the client. Retries should therefore be designed around idempotency and safe status checks, not assumed to be harmless.

## 4. What could be causing it?

Build and test hypotheses using evidence:

1. A recent release changed the query or its parameters.
2. A query plan changed as table size or data distribution changed.
3. A needed index is missing or unsuitable.
4. Lock contention, storage latency, or database resource pressure increased.
5. The application is leaking connections or holding them longer than expected.

Compare the time of the first latency increase with deployment events, query statistics, pool metrics, and database wait information.

## 5. What is actually happening?

A representative query might be:

```sql
SELECT id, amount, status, created_at
FROM payments
WHERE customer_id = ?
ORDER BY created_at DESC
LIMIT 50;
```

Without a suitable access path, the database may examine and sort far more rows than expected. The exact cause must be confirmed from the actual query plan and runtime evidence; the SQL alone does not prove that an index is missing.

A simplified pool-capacity illustration: if 50 connections each stay busy for 50 ms, the rough service capacity is 50 / 0.05 = 1,000 query executions per second. If each occupies a connection for 5 seconds, the same simple model gives 50 / 5 = 10 executions per second. This is an intuition aid, not a universal throughput formula; real capacity also depends on concurrency, query mix, database resources, and queueing.

As connection hold time increases, all 50 connections can become occupied. New requests queue while waiting for a connection, then exceed application or gateway deadlines. The database need not be completely down for the API to fail.

## 6. Mitigate the incident

Choose the safest action supported by evidence and the service's operating procedures:

- If the issue clearly correlates with a release, consider rolling back that release.
- Disable or shed the problematic operation if the business flow safely allows it.
- Reduce excess load or retry pressure where possible.
- Use a restart only as temporary relief if needed and safe. It may release stuck application-side resources, but it does not fix the slow query; saturation can return as traffic resumes.

Protect payment flows with idempotency and reliable status reconciliation. Avoid blindly retrying requests that may already have succeeded.

## 7. Fix the root cause

First identify the actual query, parameters, plan, and waits. In PostgreSQL, inspect query statistics and use `EXPLAIN` / `EXPLAIN ANALYZE` appropriately. **`EXPLAIN ANALYZE` executes the statement**, so evaluate the operational risk before running it on a busy production system; use safe representative data or a controlled window when appropriate.

If evidence shows the filter and sort need a matching index, a candidate for evaluation might be:

```sql
CREATE INDEX CONCURRENTLY idx_payments_customer_created
ON payments (customer_id, created_at DESC);
```

This is PostgreSQL-specific. Check whether an equivalent index already exists, validate the plan and write workload, and follow the deployment team's practices. `CREATE INDEX CONCURRENTLY` has operational caveats and cannot run inside a transaction block. Do not add an index solely because this example contains one.

Test with production-like data volume and distribution, realistic concurrency, and representative parameters before deploying.

## 8. Engineering trade-offs

- **Increase pool size:** can help if the pool is undersized, but can increase database contention when queries are already slow.
- **Add an index:** may improve reads, but consumes storage and adds write and maintenance overhead.
- **Tighten timeouts:** can fail requests sooner and free resources, but does not make the underlying query faster.
- **Cache results:** may reduce database work, but introduces freshness, invalidation, and correctness considerations.

Timeouts should be designed as a coherent budget across database execution, pool acquisition, application processing, and gateway deadlines. The exact ordering depends on the architecture. The general principle is to stop work at the closest practical layer before it consumes resources beyond the caller's deadline. For HikariCP, `connectionTimeout` controls how long callers wait to acquire a pooled connection; it is not the SQL execution timeout.

## 9. Prevent recurrence

- Track query latency and pool acquisition wait separately.
- Alert on pool saturation, queue depth, query regressions, and endpoint p95/p99.
- Review query plans against representative data volumes before risky releases.
- Test under realistic concurrency, including the effect of retries.
- Set clear end-to-end timeout budgets and bound retries with backoff and jitter.
- Use idempotency and reconciliation for operations where a timeout can obscure the outcome.
- Document rollback and traffic-shedding actions in a runbook.

## 10. FDE challenge

> **You are on call:** an API starts returning 504s. CPU is at 40%, memory is normal, the database pool is 50/50, and one query's latency has jumped from 50 ms to 5 seconds.

What do you investigate first, what signal would confirm your hypothesis, and how do you reduce customer impact without creating duplicate payment operations?

A strong answer separates **mitigation** from **root-cause correction**: correlate the pool wait with the slow query, reduce the immediate blast radius, then validate and fix the query plan or underlying database bottleneck.

---

### Watch the episode

This article is the technical companion to the Real Production Problems episode. The video distills the incident into a short walkthrough; this page keeps the deeper reasoning, caveats, and operational checks.

[Visit the YouTube channel](https://www.youtube.com/@RealProductionProblems) ↗
</div>