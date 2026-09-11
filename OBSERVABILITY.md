# Observability Plan — Identity Server & Rails Application

Working reference for on-call. Goal: replace "I can't tell if something is wrong until a customer calls" with alarms that fire before that phone rings, on signals specific enough to point at a cause, not just "something's off."

## SLOs and SLIs

**Identity Server** — availability and latency, since a slow or unavailable auth service takes every downstream product down with it.

| SLO | SLI | Target | Measured via |
|---|---|---|---|
| **Availability** | Proportion of requests that don't return a 5xx | 99.9% over a rolling 30-day window (≈43 min of budget/month) | ALB `RequestCount` vs. `HTTPCode_Target_5XX_Count` |
| **Latency** | p95 response time on token-issuance/validation endpoints | p95 < 300ms, p99 < 800ms, over rolling 5-minute windows | ALB `TargetResponseTime` |

**If the availability SLO is breached** (error budget exhausted for the month): non-essential deploys to the Identity Server pause until a reliability review brings it back into compliance, and the incident gets a post-mortem. This isn't bureaucracy for its own sake — this is the one service where "move fast" directly trades off against every other product's ability to authenticate.

**If the latency SLO is breached:** if it correlates with a specific deploy, D2's rollback already handles it automatically. If it doesn't correlate with a deploy, it opens a P2 investigation — see Distributed Tracing below for how to find the cause instead of just the symptom.

**Rails application** — looser than the Identity Server's, since a slow page load doesn't take every other product down with it the way a slow auth call does.

| SLO | SLI | Target | Measured via |
|---|---|---|---|
| **Availability** | Proportion of requests that don't return a 5xx | 99.5% over a rolling 30-day window | ALB `RequestCount` vs. `HTTPCode_Target_5XX_Count` |
| **Latency** | p95 response time | p95 < 500ms over rolling 5-minute windows | ALB `TargetResponseTime` |

