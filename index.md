---
layout: default
title: Home
description: Real engineering problems, explained simply.
---

<section class="home-intro">
  <p class="eyebrow">REAL PRODUCTION PROBLEMS</p>
  <h1>Real systems.<br><span>Real failures.</span></h1>
  <p class="lead">Understand why production systems fail, how engineers investigate, and what prevents the next incident.</p>
  <p><a class="button" href="{{ '/articles/' | relative_url }}">Explore the articles →</a></p>
</section>

<section class="home-section">
  <h2>Latest breakdown</h2>
  <a class="article-card" href="{{ '/articles/retries-turn-slow-dependency-into-outage/' | relative_url }}">
    <p class="eyebrow">API RELIABILITY · RETRY SAFETY</p>
    <h3>How Retries Turn a Slow Dependency into an Outage</h3>
    <p>How layered retries amplify downstream load, how to diagnose the feedback loop, and how to build bounded, payment-safe retries.</p>
    <span class="card-link">Read the breakdown →</span>
  </a>
  <a class="article-card" href="{{ '/articles/slow-database-query-api/' | relative_url }}">
    <p class="eyebrow">DATABASES · API RELIABILITY</p>
    <h3>One Slow Database Query Took Down an API</h3>
    <p>A slow query fills the connection pool. Requests queue up. The API starts timing out—even while the database is still responding.</p>
    <span class="card-link">Read the breakdown →</span>
  </a>
</section>

<section class="home-section method">
  <h2>Every breakdown covers</h2>
  <ul>
    <li>What engineers observe</li>
    <li>What is actually happening</li>
    <li>How to mitigate and fix the problem</li>
    <li>Trade-offs, prevention, and an engineering challenge</li>
  </ul>
</section>