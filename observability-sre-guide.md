# Observability & SRE Practices — Principal Engineer Reference

> File 6 of 7 in the Cloud-Native & DevOps series. Extends the observability foundations from the Java Testing & Production Readiness guide (§7) and Spring Testing & Observability guide (§6-7) into the full infrastructure stack — SLOs/error budgets, the Prometheus/Grafana/tracing ecosystem, incident management, and chaos engineering.

**Series:** 1. Docker & Containerization · 2. Kubernetes for Java · 3. CI/CD & GitOps · 4. Cloud Provider Deep Dive · 5. Infrastructure as Code · **6. Observability & SRE (this file)** · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [SLOs, SLIs & Error Budgets](#1-slos-slis--error-budgets)
2. [The Observability Stack — Prometheus, Grafana, Tracing](#2-the-observability-stack--prometheus-grafana-tracing)
3. [Log Aggregation at Scale](#3-log-aggregation-at-scale)
4. [Distributed Tracing Infrastructure](#4-distributed-tracing-infrastructure)
5. [Alerting Design](#5-alerting-design)
6. [Incident Management & On-Call](#6-incident-management--on-call)
7. [Postmortems & Blameless Culture](#7-postmortems--blameless-culture)
8. [Chaos Engineering](#8-chaos-engineering)
9. [A Worked Observability Stack](#9-a-worked-observability-stack)

---

## 1. SLOs, SLIs & Error Budgets

### 1.1 The Vocabulary, Precisely

```
SLI (Service Level Indicator) — a MEASURED metric: "% of requests completing under 200ms,"
  "% of requests returning a non-5xx status" — directly the Java Testing & Production
  Readiness guide's §7.3 RED-metrics percentile discipline, given a formal name here.
SLO (Service Level Objective) — a TARGET for an SLI: "99.9% of requests complete under
  200ms, measured over a rolling 30-day window" — an INTERNAL goal, more conservative
  than any customer-facing promise.
SLA (Service Level Agreement) — a CONTRACTUAL commitment to a CUSTOMER, with consequences
  (credits, penalties) for breach — should ALWAYS be set LESS strict than the internal SLO,
  giving genuine margin before a customer-facing breach becomes a contractual liability.
```

### 1.2 Error Budgets — Turning an SLO Into an Actionable Decision Tool

```
An SLO of "99.9% availability" implies an ERROR BUDGET of 0.1% — a concrete, spendable
allowance of acceptable failure (roughly 43 minutes per month). This REFRAMES a recurring
organizational tension explicitly: directly the Java Testing & Production Readiness guide
§6.2 canary/feature-flag risk-taking discussion, an error budget gives a DATA-DRIVEN answer
to "can we ship this risky change this week?" — if the error budget is ALREADY exhausted
for the period, the answer is "slow down, prioritize reliability work, defer risky
launches" — and if there's ample budget remaining, teams have EXPLICIT, quantified
permission to take on MORE deployment risk, rather than an perpetually-conservative,
un-quantified "let's be careful" instinct that never actually trades off against velocity explicitly.
```

### 1.3 Choosing SLIs That Actually Reflect User Experience

```
Directly the Java Testing & Production Readiness guide's §7.3 "percentiles, not averages"
discipline, extended: an SLI should be measured as CLOSE TO THE USER as architecturally
feasible — a load-balancer-measured latency SLI is more trustworthy than an application-
internal one, since it captures EVERYTHING between the user and the response, including
network/load-balancer-layer issues an internal metric would miss entirely. For
MULTI-STEP user journeys (a checkout flow spanning several API calls), consider a
COMPOSITE SLI reflecting the FULL journey's success, not just any single API call in isolation —
a checkout that fails at step 3 of 4 is a 100% failure from the user's perspective, even if
three of the four underlying API calls individually "succeeded" by their own narrow SLI.
```

---

## 2. The Observability Stack — Prometheus, Grafana, Tracing

### 2.1 Prometheus's Pull-Based Model

```yaml
# Prometheus SCRAPES metrics endpoints on a schedule — the INVERSE of a push-based
# system (where applications proactively send metrics) — directly enabled by Spring Boot
# Actuator's /actuator/prometheus endpoint (Spring Testing & Observability guide §6.1, §7.1)
scrape_configs:
  - job_name: 'order-service'
    kubernetes_sd_configs: [{ role: pod }]  # auto-discovers pods via Kubernetes' own
    relabel_configs:                           # API (File 2 of this series) — NEW pods are
      - source_labels: [__meta_kubernetes_pod_label_app]  # scraped automatically, with
        action: keep                                            # NO per-pod manual registration
        regex: order-service
```

```
The PULL model's genuine advantage: Prometheus itself can DETECT a target that's stopped
responding (a clear, unambiguous "down" signal) — a PUSH-based system can't as easily
distinguish "the application stopped sending metrics because it's down" from "the
application stopped sending metrics because of a network issue between it and the
collector," a meaningfully different diagnostic ambiguity that the pull model avoids.
```

### 2.2 PromQL — Querying Time-Series Data

```promql
# Directly computing the §1.1 SLI from raw counters — the SAME RED-metrics discipline,
# expressed as an actual, runnable query against real production data
sum(rate(http_requests_total{job="order-service", status!~"5.."}[5m]))
/
sum(rate(http_requests_total{job="order-service"}[5m]))

# p99 latency — directly the Testing & Production Readiness guide's §7.3 "ALWAYS
# percentiles, NEVER averages" discipline, computed from a Prometheus HISTOGRAM metric
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le))
```

### 2.3 Grafana — Dashboards as Code

```json
// Dashboards-as-code (a JSON/Jsonnet definition, version-controlled alongside application
// code) — directly the SAME GitOps/IaC discipline from Files 3 and 5, applied to
// OBSERVABILITY CONFIGURATION itself: a dashboard hand-edited through Grafana's UI and
// never saved anywhere durable is exactly the "click-ops" chaos File 5 §1.1 warns against,
// just for dashboards instead of infrastructure
{
  "title": "Order Service - RED Metrics",
  "panels": [
    { "title": "Request Rate", "targets": [{ "expr": "sum(rate(http_requests_total{job=\"order-service\"}[5m]))" }] },
    { "title": "Error Rate", "targets": [{ "expr": "..." }] },
    { "title": "p99 Latency", "targets": [{ "expr": "..." }] }
  ]
}
```

---

## 3. Log Aggregation at Scale

### 3.1 The Aggregation Pipeline

```
Application (structured JSON logs, Java Testing & Production Readiness guide §7.2) →
  a log-shipping agent (Fluentd/Fluent Bit/Promtail, often deployed as a Kubernetes
  DaemonSet — ONE per node, collecting EVERY pod's logs on that node) →
  a centralized aggregation/storage backend (Loki, Elasticsearch, a cloud-native log
  service) → queried via Grafana (for Loki) or Kibana (for Elasticsearch)
```

### 3.2 Structured Logging Discipline, Restated at Infrastructure Scale

```json
{"timestamp":"2026-01-20T14:32:01Z","level":"ERROR","service":"order-service","correlationId":"abc-123","tenantId":"acme-corp","message":"Payment gateway timeout","exception":"java.util.concurrent.TimeoutException"}
```

```
Directly the Java Testing & Production Readiness guide's §7.2 structured-logging
discipline, now essential at AGGREGATION scale specifically: UNSTRUCTURED log lines force
the aggregation backend into fragile, regex-based parsing to extract any queryable field —
at the volume a Kubernetes cluster running dozens of services generates, this parsing
fragility compounds into a genuinely significant operational liability, while structured
JSON logs are NATIVELY queryable/filterable (by tenantId, correlationId, service) the
MOMENT they're ingested, with no parsing-layer fragility at all.
```

### 3.3 Log Volume & Cost — A Genuine Capacity-Planning Concern

```
Directly extending the Database Architecture File 5 capacity-planning discipline to LOG
VOLUME specifically: log ingestion/storage cost scales DIRECTLY with verbosity and
retention — a DEBUG-level-logging-left-on-in-production mistake (an easy one to make,
and easy to forget) can silently multiply log-infrastructure cost by an order of
magnitude. Set log LEVELS deliberately per environment (INFO or WARN as the production
default, DEBUG reserved for active troubleshooting via the Spring Testing & Observability
guide's §6.1 `/actuator/loggers` DYNAMIC level-change capability — changed temporarily,
then reverted, not left on indefinitely) and RETENTION periods matching actual
operational/compliance need (Database Architecture File 6 §5.2's "neither under- nor
over-log" discipline, here applied to retention DURATION specifically).
```

---

## 4. Distributed Tracing Infrastructure

### 4.1 The Backend Choice — Jaeger, Tempo, Zipkin

```
All three are largely interchangeable CONSUMERS of the SAME OpenTelemetry-instrumented
trace data (directly the Spring Cloud guide's §7.1 Micrometer Tracing discussion) — the
choice is mostly about ECOSYSTEM fit: Tempo integrates tightly with the Grafana stack
(querying traces alongside metrics/logs in one UI, directly enabling the §9 worked-stack
"correlate across all three observability pillars in one place" goal); Jaeger has the
longest track record and broadest standalone tooling; Zipkin is the most lightweight,
historically simpler option for smaller-scale needs.
```

### 4.2 Trace Sampling — A Real Cost/Completeness Trade-off

```yaml
# 100% trace capture is rarely sustainable at scale (storage/processing cost scales
# directly with trace volume) — TAIL-BASED sampling captures EVERY trace initially,
# then RETAINS only a sample AFTER seeing the full trace (keeping ALL error traces and
# a sample of successful ones) — giving COMPLETE visibility into failures specifically,
# while controlling overall volume for the (much more numerous) successful requests
processors:
  tail_sampling:
    policies:
      - name: errors-policy
        type: status_code
        status_code: { status_codes: [ERROR] }   # ALWAYS keep error traces — directly
      - name: probabilistic-policy                  # the Database Architecture File 6
        type: probabilistic                           # §5.2 "highest-value, lowest-volume
        probabilistic: { sampling_percentage: 10 }       # audit targets first" principle,
                                                            # applied to trace retention
```

### 4.3 Correlating Traces With Logs and Metrics

```
Directly closing the loop on the Spring Cloud guide's §7.1 correlation-ID discussion: a
WELL-INSTRUMENTED platform propagates the SAME trace ID into structured logs (§3.2's
JSON log format should include a `traceId` field) and exposes it as an exemplar
attached to Prometheus metrics (a specific slow request's trace ID attached directly to
the histogram bucket it fell into) — letting an engineer go from "this metric looks bad"
→ "here's an EXACT slow request's trace" → "here's that SAME request's full structured
logs" in three clicks, rather than three separate, manually-correlated investigations.
```

---

## 5. Alerting Design

### 5.1 Alert on Symptoms, Not Causes — Restated for Infrastructure-Layer Alerting

```yaml
# Directly the Java Testing & Production Readiness guide's §7.5 discipline, expressed as
# an actual Prometheus Alertmanager rule
groups:
  - name: order-service-slo
    rules:
      - alert: ErrorBudgetBurnRateHigh
        expr: |
          (sum(rate(http_requests_total{job="order-service",status=~"5.."}[1h]))
           / sum(rate(http_requests_total{job="order-service"}[1h]))) > 0.01
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "Order service error rate exceeds SLO error budget burn rate"
          # directly an ACTIONABLE alert tied to the §1.2 error-budget concept, NOT a raw
          # "CPU > 90%" infrastructure-metric alert that may or may not reflect actual
          # user-facing impact — the alert fires on the SYMPTOM the business actually cares about
```

### 5.2 Multi-Window, Multi-Burn-Rate Alerting — Reducing Both False Positives and Slow Detection

```
A SINGLE threshold/window alert faces a genuine tension: a SHORT window catches fast
incidents quickly but is prone to noisy false positives from brief blips; a LONG window
filters noise but is slow to detect genuinely fast-developing incidents. The standard SRE
refinement: alert on MULTIPLE burn-rate windows simultaneously (e.g., a FAST 5-minute
window catching severe, rapid degradation AND a SLOWER 1-hour window catching sustained,
moderate degradation) — directly the SAME "balance sensitivity against false-positive
cost" trade-off as the Java JVM Internals guide's §4.4 G1GC soft-pause-time-target
discussion, here applied to alerting THRESHOLDS rather than GC pause behavior.
```

### 5.3 Alert Routing & Severity Tiers

```yaml
route:
  routes:
    - match: { severity: page }       # genuinely urgent, user-facing-impact — pages
      receiver: pagerduty-oncall          # the on-call engineer IMMEDIATELY (§6)
    - match: { severity: warning }       # worth knowing about, NOT urgent — routes to
      receiver: slack-alerts-channel        # a Slack channel for awareness, reviewed
                                               # during business hours, NOT a 3am page
```

---

## 6. Incident Management & On-Call

### 6.1 Severity Classification — A Shared Vocabulary

```
SEV1 — full or near-full outage, significant customer impact, ALL-HANDS response, customer
  communication required.
SEV2 — significant degradation, a SUBSET of functionality/customers affected, urgent but
  not all-hands.
SEV3 — minor issue, workaround available, can be addressed during business hours.

A SHARED, EXPLICIT severity taxonomy (rather than each engineer informally judging
"how bad is this") is what lets an on-call ROTATION function consistently across
DIFFERENT engineers' individual judgment, directly the SAME "shared vocabulary enables
consistent response" principle as the Database Architecture File 6 §8's compliance-
framework discussion, here applied to incident severity rather than regulatory classification.
```

### 6.2 The Incident Commander Role

```
For a SEV1/SEV2, an explicitly-designated INCIDENT COMMANDER (not necessarily the most
technically-senior person in the room) coordinates the RESPONSE — delegating
investigation/mitigation tasks, managing communication, and EXPLICITLY tracking the
incident's timeline — directly the SAME "centralized orchestration beats ad hoc
coordination" principle as the Multi-Agent guide's §2.2 Supervisor pattern's stated
advantage over a fully peer-to-peer agent mesh, here applied to HUMAN incident response:
a clear coordination point makes a chaotic, multi-person incident response dramatically
more tractable than everyone independently investigating and occasionally colliding.
```

### 6.3 Sustainable On-Call — A People-System Concern, Not Just a Technical One

```
Directly connecting back to this series' people-aware framing (the AI/Agentic guide's
File 5 §8.2 alert-fatigue discussion, generalized): an on-call rotation with TOO FEW
people, or an alerting configuration that pages TOO FREQUENTLY for non-actionable noise
(§5.1's discipline, violated), produces genuine on-call BURNOUT over time — a real,
human cost that should be tracked explicitly (pages-per-week per on-call engineer,
after-hours-page frequency) and treated as a metric worth improving, not an inevitable
cost of running production systems.
```

---

## 7. Postmortems & Blameless Culture

### 7.1 The Blameless Principle, Precisely

```
A postmortem investigates WHY a system (including its surrounding processes) allowed an
incident to occur — NOT who personally made a mistake. Directly the Java Core Language
guide's §3.3 "name the principle, not detection mechanics" framing applied to incident
analysis: the goal is identifying the SYSTEMIC gap (a missing test, an unclear runbook, an
alert that should have existed but didn't) that ALLOWED an individual, entirely human,
inevitable mistake to become a production incident — because individual mistakes are
GUARANTEED to keep happening; the SYSTEM's resilience to them is the actual, durable lever.
```

### 7.2 The Postmortem Document Structure

```
1. TIMELINE — precisely what happened, when, established from logs/traces/alerts (§2-4) —
   not from memory or assumption.
2. IMPACT — quantified (how many users, how much revenue, how long) — directly informing
   the §1.2 error-budget accounting.
3. ROOT CAUSE(S) — often MULTIPLE contributing factors, not a single simplistic cause —
   directly the Java JVM Internals guide's §10.4 "name the specific reference chain"
   discipline applied at the INCIDENT level.
4. ACTION ITEMS — concrete, OWNED, TRACKED follow-ups (a new alert, a runbook update, a
   code fix) — directly the "every incident becomes a permanent addition to the baseline"
   discipline this entire reference series has repeated throughout EVERY file's closing
   sections; a postmortem with no tracked action items is a wasted exercise.
```

### 7.3 Postmortem Review as Its Own Process Discipline

```
Postmortems should be REVIEWED (not just written) by people beyond the immediate incident
responders — directly the SAME "review by someone other than the author" discipline as
the SQL Top-50 guide's §46 schema-change-review practice, applied to incident analysis:
a second perspective often catches a systemic gap the people closest to the incident,
having lived through the stress of it, are too close to see clearly themselves.
```

---

## 8. Chaos Engineering

### 8.1 The Core Idea — Proactively Verify Resilience Claims

```
Every resilience pattern from the Java System Design guide's §7 (circuit breakers,
retries, bulkheads, timeouts) is a CLAIM about how the system behaves under failure —
chaos engineering TESTS that claim by DELIBERATELY INJECTING failure (killing a pod,
adding artificial network latency, simulating a downstream dependency's outage) in a
CONTROLLED way, BEFORE a real, uncontrolled failure reveals whether the claim was actually
true. Directly the SAME "verify, don't assume" discipline as the Database Architecture
File 6 §9.1 "test restoring from backup" principle, here applied to RESILIENCE PATTERNS
broadly rather than just backup/restore specifically.
```

### 8.2 A Concrete Chaos Experiment

```yaml
# Using a chaos-engineering tool (Chaos Mesh, Litmus) on Kubernetes — directly testing
# the File 2 §7.2 PodDisruptionBudget's actual effectiveness under a SIMULATED node failure,
# rather than trusting its configuration was correct without ever having exercised it
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata: { name: order-service-pod-kill-experiment }
spec:
  action: pod-kill
  mode: one
  selector: { labelSelectors: { app: order-service } }
  scheduler: { cron: "@every 168h" } # run weekly, in a controlled, monitored window —
                                        # NOT a one-time exercise, exactly the "ongoing
                                        # monitoring, not a one-time validation" principle
                                        # from the AI/Agentic guide's File 6 §2.1, here
                                        # applied to RESILIENCE VERIFICATION specifically
```

### 8.3 Starting Small, Building Confidence Incrementally

```
Directly the SAME canary/gradual-rollout philosophy from the Java Testing & Production
Readiness guide's §6.2, applied to chaos engineering ADOPTION itself: begin chaos
experiments in STAGING, then a LOW-TRAFFIC production service, with experiments run
DURING business hours when the full engineering team is available to observe and respond —
never start chaos engineering by injecting failure into a critical, high-traffic
production system overnight with nobody watching, which converts a controlled LEARNING
exercise into an actual, uncontrolled incident.
```

---

## 9. A Worked Observability Stack

Synthesizing every section into one coherent, production-grade stack for a Kubernetes-based Java platform:

```
METRICS: Spring Boot Actuator (Spring Testing & Observability guide §6-7) → Prometheus
  (§2.1, auto-discovering pods via Kubernetes service discovery) → Grafana dashboards
  (§2.3, version-controlled as code) → Alertmanager (§5) routing to PagerDuty (§6) for
  page-severity alerts and Slack for warning-severity ones

LOGS: Structured JSON application logs (§3.2) → Fluent Bit DaemonSet → Loki → queried
  via the SAME Grafana instance as metrics (§9's cross-pillar correlation goal)

TRACES: OpenTelemetry instrumentation (Spring Cloud guide §7.1) → tail-based sampling
  (§4.2, retaining all errors + a sample of successes) → Grafana Tempo → correlated with
  logs and metrics via shared trace IDs (§4.3)

SLOs: Defined per-service (§1.1), with error-budget burn-rate alerts (§5.2) as the
  PRIMARY paging mechanism — supplemented by, but not REPLACED by, lower-level
  infrastructure alerts (CPU/memory thresholds) for genuinely infrastructure-specific failure modes

INCIDENT RESPONSE: Severity-classified (§6.1), Incident-Commander-coordinated (§6.2) for
  SEV1/SEV2, with EVERY incident producing a reviewed postmortem (§7) whose action items
  feed back into: new Grafana dashboards, new Alertmanager rules, new chaos experiments
  (§8) specifically targeting the failure mode that caused the incident, and updated
  Infrastructure-as-Code (File 5) where the root cause was a misconfiguration

This closes the loop the ENTIRE Cloud-Native & DevOps series has built toward: observability
isn't a separate concern bolted on at the end — it's the FEEDBACK MECHANISM that makes
every other practice in Files 1-5 (containerization, Kubernetes configuration, CI/CD,
cloud architecture, infrastructure code) VERIFIABLE and CONTINUOUSLY IMPROVABLE, rather
than a set of one-time decisions never revisited against real production evidence.
```
