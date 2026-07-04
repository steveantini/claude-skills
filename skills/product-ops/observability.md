# System Observability

| Field | Value |
|---|---|
| **Version** | 1.1.0 |
| **Last Updated** | 2026-06-04 |
| **Applicability** | Backend services, APIs, distributed systems, serverless functions |
| **Dependencies** | None (standalone reference) |

---

## Three Pillars

| Pillar | Purpose | Tools |
|---|---|---|
| **Logs** | Discrete events, debugging context | Structured JSON logging, log aggregation |
| **Traces** | Request flow across services | OpenTelemetry, Jaeger, Tempo |
| **Metrics** | Aggregated measurements over time | Prometheus, Grafana, CloudWatch |

All three must be correlated via shared identifiers (trace ID, request ID).

---

## Structured Logging

### Format: JSON

Every log entry must be machine-parseable JSON. Never use unstructured text in production.

```json
{
  "timestamp": "2026-03-06T12:00:00.123Z",
  "level": "error",
  "message": "Payment processing failed",
  "service": "billing-service",
  "version": "2.4.1",
  "trace_id": "abc123def456",
  "span_id": "span789",
  "request_id": "req-uuid-here",
  "user_id_hash": "sha256:a1b2c3",
  "error": {
    "type": "StripeAPIError",
    "message": "Card declined",
    "code": "card_declined",
    "stack": "..."
  },
  "context": {
    "plan": "pro",
    "amount_cents": 2900,
    "retry_count": 2
  }
}
```

### Log Levels

| Level | When to Use | Example |
|---|---|---|
| **fatal** | Process cannot continue, requires immediate intervention | Database connection pool exhausted, out of memory |
| **error** | Operation failed, requires investigation | Payment failed, external API 500, unhandled exception |
| **warn** | Degraded behavior, not yet broken | Rate limit approaching, retry succeeded after failure, deprecated API called |
| **info** | Normal business operations worth recording | Request completed, user signed up, job finished |
| **debug** | Development/troubleshooting detail | SQL query executed, cache hit/miss, intermediate computation |

### Rules

- **Never log PII** (emails, names, tokens, passwords, full IPs)
- **Always log**: request ID, trace ID, service name, duration, status code
- **Log at boundaries**: incoming requests, outgoing calls, queue consumption, job start/end
- **Include timing**: log duration of external calls, DB queries, processing steps
- **Use consistent field names** across all services

### Correlation IDs

```
Request arrives at API gateway
  → Generate request_id (UUID v4)
  → Generate or propagate trace_id (from W3C Trace Context header)
  → Pass both via headers to all downstream services
  → Every log line includes both IDs
  → Every span includes both IDs
```

Header propagation:

```
traceparent: 00-{trace_id}-{span_id}-01
x-request-id: {request_id}
```

---

## Distributed Tracing

### OpenTelemetry Setup

OpenTelemetry (OTel) is the standard. Use it for instrumentation; export to your backend of choice.

```
Instrumentation (OTel SDK)
  → Exporter (OTLP)
    → Collector (OTel Collector)
      → Backend (Jaeger, Tempo, Honeycomb, Datadog)
```

### What to Trace

| Operation | Span Name | Key Attributes |
|---|---|---|
| HTTP request received | `HTTP {method} {route}` | `http.status_code`, `http.url`, duration |
| Database query | `DB {operation} {table}` | `db.system`, `db.statement` (parameterized), row count |
| External API call | `HTTP {method} {service}` | `http.status_code`, `peer.service`, duration |
| Queue publish | `{queue} send` | `messaging.system`, `messaging.destination` |
| Queue consume | `{queue} process` | `messaging.system`, processing duration |
| Cache operation | `Cache {get\|set\|del}` | `cache.hit`, `cache.key_pattern` |
| AI model call | `LLM {provider} {model}` | `llm.model`, `llm.token_count`, duration, cost |

### Sampling Strategy

- **100% tracing** in development and staging
- **Production**: head-based sampling at 10-20% for normal traffic, 100% for errors
- **Always trace**: errors, slow requests (>p95), requests from internal tools
- **Tail-based sampling** (at collector) if volume is high: keep all error/slow traces, sample normal ones

---

## Error Tracking

