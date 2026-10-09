---
layout: default
title: Home
---

<section class="hero">
  <div class="hero-copy">
    <p class="eyebrow"><span class="status-dot"></span> FIELD NOTES FROM PRODUCTION</p>
    <h1>Systems fail.<br><span>Learn why.</span></h1>
    <p class="hero-description">Real engineering incidents, broken down into the signals, decisions, and fixes that matter when production is on fire.</p>
    <div class="hero-actions">
      <a class="button-primary" href="{{ '/articles/slow-database-query-api/' | relative_url }}">Explore the first breakdown <span aria-hidden="true">→</span></a>
      <span class="read-time">NO FLUFF. JUST ENGINEERING.</span>
    </div>
  </div>
  <div class="hero-visual" aria-label="Illustration of a production service under load">
    <div class="visual-top"><span class="window-dots"><i></i><i></i><i></i></span><span>LIVE SYSTEM / 01</span><span class="live-pill"><b></b> INCIDENT</span></div>
    <div class="visual-label">API LATENCY <span>↑ 8.4s</span></div>
    <div class="chart-area">
      <div class="chart-grid"></div>
      <svg viewBox="0 0 360 110" role="img" aria-label="Latency line rising sharply" preserveAspectRatio="none">
        <defs><linearGradient id="chartFill" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#ff705f" stop-opacity=".28"/><stop offset="100%" stop-color="#ff705f" stop-opacity="0"/></linearGradient></defs>
        <path d="M0,90 L30,86 L58,88 L88,79 L115,82 L142,65 L168,70 L190,44 L218,49 L242,24 L265,30 L286,10 L315,16 L340,3 L360,7 L360,110 L0,110 Z" fill="url(#chartFill)"/>
        <path d="M0,90 L30,86 L58,88 L88,79 L115,82 L142,65 L168,70 L190,44 L218,49 L242,24 L265,30 L286,10 L315,16 L340,3 L360,7" fill="none" stroke="#ff806f" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
    </div>
    <div class="visual-stats"><div><span>DB POOL</span><strong class="bad">50 / 50</strong></div><div><span>APP CPU</span><strong>40%</strong></div><div><span>HTTP 504</span><strong class="bad">RISING</strong></div></div>
    <div class="visual-bottom"><span class="pulse"></span> One slow query. Whole API waiting.</div>
  </div>
  <div class="hero-index"><span>01</span><span class="index-line"></span><span>INCIDENT ANALYSIS</span></div>
</section>

<section class="section articles-section">
  <div class="section-heading">
    <div><p class="eyebrow">THE INCIDENT LOG</p><h2>Understand the failure.<br><span>Improve the system.</span></h2></div>
    <span class="section-count">01 / BREAKDOWN</span>
  </div>
  <a class="feature-card" href="{{ '/articles/slow-database-query-api/' | relative_url }}">
    <div class="card-number">01</div>
    <div class="card-content">
      <div class="card-tags"><span>DATABASES</span><span>LATENCY</span><span>CONNECTION POOLS</span></div>
      <h3>One Slow Database Query Took Down an API</h3>
      <p>How a five-second query can occupy every database connection, back up requests, and turn a slow dependency into an API outage.</p>
      <span class="card-link">Read the breakdown <b aria-hidden="true">↗</b></span>
    </div>
    <div class="card-graphic" aria-hidden="true">
      <div class="stack-label">REQUEST FLOW</div>
      <div class="flow-node">API</div><div class="flow-arrow">↓</div>
      <div class="flow-node pool-node">DB POOL <span>FULL</span></div><div class="flow-arrow alert-arrow">↓</div>
      <div class="flow-node db-node">DATABASE <span>5.0s</span></div>
      <div class="flow-caption">QUEUE → TIMEOUT</div>
    </div>
    <span class="card-corner" aria-hidden="true">↗</span>
  </a>
</section>

<section class="section method-section">
  <div class="section-heading compact">
    <div><p class="eyebrow">THE RPP METHOD</p><h2>From alert to <span>action.</span></h2></div>
    <p class="method-intro">Not another tutorial. A practical framework for making better decisions under production pressure.</p>
  </div>
  <div class="method-grid">
    <article><span class="method-number">01</span><h3>Observe</h3><p>Read the signals. Separate symptoms from assumptions.</p></article>
    <article><span class="method-number">02</span><h3>Diagnose</h3><p>Test hypotheses and understand the failure mechanism.</p></article>
    <article><span class="method-number">03</span><h3>Recover</h3><p>Mitigate customer impact, then fix the root cause.</p></article>
    <article><span class="method-number">04</span><h3>Prevent</h3><p>Understand trade-offs and build safer systems.</p></article>
  </div>
</section>
<section class="closing-strip"><span class="closing-mark">R/P</span><p>Every incident leaves a lesson.<br><strong>Make the next failure less likely.</strong></p><span class="closing-tag">TEST HOW IT FAILS.</span></section>
