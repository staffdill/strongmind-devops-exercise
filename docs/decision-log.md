# Migration Decision Log

A running, source-cited record of the calls made while building the Identity Server ADR, the Rails CI/CD pipeline, the Dockerfile, and the observability plan — kept so every decision can be traced back to evidence.

This is the plain-text companion to [`decision-log.html`](./decision-log.html), which has the same content with styling and inline navigation. Started 2026-09-11, scope Parts 1–4.

**How to read this:** each entry is one question that came up while drafting the deliverables, the findings that settled it (with links to primary sources), the decision, and the reasoning — the same trail a reviewer could follow to ask "why did you choose this?"

---

## Part 1 — Identity Server Migration (ADR)

### D1 — Stand up a directory service to bridge Azure AD Domain Services?

**Question:** The Identity Server currently calls Azure AD Domain Services (AADDS) for directory lookups. After the move to ECS Fargate, does it make sense to stand up an AWS directory service (Simple AD) to preserve that dependency?

**Findings:**
- AWS Directory Service Simple AD closed to new customers on July 30, 2026 — it can no longer be provisioned at all as of this exercise.
- Even before that cutoff, Simple AD never supported trust relationships with other domains — only AWS Managed Microsoft AD does.
- Azure AD Domain Services doesn't support trust relationships either — it's a one-way sync target *from* Entra ID, not a domain controller you can federate with. So even Managed Microsoft AD would have nothing on the Azure side to trust against.

**Decision:** Do not stand up any AWS directory service to bridge this dependency. Treat the AADDS call as a question of "what does the Identity Server actually need from directory lookups," not "how do we replicate the directory."

**Rationale:** If the lookups are logical (user/group existence, membership) rather than raw LDAP/Kerberos binds, the clean post-migration path is calling the Microsoft Graph API against Entra ID directly — a REST/OAuth call from anywhere, with no cross-cloud AD infrastructure to run or pay for. If the code is genuinely hard-wired to LDAP/Kerberos and rewriting it is out of scope here, the honest move is to keep the interim cross-cloud call to AADDS and document it as tech debt in the ADR's Risk Register — not stand up parallel AD infrastructure that solves an infrastructure problem the app doesn't actually have.

**Sources:**
- [AWS Directory Service — Simple AD overview](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_simple_ad.html)
- [AWS Directory Service — Simple AD best practices (trust limitation)](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/simple_ad_best_practices.html)
- [AWS Directory Service — AWS Managed Microsoft AD (trust support)](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html)
- [Microsoft Learn — Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)

### D3 — Traffic cutover: Route 53 weighted routing vs. ALB blue/green

**Question:** How do we move traffic from the Azure App Service to the new ECS Fargate service with zero downtime — weighted DNS, or a full blue/green cutover via an ALB/CodeDeploy?

**Findings:**
- Route 53 supports weighted routing combined with per-record health checks in an "active-active" pattern: every healthy record is a live, eligible answer, and traffic share shifts by adjusting weights.
- If the AWS-side health check goes unhealthy, Route 53 stops answering with it — traffic effectively reverts to the Azure origin automatically, with no custom failover logic to write.
- This works at the DNS layer, so it's agnostic to which cloud either endpoint lives in — unlike ALB-native blue/green, which only operates inside one load balancer in one account.

**Decision:** Cut over with Route 53 weighted routing across health-checked records for the Azure App Service and the new ALB in front of ECS, shifting weight in stages (e.g. 10% → 50% → 100%) rather than a single flip.

**Rationale:** Consistent with D2: this is a one-time cross-cloud cutover, so it doesn't justify standing up CodeDeploy just for this migration. DNS weighting is the natural mechanism when the two endpoints being compared live in different clouds, and the health-check-driven reversion gives the ADR a concrete, low-effort rollback trigger and procedure instead of a manual one.