### Sentry (or Equivalent)

Sentry is the standard for application error tracking. Alternatives: Bugsnag, Rollbar, GlitchTip (self-hosted).

### Configuration

```typescript
// Sentry initialization
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  tracesSampleRate: 0.2,           // 20% of transactions
  profilesSampleRate: 0.1,          // 10% of sampled transactions
  beforeSend(event) {
    // Strip PII from error events
    return stripPII(event);
  },
});
```

### Error Categorization

| Category | Action | Example |
|---|---|---|
| **P0 — Service Down** | Page immediately, fix now | Database unreachable, auth service 500 |
| **P1 — Feature Broken** | Fix within hours | Payment processing failing, file uploads error |
| **P2 — Degraded** | Fix within 1-2 days | Slow queries, intermittent timeouts |
| **P3 — Minor** | Fix in next sprint | UI glitch, non-critical validation edge case |

### Error Handling Rules

- Catch at boundaries, not everywhere (avoid swallowing errors)
- Include context: what was the user trying to do, what inputs were involved (sans PII)
- Group errors by root cause, not by stack trace variation
- Set up alerts for new error types and error rate spikes
- Review error dashboard weekly; target zero unresolved P0/P1

---

## Performance Monitoring

### Percentile Targets

| Metric | p50 (Median) | p95 | p99 | Notes |
|---|---|---|---|---|
| **API response time** | <100ms | <500ms | <1s | Measure at server, not client |
| **Page load (LCP)** | <1.5s | <2.5s | <4s | Largest Contentful Paint |
| **Database query** | <10ms | <50ms | <200ms | Parameterized query time |
| **External API call** | <200ms | <1s | <3s | Include timeout/retry |
| **AI inference** | Varies | Track trend | Track trend | Model-specific baselines |

### Why Percentiles, Not Averages

Averages hide tail latency. A p50 of 50ms and p99 of 10s looks like an average of 150ms, which tells you nothing useful. Always report p50, p95, p99.

### Key Metrics to Track

```
RED Method (for services):
  Rate     — requests per second
  Errors   — error count and error rate (%)
  Duration — response time distribution (p50/p95/p99)

USE Method (for infrastructure):
  Utilization — % of resource capacity in use
  Saturation  — queue depth, backpressure signals
  Errors      — hardware/system errors
```

### Apdex Score

```
Apdex = (Satisfied + Tolerating/2) / Total

Satisfied:  response < T        (e.g., T = 200ms)
Tolerating: response < 4T       (e.g., < 800ms)
Frustrated: response >= 4T      (e.g., >= 800ms)

Target: Apdex >= 0.95
```

---

## Uptime Monitoring

### External Checks

Run checks from multiple geographic regions, outside your infrastructure.

| Check Type | Frequency | What It Validates |
|---|---|---|
| **HTTP ping** | 30-60s | Endpoint responds with 2xx |
| **API health check** | 60s | `/health` returns OK with dependency status |
| **SSL certificate** | Daily | Certificate not expiring within 30 days |
| **DNS resolution** | 5 min | Domain resolves correctly |
| **Full transaction** | 5 min | Critical user flow completes (synthetic monitoring) |

### Health Check Endpoint Design

```json
GET /health
{
  "status": "healthy",
  "version": "2.4.1",
  "uptime_seconds": 86400,
  "checks": {
    "database": { "status": "healthy", "latency_ms": 3 },
    "redis": { "status": "healthy", "latency_ms": 1 },
    "external_api": { "status": "degraded", "latency_ms": 450 }
  }
}
```

- Return `200` if core functionality works (even if non-critical dependencies are degraded)
- Return `503` only if the service truly cannot serve requests
- Include dependency status for debugging, but don't let optional dependency failures take down the health check

### Uptime Targets

| Tier | Uptime | Downtime/Month | Typical For |
|---|---|---|---|
| 99.0% | 7h 18m | Internal tools |
| 99.9% | 43m 50s | Standard SaaS |
| 99.95% | 21m 55s | Business-critical SaaS |
| 99.99% | 4m 23s | Infrastructure/platform |

---

## Alerting

### Thresholds

