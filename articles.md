---
layout: default
title: Articles
description: Practical breakdowns of production incidents and engineering trade-offs.
permalink: /articles/
---

<div class="simple-page">
  <p class="eyebrow">THE INCIDENT LIBRARY</p>
  <h1>Production problems, explained.</h1>
  <p class="lead">Technical breakdowns focused on how systems fail and how engineers make them more reliable.</p>

  <a class="article-card" href="{{ '/articles/retries-turn-slow-dependency-into-outage/' | relative_url }}">
    <p class="eyebrow">02 · API RELIABILITY · 7 MIN READ</p>
    <h2>How Retries Turn a Slow Dependency into an Outage</h2>
    <p>How layered retries amplify load, how to investigate a retry storm, and how to protect payment flows with bounded retries and idempotency.</p>
    <span class="card-link">Read article →</span>
  </a>

  <a class="article-card" href="{{ '/articles/slow-database-query-api/' | relative_url }}">
    <p class="eyebrow">01 · DATABASE RELIABILITY · 8 MIN READ</p>
    <h2>One Slow Database Query Took Down an API</h2>
    <p>How slow query execution can exhaust a connection pool, build a request queue, and cause API timeouts.</p>
    <span class="card-link">Read article →</span>
  </a>

  <p class="muted">Each breakdown combines production diagnosis, safe mitigation, root-cause correction, prevention, and an engineering challenge.</p>
</div>