**If either Rails SLO is breached:** same deploy-correlation check as the Identity Server — D2's circuit breaker and the workflow's own post-deploy verification (added after this document's peer review; see the decision log) catch a bad deploy automatically. An uncorrelated breach opens a P3 investigation, not a P2 — Rails' blast radius is one product, not every product's ability to authenticate.

## Metrics and Alarms

**Health-check story, stated once so all four deliverables agree on it:** the Identity Server uses tagged `/health/live` vs. `/health/ready` checks (ADR.md) — readiness includes its DB and directory dependency, and the ALB routes on readiness. Rails is deliberately simpler: the ALB target group checks Rails' built-in `/up` route, which is a **liveness**-level check only (did the process boot), not a dependency check — Rails doesn't ship a readiness variant the way the custom ASP.NET Core health checks do, and building one is out of scope here since it means adding a custom controller to an app whose source isn't part of this exercise. This is a known, accepted simplification: a wedged Rails process gets pulled from rotation, but a Rails process that's up with a dead DB connection currently would not be, until that shows up as request-level errors instead. The Dockerfile's `HEALTHCHECK` (also `/up`) has no effect on ECS Fargate unless the same check is separately declared in the task definition's `healthCheck` field — that's the block to add if a wedged container PID (not just a bad HTTP response) needs to trigger an ECS-level restart:
```json
"healthCheck": {
  "command": ["CMD-SHELL", "ruby -e 'require \"net/http\"; exit(Net::HTTP.get_response(URI(\"http://localhost:3000/up\")).code == \"200\" ? 0 : 1)'"],
  "interval": 15, "timeout": 3, "retries": 2, "startPeriod": 30
}
```

**ECS task health** (both services)

| Metric | Threshold | What it means |
|---|---|---|
| `RunningTaskCount` vs. `DesiredCount` | Mismatch sustained > 5 min | Tasks failing to start or stay healthy — check the service's health-check story above and the task's recent stop reasons |
| `ECSServiceAverageCPUUtilization` | > 85% sustained 10 min | Approaching the autoscaling ceiling despite scaling (max 6 tasks for Identity per D9) — capacity plan needs revisiting, not just another alarm |
| ALB `UnHealthyHostCount` | > 0 sustained 3 min | A task is registered but failing its health check |
| ECS deployment circuit breaker fired (EventBridge `DEPLOYMENT_FAILED`), per service | Any occurrence | A bad deploy just got auto-rolled-back per D2 — pages immediately, because a bad revision was loose in production even briefly. For Rails specifically, this is the same signal the pipeline's own post-deploy verification step checks for directly (see the decision log) — belt and suspenders, not two different stories |

**RDS performance — Identity Server (SQL Server)**

| Metric | Threshold | What it means |
|---|---|---|
| `CPUUtilization` | Warning > 70% sustained, critical > 80% for 15 min | Sustained headroom loss, not a momentary spike |
| `DatabaseConnections` | > 90% of the instance class's max connections | Imminent connection exhaustion — for an auth database, that's an outage, not a warning |
| `FreeStorageSpace` | Warning < 20% of provisioned, critical < 10% | Prevents the hard failure mode of an RDS instance running out of disk |
| `ReadLatency` / `WriteLatency` | > 20ms sustained | I/O-bound query pattern worth investigating before it shows up as app-level latency |

**RDS performance — Rails (PostgreSQL)**

| Metric | Threshold | What it means |
|---|---|---|
| `CPUUtilization` | Warning > 70%, critical > 80% for 15 min | Same reasoning as the Identity Server's instance — headroom, not a spike |
| `DatabaseConnections` | > 90% of `max_connections` | Same failure mode as Identity: connection exhaustion presents as app-wide errors, not a slow query |
| `FreeStorageSpace` | Warning < 20%, critical < 10% | Same hard-failure prevention |
| `ReplicaLag` (if a read replica exists) | > 30s | Only relevant once/if Rails adds a read replica — included so it isn't forgotten if that happens |

**Application-level — Identity Server**

| Metric | Threshold | What it means |
|---|---|---|
| ALB `HTTPCode_Target_5XX_Count` | > 1% of requests over 5 min | Directly burns the availability SLO's error budget |
| ALB `TargetResponseTime` (p95 / p99) | > 300ms / > 800ms over 5 min | Directly burns the latency SLO |
| Metric filter on structured app logs for `TokenIssuanceFailed` / `AuthenticationException` | Rate exceeds baseline | Catches auth-specific failures that don't necessarily show up as HTTP 5xx (e.g., a 200 with a failure payload) |

**Application-level — Rails**

| Metric | Threshold | What it means |
|---|---|---|
| ALB `HTTPCode_Target_5XX_Count` | > 1% of requests over 5 min | Directly burns the Rails availability SLO |
| ALB `TargetResponseTime` (p95) | > 500ms over 5 min | Directly burns the Rails latency SLO |
| Puma request-queue time (via a `X-Request-Start`/queue-time header, or Puma's own stats socket exported as a custom CloudWatch metric) | > 100ms sustained | Requests waiting on a thread before Puma even starts working them — a capacity signal distinct from response time, and the one that shows up first when a service is about to fall over from load rather than a slow dependency |

## Distributed Tracing

AWS's current guidance for tracing on ECS Fargate has moved past the classic standalone X-Ray daemon to the **AWS Distro for OpenTelemetry (ADOT) Collector** running as a sidecar container in the same task definition — the .NET app is instrumented with the OpenTelemetry .NET SDK, exports over OTLP to the sidecar on `localhost`, and the ADOT collector forwards to X-Ray. Same destination (X-Ray traces, viewable in the Service Map), current mechanism instead of the superseded one.

**Diagnosing a latency spike from traces:** the Service Map view is the starting point — it shows whether elevated latency is concentrated at the Identity Server node itself (CPU-bound, likely token signing under load — cross-reference the ECS CPU metric above), at the RDS node (DB-bound — cross-reference `DatabaseConnections` and `ReadLatency`/`WriteLatency`), or at an external node (the Graph API or AADDS LDAP call from D1 — a throttling response from either would show up here as a slow external segment, not a slow local one). Pulling a handful of the slowest individual traces from the spike window and reading their subsegment breakdown (DB query time vs. signing/compute time vs. external call time) turns "it got slower" into "it got slower because X" — which is the entire point of tracing over metrics alone.

This exercise scopes X-Ray to the Identity Server specifically, but the same ADOT sidecar + OTLP pattern applies to Rails without a different mechanism — worth noting so the two services don't end up on divergent tracing setups later by default rather than by decision.

## Log Strategy

- **Log groups:** one per service (`/ecs/strongmind-production/identity-server`, `/ecs/strongmind-production/rails-app`), matching the existing ECS cluster naming.
- **Retention:** 30 days for general application logs — enough for operational debugging without paying to keep noise indefinitely. Authentication-relevant logs (login attempts, token issuance/revocation) are flagged as a probable exception: StrongMind serves K-12 education customers, and student-data-adjacent audit trails commonly need longer retention for compliance. This is called out as an assumption to confirm with StrongMind's actual data retention policy, not decided unilaterally here.
- **Useful CloudWatch Insights query** — slowest requests in a window, ranked, to jump straight to what to trace:

```
fields @timestamp, path, statusCode, durationMs
| filter durationMs > 1000
| sort durationMs desc
| limit 20
```

## Alerting Pipeline

CloudWatch Alarms → an SNS topic → an HTTPS subscription to Jira Service Management's CloudWatch integration endpoint (Atlassian's documented pattern: JSM issues an endpoint URL, subscribed to the SNS topic as an HTTPS target) → Jira Ops routes to the on-call engineer by schedule, with dedup/escalation handled on the JSM side rather than reinvented in CloudWatch. This is the same pipeline D4 and D9's failure signals both drain into — one alerting path, not a separate one per part of this migration.

`rails-deploy.yml`'s own `if: failure()` step publishes directly to this same SNS topic on a failed/rolled-back deploy, rather than relying solely on the resulting alarms to notice. That closes a real gap the alarm-only version had: the pipeline requirement is "roll back and send a failure notification," and a notification that only exists as an inference from a separate document isn't the same thing as a property of the pipeline itself. Publishing into the existing topic — instead of a new Slack webhook or a second pager — keeps it the one alerting path this section describes, not a second one bolted onto Part 2.

| Severity | Examples | Behavior |
|---|---|---|
| **P1 — page immediately** | Zero healthy ECS tasks; RDS connections exhausted; ALB 5xx rate > 5% | Wakes on-call regardless of time of day |
| **P2 — page during business hours, notify + 15-min escalation off-hours** | Single-AZ degradation; latency SLO breach; deployment circuit breaker rollback fired (D2) | Urgent, but the system is still serving traffic |
| **P3 — notify only, no page** | Storage approaching threshold; a single alarm flapping during self-recovery | Needs attention, not a 2 AM wakeup |