| Alert | Condition | Severity | Channel |
|---|---|---|---|
| Error rate spike | >5% of requests for 5 min | P0 | PagerDuty / phone |
| p99 latency | >2s for 10 min | P1 | Slack + PagerDuty |
| Uptime check failure | 3 consecutive failures | P0 | PagerDuty / phone |
| Disk usage | >80% | P2 | Slack |
| Memory usage | >90% for 10 min | P1 | Slack + PagerDuty |
| Certificate expiry | <14 days | P2 | Slack + email |
| Error budget burn | >50% consumed in first half of window | P1 | Slack |
| Queue depth | >1000 messages for 15 min | P2 | Slack |

### Alert Design Rules

- **Every alert must have a runbook link** — what to check, how to mitigate
- **Avoid alert fatigue** — if an alert fires and no one needs to act, delete it or downgrade it
- **Use multi-window burn rates** for SLO-based alerting rather than static thresholds
- **Group related alerts** to avoid notification storms
- **Escalation policy**: Slack (5 min) -> PagerDuty (15 min) -> Phone (30 min)
- **Suppress during maintenance windows**

---

## Log Retention and Search

### Retention Policy

| Log Type | Hot Storage (searchable) | Warm Storage (slower query) | Cold/Archive |
|---|---|---|---|
| Application logs | 7-14 days | 30-90 days | 1 year |
| Access logs | 7 days | 30 days | 90 days |
| Audit logs | 90 days | 1 year | 7 years (compliance) |
| Error/crash logs | 30 days | 90 days | 1 year |
| Debug logs | 3-7 days | -- | -- |

### Search Infrastructure

| Tool | Strengths | Best For |
|---|---|---|
| **Loki + Grafana** | Low cost, label-based indexing, integrates with Grafana | Teams already on Grafana stack |
| **Elasticsearch/OpenSearch** | Full-text search, flexible queries | Large-scale, complex query needs |
| **CloudWatch Logs Insights** | Zero infra, pay-per-query | AWS-native stacks |
| **Axiom** | Serverless, fast ingest, generous free tier | Small-to-mid teams wanting managed solution |

### Log Search Patterns

Essential queries every team should have saved:

```
# All errors for a specific request
trace_id = "abc123" AND level = "error"

# Slow requests in the last hour
duration_ms > 1000 AND timestamp > now() - 1h

# Error rate by service (last 24h)
level = "error" | group by service | count / total

# New error types (appeared in last 24h but not before)
level = "error" AND error.type NOT IN (known_errors)
```

---

## Audit Surfaces: Design the View-Model for Redaction

When an internal (operator/admin) audit surface will later have a user-facing
counterpart, define ONE shared view-model type for the audit data and design
it for redaction from day one:

- Mark every internal-only field (raw prompts, raw model responses, raw
  failure payloads, internal error detail) as OPTIONAL in the type, with an
  explicit "internal-only" comment. A redacted instance is then the SAME type
  with those fields absent, so the user-facing view reuses the same renderers
  with no structural change.
- Encode the redaction rule in one function (strip exactly the internal-only
  fields), not scattered per-surface logic. Never add a REQUIRED
  internal-only field; it breaks the redacted instance.
- If the API is cross-language (e.g. Python backend, TypeScript frontend),
  mirror the type on both sides and document that they must stay in sync.

Presentation rule for trust-relevant signals: distinguish "the safety net
worked" from "the system failed". A guardrail rejecting bad output (e.g. a
grounding filter dropping a fabricated citation) is an informational signal
and should render neutrally; reserve error styling for genuine failures.
Showing rejected items honestly, with the reason, IS the trust story.

---

## Implementation Checklist

- [ ] Structured JSON logging in all services
- [ ] Correlation IDs (request ID, trace ID) propagated across service boundaries
- [ ] OpenTelemetry instrumentation on HTTP, DB, cache, and external call boundaries
- [ ] Sentry (or equivalent) configured with PII scrubbing
- [ ] Health check endpoints on all services
- [ ] External uptime monitoring from multiple regions
- [ ] RED metrics dashboards for all services
- [ ] p50/p95/p99 latency tracked and baselined
- [ ] Alerting configured with runbooks and escalation policies
- [ ] Log retention policy documented and enforced
- [ ] Weekly observability review (error trends, latency trends, alert noise audit)
