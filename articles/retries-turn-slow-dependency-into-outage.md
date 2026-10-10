---
layout: article
title: How Retries Turn a Slow Dependency into an Outage
description: Diagnose retry amplification, safely mitigate a retry storm, and design bounded, payment-safe retries.
permalink: /articles/retries-turn-slow-dependency-into-outage/
category: API Reliability
reading_time: 7 min read
episode: 002
---

## Watch the Day 2 video

<div class="video-embed">
  <iframe
    src="https://www.youtube-nocookie.com/embed/nBmzJtkKhmc"
    title="How Retries Turn a Slow Dependency into an Outage — Real Production Problems Day 2"
    loading="lazy"
    referrerpolicy="strict-origin-when-cross-origin"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>

[Watch directly on YouTube](https://youtu.be/nBmzJtkKhmc) ↗

## Day 2 — The incident

A payment API is handling about **1,000 original requests per second**. Its downstream provider can sustain roughly **1,200 requests per second** under the current workload. Provider latency rises and some requests time out. The API Gateway, application service, and HTTP client may each retry independently.

The service starts sending more work precisely when the dependency has less capacity to handle it.

This is a realistic incident scenario, not a claim about a specific company's outage. Treat the numbers as illustrative assumptions to validate against telemetry.

## 1. Diagnose: what engineers observe

Look for a pattern across metrics, traces, logs, and configuration:

- **Original request rate:** is customer traffic stable, or did it rise?
- **Attempt rate:** how many total downstream attempts are being made, including retries?
- **Latency and timeouts:** did provider latency rise before retry volume increased?
- **Saturation:** are queues, in-flight requests, connection pools, or concurrency limits filling up?
- **Trace shape:** does one logical request produce multiple downstream attempts? Which component initiates each retry?
- **Configuration and deployments:** do the gateway, application, SDK, or service mesh each have retry policies?

Do not assume retries are the root cause just because retry counts are high. Correlate the timeline and establish whether retry traffic is materially amplifying the dependency's workload.

### Use AI as an engineering assistant

Give an AI coding assistant the relevant, sanitized logs, traces, retry configuration, and code paths. Ask it to separate observed facts from hypotheses, identify every retry layer, calculate the maximum attempts under the actual settings, and recommend the next evidence to inspect.

Example investigation prompt:

> You are assisting with a production incident in a payment API. Original traffic appears stable while downstream latency, timeouts, and retry volume are increasing. Inspect the supplied logs, traces, client configuration, and code. Identify every retry layer and calculate the worst-case attempts from the actual configuration. Separate facts from hypotheses, identify evidence that would confirm or disprove retry amplification, and recommend safe next diagnostic steps. Do not invent metrics or claim root cause without evidence. Do not make production changes.

AI can accelerate investigation, but the engineer must verify the evidence and own the decision.

## 2. Understand the amplification

Suppose three layers each allow **three total attempts**: one initial attempt plus two retries. If each layer's attempts compose on the same failure path, the worst-case downstream amplification can be:

`3 × 3 × 3 = 27` attempts for one original logical request.

If each layer instead allows one initial attempt plus three retries, that is four attempts per layer and a theoretical worst case of:

`4 × 4 × 4 = 64` attempts.

These are **worst-case bounds, not expected or measured traffic**. Actual amplification depends on retry conditions, timing, cancellation behavior, and whether the layers' policies compose. A timeout at the caller also does not prove that the downstream operation stopped executing.

Capacity math needs the same care. A dependency capacity of 1,200 requests/second and original traffic of 1,000 requests/second does not by itself imply that traffic doubles. The actual attempt rate depends on the proportion of requests retried and how many attempts each request makes. Measure original requests and retry attempts separately.

## 3. Fix: mitigate the active incident safely

Prefer actions that reduce pressure without making payment outcomes ambiguous:

1. **Bound or reduce retries** at the layer shown by evidence to be amplifying load.
2. **Shed or defer low-priority work** if the service supports it safely.
3. **Apply concurrency limits or load shedding** to prevent an overloaded dependency from accumulating unbounded work.
4. **Use a circuit breaker where appropriate** to stop repeated calls during sustained failure and allow controlled recovery.
5. **Scale only when evidence supports it.** Adding capacity may help a true capacity bottleneck, but can fail to solve slow provider operations or simply move the bottleneck elsewhere.
6. **Avoid blind restarts.** They can briefly clear local queues while leaving the retry behavior unchanged.

Follow the service's incident procedures and validate the effect of each change through attempt rate, latency, error rate, saturation, and successful business outcomes.

## 4. Fix the root cause

Design retries as an explicit reliability policy, not as a default added independently at every layer.

- **One clear retry owner:** avoid stacking independent retries across the gateway, application, and client without a deliberate budget.
- **End-to-end deadline:** propagate a deadline so retries and downstream work cannot exceed the caller's useful time budget.
- **Explicit timeouts:** set coherent connection, request, and operation timeouts. A timeout should fit within the remaining deadline.
- **Bounded attempts:** cap attempts and stop when the deadline or retry budget is exhausted.
- **Exponential backoff with jitter:** spread retries over time instead of synchronizing them into bursts.
- **Retry only appropriate failures:** retry selected transient failures according to the provider contract. Do not retry validation or other non-retryable errors. Respect `Retry-After` when applicable.
- **Retry budget:** cap retry traffic relative to original requests over a defined window. For example, a team might evaluate a 10% retry budget; that is an illustrative policy, not a universal threshold.
- **Idempotency for payments:** reuse the same idempotency key when retrying the same logical payment, according to the provider's contract.

### The payment-specific rule

**A timeout is an unknown outcome, not proof of failure.**

If a payment request times out, the provider may have completed the payment while the response was lost. Do not create a new logical payment or blindly retry with a new idempotency key.

- If the provider confirms success, return or persist the confirmed result.
- If the provider confirms failure, follow the provider's documented retry rules.
- If the outcome is unknown, keep the transaction in a pending/unknown state and reconcile using the provider's status API, webhook, or other supported mechanism.

Idempotency reduces duplicate-operation risk, but its guarantees and retention period are provider-specific.

## 5. Validate the fix with AI-assisted tests

Ask the coding assistant to propose a small patch and regression tests. Require it to show the diff, explain the risks, and identify which tests it has merely proposed versus actually run.

A useful test matrix includes:

- A transient failure results in no more than the configured number of attempts.
- Non-retryable responses do not trigger retries.
- Backoff includes jitter and respects the remaining deadline.
- Retry-budget exhaustion stops additional attempts.
- Cancellation and timeout behavior do not leave unbounded work running where cancellation is supported.
- A payment retry uses the same idempotency key for the same logical operation.
- An uncertain payment outcome enters reconciliation rather than creating a duplicate payment.

Run tests in a controlled environment, inspect the results, and use fault injection or load tests to verify behavior under dependency latency and failure. Do not claim the fix is validated until the tests actually pass and the evidence is reviewed.

## 6. Prevent recurrence

Monitor both technical and business signals:

- Original logical requests versus total downstream attempts.
- Retry rate by layer and failure reason.
- Downstream latency, timeouts, in-flight requests, queue depth, and saturation.
- Retry-budget exhaustion and circuit-breaker state.
- Payment success, duplicate prevention, pending outcomes, and reconciliation lag.

Add regression tests and dependency-failure exercises to the delivery pipeline. Review retry configuration as part of service ownership, document safe mitigation steps in the runbook, and alert before retry traffic consumes the dependency's remaining capacity.

## 7. Trade-offs and FDE challenge

Retries can improve resilience to brief transient failures, but increase load and latency during sustained failure. Circuit breakers reduce pressure but can reject requests that might otherwise succeed. Aggressive timeouts free local resources sooner but can increase uncertain outcomes for operations already accepted downstream.

**Challenge:** An AI coding assistant suggests retrying a timed-out payment with a new idempotency key. Do you accept the patch?

A strong answer is **no**. First establish the provider's idempotency contract and check the transaction's status. Reuse the same key for the same logical payment; if the outcome is unknown, reconcile it instead of creating a second operation.

In an interview, structure your response as **Diagnose → Fix → Prevent**: prove the amplification with telemetry, mitigate safely, implement bounded and deadline-aware retry behavior, protect payment correctness, and verify the fix with tests and production signals.

---

*This article is the technical learning companion for RPP Day 2. The scenario and numeric values are illustrative; validate real incidents using actual evidence.*