**Sources:**
- [AWS re:Post — Use Route 53 health checks for DNS failover](https://repost.aws/knowledge-center/route-53-dns-health-checks)
- [AWS re:Post — Route 53 automatic failback behavior](https://repost.aws/knowledge-center/route-53-prevent-automatic-failback)

### D8 — DB migration: AWS DMS has no CDC path from Azure SQL Database

**Question:** What tooling migrates Azure SQL Database to RDS for SQL Server, and can it run as continuous replication ahead of a near-zero-downtime cutover?

**Findings:**
- AWS's own DMS documentation states plainly: "AWS DMS doesn't support change data capture operations (CDC) with Azure SQL Database." Azure SQL Database can only be a DMS full-load source, not an ongoing-replication source.
- Azure SQL Database (PaaS, not Managed Instance) doesn't expose native `BACKUP DATABASE` — backups are fully automatic and system-managed, so the RDS "native backup/restore via S3" path isn't available either.
- Microsoft's own supported export mechanism for Azure SQL Database is the .bacpac format (schema + data), best-practiced for databases under 200GB — and an identity/auth database (users, clients, grants/tokens) is almost certainly well inside that.
- This has a knock-on effect on D3: since neither DB can be authoritative for both origins at once without continuous sync, the weighted Route 53 ramp can't be a multi-day gradual shift — it has to be compressed into the same short maintenance window as the DB cutover.

**Decision:** Pre-stage with rehearsal bacpac export/import runs; cut over in a short, overnight (Mountain Time) maintenance window: write-freeze → final bacpac export/import → row-count/checksum validation → point ECS at RDS → compressed Route 53 ramp (10% → 50% → 100% in minutes, not days) → 24–48h bake with Azure kept warm at 0% weight as a rollback option.

**Rationale:** This is the same theme as D1 and D5: the tool that looks like the obvious default (DMS with CDC, the way it works for most other DMS sources) doesn't actually apply to this specific source engine, and finding that out before writing the ADR avoids promising a live cutover the tooling can't deliver. Being upfront that this needs a real (short, scheduled, off-peak) maintenance window is more defensible than an inaccurate "zero-downtime" claim the on-call engineer would discover the hard way. The one limitation this doesn't remove: any write that lands on AWS during the bake period can't cleanly roll back to Azure without manual reconciliation — that's going in the Risk Register rather than glossed over.

**Sources:**
- [AWS DMS — Using Microsoft Azure SQL Database as a source (CDC not supported)](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.AzureSQL.html)
- [Microsoft Learn — Back up and restore for Azure SQL Database (automatic, no native BACKUP)](https://learn.microsoft.com/en-us/training/modules/backup-restore-databases/4-use-azure-sql-database)
- [NextGenSoft — Migrating Azure SQL Database to RDS SQL Server via .bacpac](https://www.nextgensoft.io/blogs/guide-to-migrating-azure-sql-database-to-aws-rds-using-bacpac/)

### D9 — Pre-scale ahead of the 7–9 AM spike instead of relying on reactive autoscaling alone

**Question:** The 7–9 AM Mountain Time school-start spike (400→1,200 req/min) is a known, recurring pattern. Should capacity be handled purely by CPU-based target tracking, or scaled up ahead of time?

**Findings:**
- Target tracking is reactive: a Fargate task has to pull its image, boot, and pass two consecutive health checks before it can take traffic, so waiting for CPU to cross the 60% threshold at 7:00 AM means the front edge of the spike is served understaffed.
- Application Auto Scaling supports `put-scheduled-action` against an ECS service's `DesiredCount`/`MinCapacity`, run alongside — not instead of — a target-tracking policy; the two compose (target tracking still scales above whatever floor the schedule sets).
- Scheduled actions support a `--timezone` parameter (e.g. `America/Denver`) instead of a fixed UTC offset, specifically so the schedule doesn't drift by an hour across DST transitions.

**Decision:** Add two scheduled actions on top of the existing target-tracking policy: raise `MinCapacity` 3→5 at 6:15 AM (45-minute lead time for task startup), drop it back to 3 at 9:15 AM (15-minute trailing buffer), weekdays only, both declared with `--timezone "America/Denver"`. (Baseline raised from 2→3 tasks during peer review — see the Review Sheet below — for AZ-failure headroom, and the scheduled floor moved with it.)

**Rationale:** This is the standard AWS-recommended pairing for a workload with one predictable daily pattern layered on otherwise-normal variance: schedule the floor for the known pattern, let target tracking handle everything unpredictable above it. The DST-safe timezone parameter is a small detail that's cheap to get right up front and easy to silently get wrong (an hour of under-capacity, twice a year, at the exact worst time) if a fixed UTC offset is hardcoded instead.

**Sources:**
- [AWS — Scheduled Actions of Application Auto Scaling now support Local Time Zone](https://aws.amazon.com/about-aws/whats-new/2021/02/scheduled-actions-application-auto-scaling-support-local-time-zone)
- [AWS re:Post — Configure Amazon ECS Service Auto Scaling on Fargate](https://repost.aws/knowledge-center/ecs-fargate-service-auto-scaling)

---

## Cross-cutting — Deployment Mechanics (Parts 1 & 2)

### D2 — ECS deployment mechanism: native rolling + circuit breaker, not CodeDeploy blue/green

**Question:** StrongMind already has a partially-built GitHub Actions deployment pipeline for Rails. Should the standardized pipeline (Part 2) — and the Identity Server ECS service (Part 1) — deploy via CodeDeploy blue/green, or ECS's native rolling update?

**Findings:**
- ECS rolling deployment is the default strategy and needs no extra infrastructure — no CodeDeploy application, deployment group, or additional IAM roles.
- The ECS deployment circuit breaker (GA since December 2020, with configurable failure thresholds added in 2026) automatically detects a deployment that can't stabilize and rolls back to the last successful task definition — automated rollback without CodeDeploy.
- ECS also shipped 1-click manual rollback for service deployments in 2025, giving a fast manual escape hatch on top of the automatic circuit breaker.

**Decision:** Standardize both the Identity Server and the Rails pipeline on ECS's native rolling deployment with the deployment circuit breaker enabled, driven entirely from GitHub Actions (register task definition → `update-service` → wait for stability). No CodeDeploy.

**Rationale:** Workstream B is explicitly about standardizing CI/CD that "any Rails service in the org can adopt" — introducing CodeDeploy would add a second deployment control plane (application, deployment group, lifecycle hooks) for every team to learn on top of GitHub Actions, for marginal benefit over rolling + circuit breaker at this traffic profile. One mechanism, reused everywhere, is easier to standardize and easier to hand off.

**Sources:**
- [AWS — ECS Deployment Circuit Breaker GA (2020)](https://aws.amazon.com/about-aws/whats-new/2020/12/amazon-ecs-announces-the-general-availability-of-ecs-deployment-circuit-breaker/)
- [AWS — Configurable ECS deployment circuit breaker settings (2026)](https://aws.amazon.com/about-aws/whats-new/2026/07/amazon-ecs-circuit-breaker-settings/)
- [AWS — ECS 1-click rollbacks for service deployments (2025)](https://aws.amazon.com/about-aws/whats-new/2025/05/amazon-ecs-1-click-rollbacks-service-deployments)

---

## Cross-cutting — Failure Notification (Parts 1, 2 & 4)

### D4 — Route deploy/cutover failures through Part 4's alerting pipeline, not a bespoke webhook

**Question:** How should a failed Rails deploy (Part 2) or a failed Identity Server cutover (Part 1, D3) notify someone — a dedicated Slack/webhook step in the GitHub Actions workflow, or the CloudWatch → Jira Ops pipeline planned for Part 4? Could the D3 Route 53 health checks double as that signal?

**Findings:**
- Route 53 health checks publish their pass/fail state natively as a `HealthCheckStatus` CloudWatch metric (1/0) in the `AWS/Route53` namespace — no extra plumbing required to alarm on it.
- That means the D3 cutover health checks can feed a CloudWatch alarm that terminates in the same Jira Ops pipeline Part 4 already needs — one alerting path, not two.
- Route 53 health checks don't apply to the standing Rails pipeline: routine deploys don't compare two different cloud endpoints, so there's nothing meaningful to point a Route 53 health check at that isn't already an ALB target-group health check.
- The ECS deployment circuit breaker (D2) already reads ALB target health directly during a rolling update, and `aws ecs wait services-stable` surfaces that as a GitHub Actions job failure — the tighter, native signal for a same-cloud deploy.

**Decision:** No bespoke Slack/webhook step in `rails-deploy.yml`. Rollback is triggered by the ECS circuit breaker + a failed `wait services-stable`; the resulting service-health alarms (Part 4) are what page the on-call engineer. The Identity Server's Route 53 cutover health check feeds a CloudWatch alarm into that same Part 4 pipeline.

**Rationale:** One alerting pipeline that everything drains into is easier to reason about and doesn't make the CI/CD workflow depend on a chat tool or webhook secret that isn't confirmed to exist at StrongMind. It also means Part 4's design work directly pays for Part 1 and Part 2's "how do we know it failed" requirements instead of duplicating it.

**Sources:**
- [AWS — Monitoring Route 53 health checks using CloudWatch](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/monitoring-health-checks.html)
- [AWS — Route 53 CloudWatch metrics for health checks (launch)](https://aws.amazon.com/about-aws/whats-new/2013/06/26/route-53-announces-cloudwatch-metrics-for-health-checks)

---

## Part 3 — Multi-stage Dockerfile (Rails Application)

### D5 — Base image: start on `ruby:3.3-slim`, revisit Alpine later

**Question:** Should the production Dockerfile start on `ruby:3.3-slim` (Debian/glibc) or `ruby:3.3-alpine` (musl libc), with a plan to move to Alpine once it's "more mature" for this app?

**Findings:**
- Alpine swaps glibc for musl libc. Gems with native C extensions built against glibc assumptions can fail outright — documented real-world breakage includes protobuf/grpc-dependent gems and Oracle Instant Client bindings.
- Common gems (nokogiri, grpc, pg) usually compile fine on Alpine, but less common ones are hit-or-miss — the safe compatibility surface has to be verified against this app's actual Gemfile, not assumed.
- Musl's memory allocator behaves differently from glibc's: Rails apps on Alpine have been observed using roughly 15–20% more RSS memory at steady state — relevant here because Fargate task memory is billed and sized explicitly.
- Size difference is real (roughly 100MB+ smaller base layer on Alpine) but multi-stage builds already claw back most of that gap on either base, since build tooling and gem caches never reach the runtime stage.

**Decision:** Ship `ruby:3.3-slim` for the runtime stage now. Revisit Alpine only after the app's specific gem set has been tested against musl in CI and the memory-footprint tradeoff has been measured under real load — not as a default "smaller is better" swap.

**Rationale:** This is a case where the size win is the easy part to see and the compatibility/memory cost is the part that bites in production. Slim removes an entire class of native-extension surprises during a migration that already has enough moving parts (new cloud, new database engine, new secrets store); Alpine stays on the table as a deliberate, measured optimization once the team has bandwidth to validate it — not a default.

**Sources:**
- [Docker Docs — glibc and musl support](https://docs.docker.com/dhi/core-concepts/glibc-musl/)
- [google-cloud-ruby — Alpine gem incompatibility issue](https://github.com/googleapis/google-cloud-ruby/issues/1351)
- [Docker Hub — official Ruby image (slim vs alpine variants)](https://hub.docker.com/_/ruby)

### D6 — Skip Thruster; run Puma directly behind the ALB

**Question:** Rails 8 apps generate with Thruster (Basecamp's HTTP/2 proxy) in front of Puma by default. Keep it in the container, or run Puma directly?

**Findings:**
- Rails 8's default generated Dockerfile adds the `thruster` gem and runs `CMD ["./bin/thrust", "./bin/rails", "server"]`, exposing port 80.
- Thruster's value is HTTP/2 termination, TLS, compression, and X-Sendfile asset serving in front of Puma — all things an ALB in front of an ECS service already does at the load-balancer layer.

**Decision:** Run Puma directly on port 3000; no Thruster. `CMD ["./bin/rails", "server"]`.

**Rationale:** Behind an ALB, Thruster would be a second proxy hop terminating things the ALB already terminates — one more process and dependency in the image for no functional gain in this topology. Worth revisiting only if this service is ever exposed without a load balancer in front of it.

**Sources:**
- [Saeloun — Rails 8 adds Thruster as the default HTTP/2 proxy](https://blog.saeloun.com/2026/05/09/rails-8-thruster-http2-proxy-server/)

### D7 — HEALTHCHECK: Ruby one-liner against Rails' `/up`, not curl

**Question:** What should the Dockerfile's `HEALTHCHECK` hit, and does it need curl installed to do it?

**Findings:**
- Rails 7.1 added a default health check route at `/up`, served by `Rails::HealthController#show` — 200 if the app booted, 503 otherwise. It's already present on any Rails 7.1+ app.
- Ruby (with `net/http` from the standard library) is already in the image; curl is not, and adding it purely for a health check is one more general-purpose network tool sitting in a container that doesn't otherwise need one.

**Decision:** Use a one-line Ruby `Net::HTTP` request against `/up` for the HEALTHCHECK instead of installing curl.

**Rationale:** This is the "comment where you're making a deliberate tradeoff" the Part 3 requirements ask for: a slightly slower Ruby interpreter start per health check poll, traded for not adding a package to the final image at all. It also checks the same signal ECS/ALB target-group health checks should be pointed at, so the container's own liveness view matches what's deciding traffic routing.

**Sources:**
- [Saeloun — Rails 7.1 introduces default health check controller](https://blog.saeloun.com/2023/02/27/rails-introduces-default-health-check-controller/)

---

## Part 4 — Observability Design Plan

### D11 — X-Ray via ADOT sidecar, and how Jira Ops actually receives a CloudWatch alarm

**Question:** Two mechanics needed to be right, not just referenced: how does a .NET service on ECS Fargate actually get traces into X-Ray, and what's the literal mechanism connecting a CloudWatch alarm to Jira Ops (OpsGenie)?

**Findings:**
- AWS's current guidance for ECS Fargate tracing has moved past the classic standalone X-Ray daemon to the AWS Distro for OpenTelemetry (ADOT) Collector running as a sidecar container in the same task definition, with the app instrumented via the OpenTelemetry SDK rather than the legacy X-Ray SDK directly.
- Jira Service Management's own documented CloudWatch integration is: JSM issues an endpoint URL, an SNS topic gets an HTTPS subscription pointed at that URL, and CloudWatch alarms use that SNS topic as their alarm action. JSM/OpsGenie handles on-call routing and dedup on its side.
- RDS alarm thresholds (CPU, connections, storage) were cross-checked against AWS's own RDS monitoring guidance rather than picked arbitrarily: CPU warning >70%/critical >80% sustained, connections >90% of the instance class max, free storage warning <20%/critical <10%.

**Decision:** Trace via an ADOT collector sidecar exporting to X-Ray, not a bare X-Ray daemon. Alert via CloudWatch Alarm → SNS → HTTPS subscription to the JSM-issued endpoint → Jira Ops on-call routing.

**Rationale:** Same pattern as D6 (Thruster) and D1 (Simple AD): describing the outcome AWS actually wants ("get traces into X-Ray") using the mechanism that's since been superseded reads as stale rather than current. The JSM/SNS mechanism matters because "alerts go to Jira Ops" is meaningless without the actual plumbing — this is what OBSERVABILITY.md needs to be usable as real documentation, not just a diagram label.

**Sources:**
- [AWS Distro for OpenTelemetry — ECS Fargate task definition setup](https://aws-otel.github.io/docs/setup/ecs/task-definition-for-ecs-fargate/)
- [Atlassian — Integrate Jira Service Management with Amazon CloudWatch](https://support.atlassian.com/jira-service-management-cloud/docs/integrate-with-amazon-cloudwatch/)
- [AWS re:Post — CloudWatch alarms for RDS free storage space](https://repost.aws/knowledge-center/storage-full-rds-cloudwatch-alarm)

---

## Review Sheet

Every deliverable went through peer review before being called done. Findings and their resolutions, by area — the final state each file was left in, not a transcript of getting there.

### Cutover & migration design

| Finding | Resolution |
|---|---|
| The write-freeze could end before the Route 53 ramp reached 100% AWS, leaving Azure and RDS both able to accept writes at once. | Freeze now explicitly spans the entire ramp, lifting only at 100% AWS (or reverting to Azure as the writer if the rollback trigger fires mid-ramp). |
| "2 tasks, multi-AZ" doesn't survive a real AZ failure — one AZ down could leave a single task serving all auth traffic. | Baseline raised to 3 tasks across 3 AZs; the scheduled peak floor (D9) moved with it, 3→5. |
| A compressed DNS ramp assumes client/resolver caches have already expired, which a same-night TTL change doesn't achieve. | TTL drops to 60s at least 48 hours ahead of the maintenance window. |
| Neither cutover step said which path the two Route 53 health checks probe — if either hit an endpoint affected by the freeze, the freeze itself could force an unplanned weight shift. | Azure-side check hits `/health/live` (unaffected by the freeze); ALB-side hits `/health/ready` (a real dependency signal — what makes it a meaningful rollback trigger). |
| The AADDS/Entra ID decision (D1) hedged on "where the code allows" without saying what happens where it doesn't. | Week-1 call-site audit, NSG-locked secure LDAP over the internet as the interim path for whatever's genuinely LDAP-bound, one-quarter deadline instead of open-ended debt. |
| RDS for SQL Server's licensing cost, and two cheaper alternatives (RDS Postgres, Azure SQL Managed Instance), went unmentioned. | ADR states the cost tradeoff plainly and names both alternatives as considered and rejected, with reasons. |
| .NET 6 is already past end of support, and .NET 8 — the obvious-sounding fix — reaches its own end of support two months from now. | Risk Register names the EOL explicitly; the recommended target is .NET 10 LTS, not .NET 8. |

### Pipeline correctness (`rails-deploy.yml`)

| Finding | Resolution |
|---|---|
| A workflow-level `cancel-in-progress: true` could cancel an entire in-flight run — including a live deploy — regardless of the deploy job's own non-cancelling concurrency group. | Top-level `cancel-in-progress` is now conditional on ref/event: false for anything that can reach the deploy job. |
| `workflow_dispatch` can target any branch; keying the group on `ref` alone meant a later push to that branch could still cancel an in-flight manual deploy sharing the group. | `workflow_dispatch` runs key their group on `run_id` instead, so they can never share a cancellable group with a plain push. |
| `wait-for-service-stability` exiting 0 doesn't prove the new revision is running — if the ECS circuit breaker silently rolled back, the action still reports success (confirmed as a tracked, still-open upstream issue). | Added a step comparing the live PRIMARY deployment's task definition against the one just registered; a mismatch fails the job and triggers rollback. |
| Widening the rollback trigger to catch that silent-rollback case also made it fire on unrelated failures (a throttled AWS API call, OIDC auth) — forcing a needless production bounce. | Rollback now gated on the deploy or verify step specifically failing; the notification step stays broader, since an unrelated failure should still page even though it shouldn't restart the service. |
| The pipeline never sent its own failure notification, despite that being a literal requirement — relying on CloudWatch alarms alone satisfies it by inference, not in the file being graded. | Added an `if: failure()` step publishing to the same SNS topic the CloudWatch alarms already use. |
| Building `linux/amd64,linux/arm64` on a standard x86_64 GitHub runner needs QEMU registered — without it, this is a documented failure mode, not an edge case. | Added `docker/setup-qemu-action` before Buildx in the push job. |
| Multi-arch was justified only as "the prompt said so." | Named the actual reason: Graviton (arm64) Fargate tasks run roughly 20% cheaper for equivalent vCPU/memory, and switching later is a task-definition field change, not a rebuild. |
| The `:latest` ECR tag is a footgun for anyone deploying by hand from the console. | Commented explicitly: the pipeline always deploys by SHA; `:latest` exists for human convenience only. |
| `describe-task-definition` by family returns the latest *registered* revision, not necessarily the one currently *serving* traffic. | Documented as an explicit assumption rather than left implicit. |

### Container & runtime

| Finding | Resolution |
|---|---|
| Rails' Dockerfile `HEALTHCHECK` has no effect on ECS Fargate — ECS ignores an image's embedded check entirely unless the same check is separately declared in the task definition. | Documented explicitly, with the exact task-definition `healthCheck` block to mirror it. |
| Rails and the Identity Server told inconsistent health-check stories (readiness vs. liveness), with no single place stating either. | One health-check story for both services, stated once in `OBSERVABILITY.md`. |
| Entrypoint migrations plus `minimumHealthyPercent: 100` mean old and new code share the database mid-rollout, and multiple new tasks call `db:prepare` concurrently — neither constraint was written down. | Documented as an explicit assumption: migrations must be expand/contract; concurrent boots are normally safe via Postgres advisory locks, with a first-deploy caveat. |

### Observability coverage

| Finding | Resolution |
|---|---|
| Part 4 covered the Identity Server in depth and gave Rails a log-group name and little else, despite the requirement asking for both services. | Added parallel Rails SLOs, PostgreSQL alarms, and a Puma request-queue-time signal. |
| Tracing was written up as Identity/.NET-only with no mention of Rails. | Added one sentence noting Rails would use the same ADOT sidecar + OTLP pattern. |

### Checklist

| Finding | Resolution |
|---|---|
| `.dockerignore` was listed as a Part 3 deliverable without independent confirmation it was actually committed. | Confirmed present in the repo. |

---

*Compiled during the StrongMind take-home exercise. Entries are appended as each part of the exercise is worked through with AI assistance — kept as evidence of what was verified, what was assumed, and why.*
