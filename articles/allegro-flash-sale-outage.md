---
layout: article
title: How a 1-Złoty Phone Deal Took Down a Website
description: A documented Allegro outage explained through traffic spikes, scaling configuration, safe recovery, and prevention.
permalink: /articles/allegro-flash-sale-outage/
category: Scaling & Reliability
reading_time: 7 min read
episode: 004
---

## The incident in 3 minutes

A special offer can bring a sudden rush of customers to an online store. But what happens when the system cannot add capacity—even when the machines appear to have resources available?

On July 18, 2018, Allegro, a major Polish e-commerce platform, offered 100 Honor 7C smartphones for **1 Polish złoty each**. The offer attracted far more traffic than expected. Allegro reported that the website was unavailable for around 20 minutes.

This was not documented as a scam. The important engineering lesson is that a traffic spike exposed problems in how parts of the platform scaled and used resources.

**Primary source:** [Allegro’s official postmortem: “Why Allegro went down”](https://blog.allegro.tech/2018/08/postmortem-why-allegro-went-down.html)

> This article separates facts reported by Allegro from proposed engineering practices. Recommendations below are not claims that Allegro used each practice during this incident.

## 1. What engineers observe

During a flash sale, engineers need to understand the scope and sequence of failure—not just notice that traffic is high.

Signals to check include:

- **Customer impact:** page-load failures, failed requests, and a drop in successful purchases.
- **Traffic:** request rate compared with the normal baseline and the timing of the promotion.
- **Availability:** HTTP 5xx responses, timeouts, and health-check failures.
- **Latency:** p50, p95, and p99 response times across the request path.
- **Scaling:** desired versus running instances, startup failures, pending workloads, and autoscaler events.
- **Resource constraints:** CPU, memory, quotas, reservations, connection pools, and file-descriptor usage.
- **Recent changes:** deployment, scaling configuration, resource requests, and health-check settings.

Allegro reported peak traffic at roughly **200–300% of the previous day’s level, depending on the service**. Its postmortem described excessive resource reservations and scaling-configuration errors that prevented parts of the cluster from starting new instances despite available CPU, memory, and network capacity.

Do not reduce the incident to “the CPU was full.” The postmortem describes several contributing issues, including service response times, health checks, connection pools/file descriptors, and circuit-breaker configuration.

## 2. What was actually happening?

A simplified scaling flow is:

`Traffic rises → Existing instances get busy → Platform tries to add instances → Some instances cannot start → Capacity falls behind demand`

Autoscaling is not magic. A platform can have spare physical resources while an application is still unable to start more instances because of configuration constraints or resource reservations.

Think of a building with empty rooms, but a booking rule mistakenly says no more people can enter. The space exists; the rule prevents it from being used.

The documented lesson from Allegro is that **available infrastructure resources do not automatically mean an application can scale**. Engineers must inspect the scheduler and scaling configuration, the application’s resource requirements, and the other limits that affect whether new instances can start and become healthy.

## 3. Diagnose: establish evidence before changing things

A disciplined investigation should answer these questions in order:

1. **When did customer impact begin?** Correlate the offer, traffic increase, deployment/configuration changes, and error timeline.
2. **Which services failed first?** Follow request paths and dependency health rather than assuming every component failed simultaneously.
3. **Was capacity genuinely exhausted?** Compare observed usage with available CPU, memory, quotas, reservations, and connection limits.
4. **Why did scaling fail?** Inspect desired/running instance counts, scheduling events, startup logs, readiness/health checks, and autoscaler decisions.
5. **What else amplified the incident?** Check latency, connection-pool exhaustion, file descriptors, watchdog restarts, and circuit-breaker behaviour.
6. **What is the smallest safe containment action?** Identify non-critical work that can be reduced without compromising customer data or core transactions.

### Use AI as an engineering assistant

Give a coding assistant sanitized metrics, relevant logs, scaling events, and configuration. Ask it to separate observed facts from hypotheses and list the next checks that would distinguish them.

Example prompt:

> Investigate this simulated flash-sale scaling incident using the supplied sanitized metrics, logs, events, and configuration. Separate confirmed observations from hypotheses. Explain why new instances may fail to start when CPU and memory appear available. Identify the next diagnostic checks and the safest reversible mitigations. Do not invent root causes, metrics, or production actions.

AI can help correlate evidence and generate hypotheses. An engineer must verify them against telemetry and own every operational decision.

## 4. Fix: restore service safely

**What Allegro reported doing:** during recovery, the team added resources and shut down non-critical services to free CPU and memory. The platform became responsive again at around 12:20, according to the postmortem.

For a similar incident, a safe response plan could be:

1. **Stabilize the service.** Reduce or pause non-critical work where possible, and use controlled load shedding if needed.
2. **Add capacity where it can actually become usable.** Confirm that new instances can be scheduled, start, pass health checks, and serve requests.
3. **Correct blocking configuration.** Review resource reservations, scaling limits, quotas, and other constraints that prevent additional instances from starting.
4. **Protect critical paths.** Keep customer-critical operations available while reducing optional background work.
5. **Validate recovery.** Watch successful requests, error rates, latency, queue depth, and instance health—not just the number of machines.

Adding servers alone is not a complete fix if the configuration still prevents the application from using them. Avoid broad restarts or unverified configuration changes that could increase the blast radius.

## 5. Fix the root cause

The exact corrective changes must be based on the postmortem and the evidence in the affected system. For a system with similar failure modes, engineering work should include:

- Review resource requests and limits so they reflect realistic workload needs.
- Validate autoscaling rules and maximum/minimum instance settings.
- Check quotas, reservations, scheduling constraints, and startup/readiness requirements.
- Audit connection pools and file-descriptor limits under peak concurrency.
- Review health checks and watchdog behaviour so temporary slowness does not create a restart loop.
- Verify circuit-breaker thresholds and fallback behaviour under degraded dependencies.
- Test whether critical services can start and remain healthy during a sudden demand increase.

These are **recommended engineering checks**, not a claim that every item was the root cause or the exact fix used by Allegro.

## 6. Validate with AI-assisted tests

AI can draft test cases, configuration checks, load profiles, and scripts. Review the generated changes before running them, and clearly distinguish proposed tests from tests actually executed.

A practical test matrix:

- Generate a sudden traffic increase and confirm that scaling responds within the required time.
- Verify new instances can be scheduled and pass health checks under peak load.
- Test resource reservations and limits against the expected workload.
- Simulate a slow dependency and confirm that connection pools and file descriptors remain within safe limits.
- Verify non-critical work can be reduced without breaking critical customer journeys.
- Test circuit-breaker and health-check behaviour during transient latency.
- Confirm rollback or configuration recovery procedures work in a controlled environment.

Measure successful request rate, p95/p99 latency, HTTP 5xx rate, queue depth, startup time, and the gap between desired and running instances. Do not declare success merely because a load test completed.

## 7. Prevent recurrence

Prevention requires more than increasing the server count:

- **Load-test promotions:** model expected traffic and a credible burst above the forecast.
- **Exercise autoscaling:** verify that instances can actually start, become healthy, and receive traffic.
- **Monitor scaling health:** alert on desired-versus-running instance gaps and repeated startup failures.
- **Watch saturation and constraints:** track CPU, memory, reservations, quotas, connection pools, and file descriptors.
- **Protect critical services:** define which non-critical workloads can be paused or shed during an incident.
- **Review failure amplification:** test health checks, watchdogs, circuit breakers, and dependency timeouts together.
- **Prepare an incident runbook:** document who can pause a promotion, reduce non-critical work, change capacity, and validate recovery.

Set thresholds using the service’s baseline and service-level objectives. Avoid universal alert numbers that ignore workload differences.

## 8. Trade-offs and blast radius

- **Adding resources** can restore headroom quickly, but may not help when configuration prevents instances from starting.
- **Reducing non-critical work** frees capacity, but some background jobs may be delayed.
- **Aggressive health checks** can detect genuinely unhealthy instances faster, but may remove slow instances during a temporary traffic spike.
- **Higher resource reservations** can protect an instance’s expected performance, but overly conservative reservations may prevent the platform from scheduling more instances.
- **Load shedding** protects the core service, but some users will receive a controlled failure instead of a slow or cascading outage.

The goal is not to serve every request at any cost. It is to protect the most important customer journeys and recover in a controlled way.

## 9. The Staff Engineer / FDE challenge

**Scenario:** A flash sale starts. Request volume triples. CPU and memory dashboards show spare capacity, but the number of running instances does not increase and HTTP 5xx errors climb.

What do you inspect first—and what would you avoid doing?

A strong answer:

1. **Diagnose:** compare desired and running instances; inspect scheduling events, resource reservations, quotas, startup/health checks, and the timeline of errors.
2. **Fix:** contain the blast radius, reduce non-critical load if safe, and correct the verified scaling constraint. Confirm new instances can serve traffic.
3. **Prevent:** add burst-load tests, scaling-health alerts, configuration validation, and a rehearsed recovery plan.

Avoid assuming the database, CPU, or autoscaler is the root cause before evidence supports it. Avoid making several risky changes at once, because that makes cause and effect harder to establish.

### Interview follow-up questions

- How would you distinguish resource saturation from a scheduling/configuration constraint?
- Which metrics prove that autoscaling is working end to end?
- How can health checks worsen an overload incident?
- What is your safe rollback or load-shedding strategy?
- How would you test this failure mode before the next promotion?

## 10. Key takeaway

A traffic spike may trigger the incident, but the deeper problem can be a system that cannot turn available resources into usable capacity.

**Diagnose → Fix → Prevent**

Find the constraint using evidence. Restore the critical customer path safely. Then test scaling, monitor its failure modes, and rehearse recovery before the next peak.

---

## Primary source

[Allegro Engineering — Postmortem: Why Allegro went down](https://blog.allegro.tech/2018/08/postmortem-why-allegro-went-down.html)

The incident details above are based on Allegro’s published postmortem. The recommendations and test scenarios are engineering guidance for systems with similar failure modes; they are not presented as actions Allegro necessarily took.
