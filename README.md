# StrongMind DevOps Exercise

Deliverables for the Staff DevOps Engineer technical exercise: an Identity Server Azure→AWS migration plan, a standardized Rails CI/CD pipeline, a production Dockerfile, and an observability plan.

## How to navigate this repo

| File | Part | What it is |
|---|---|---|
| [`ADR.md`](./ADR.md) | 1 | Architecture Decision Record for the Identity Server migration — context, target architecture, cutover strategy, database migration plan, risk register, definition of done |
| [`.github/workflows/rails-deploy.yml`](./.github/workflows/rails-deploy.yml) | 2 | Standardized GitHub Actions pipeline: test → build → push to ECR → deploy to ECS Fargate, with OIDC auth and automated rollback |
| [`Dockerfile`](./Dockerfile) / [`.dockerignore`](./.dockerignore) | 3 | Multi-stage production Dockerfile for the Rails application |
| [`OBSERVABILITY.md`](./OBSERVABILITY.md) | 4 | SLOs/SLIs, CloudWatch metrics and alarms, X-Ray tracing, log strategy, and the alerting pipeline to on-call |
| [`docs/decision-log.html`](./docs/decision-log.html) | — | Not a required deliverable. A running, source-cited record of the non-obvious calls made while writing the above — each entry shows the question, the research that settled it (with links to primary sources), the decision, and the reasoning. Open it directly in a browser; it's referenced by ID (D1, D2, …) throughout `ADR.md` and `OBSERVABILITY.md` wherever a specific claim needed a citation rather than an assertion. |

## Approach

Two workstreams, worked as a connected system rather than four independent documents — several decisions in Part 1 constrain what's correct in Part 2, and vice versa. The throughline across all of it: verify the "obvious" default before writing it down, and be honest about the cost of a decision instead of overclaiming (e.g., "zero-downtime" where the tooling can't actually deliver that). Concretely:

- **Identity Server (Part 1):** containerize onto the *existing* production ECS Fargate cluster rather than new infrastructure, migrate the database via a short, honest maintenance window rather than a live cutover the available tooling doesn't support (AWS DMS has no CDC path from Azure SQL Database — see decision log D8), and replace the legacy Azure AD Domain Services dependency rather than replicate it into AWS (D1).
- **Rails pipeline (Part 2) and Dockerfile (Part 3):** designed together, since the pipeline's deploy step and the image it deploys have to agree on how the app starts, migrates its schema, and reports health. Both standardize on ECS's native rolling deployment with the deployment circuit breaker (D2) rather than introducing CodeDeploy as a second control plane for every Rails team to learn.
- **Observability (Part 4):** built to close the specific gap named in the prompt — "I can't tell if something is wrong until a customer calls" — so alarms are tied to the SLOs, tracing is tied to the alarms (what to check *when* an alarm fires), and the alerting pipeline has an actual mechanism (CloudWatch → SNS → Jira Ops), not just a named destination.

## Significant assumptions

- The Identity Server joins the same VPC and ECS cluster (`strongmind-production`) already hosting the Rails app, rather than new networking.
- Only one environment (`production`) currently exists; the Rails workflow's `workflow_dispatch` environment input is wired for it but structured so adding a second environment is a config change, not a rewrite.
- The Rails app follows Rails 8 defaults (Propshaft + importmap) for its asset pipeline, so the Dockerfile's build stage doesn't install Node/Yarn. If the real app uses `jsbundling-rails`/`cssbundling-rails` with esbuild, that stage needs a Node layer added.
- `bin/docker-entrypoint` exists in the app (the Rails 7+ default: run `rails db:prepare`, then exec) and is what performs migrations at container startup — this is relied on by both the Dockerfile and the deploy pipeline, and is the same assumption given for Part 2 in the exercise prompt.
- The token-signing certificate migrates as-is rather than rotating during the move, so already-issued tokens keep validating across the cutover.
- The `github-actions-rails-deploy` IAM role referenced by the workflow's OIDC step doesn't exist yet — it's specified with least-privilege intent in `ADR.md`'s IAM section, but actually provisioning it (and the GitHub OIDC identity provider in IAM) is a one-time setup step this repo doesn't include, consistent with the exercise's "nothing needs to be deployed" scope.
- ECS task sizing (1 vCPU/2GB, `db.r6i.large` for RDS) and the SLO targets in `OBSERVABILITY.md` are reasoned starting points based on the stated traffic profile, not numbers measured against StrongMind's actual load — flagged explicitly in both documents as needing validation once real data exists.
- Auth/audit log retention is called out in `OBSERVABILITY.md` as probably needing to exceed the 30-day default given StrongMind's K-12 customer base, but the actual number is left to StrongMind's compliance requirements rather than picked here.
- Migrations run at container startup (per the assumption above) with `minimumHealthyPercent: 100`, which means old and new task revisions serve traffic against the same database simultaneously during every rolling deploy — this only stays safe if migrations are written expand/contract (additive, backward-compatible with the previous release), which isn't enforced anywhere in this repo since there's no actual migration to check against.
- `rails-deploy.yml`'s failure-notification step assumes a `OPS_ALERTS_SNS_TOPIC_ARN` secret pointing at the same SNS topic `OBSERVABILITY.md`'s CloudWatch alarms already publish to — this secret and the underlying topic don't exist yet, consistent with the "nothing needs to be deployed" scope.

## Intentionally scoped out (what I'd do with more time)

- **Infrastructure as code.** The exercise is explicit that nothing needs to be deployed and no AWS account is required, so this repo is documents and pipeline/container config only — no Pulumi (StrongMind's actual IaC tool) stack for the VPC, ECS services, RDS instance, or IAM roles described in the ADR. In a real migration this would exist and be reviewed alongside the ADR, not after it.
- **A line-by-line, copy-paste command runbook for the actual cutover night.** The ADR describes the cutover sequence (freeze, final sync, DNS ramp, rollback trigger) as a design; it doesn't include the literal ordered list of commands (`aws rds`, `sqlpackage`, `aws route53 change-resource-record-sets`, the validation queries) someone would run in order at 2 AM with no room to improvise. That's the natural next artifact once the design here is agreed on, and it's worth writing before the actual migration night, not during it.
- **The actual AADDS call-site audit** that `ADR.md` specifies as a Week-1 step (D1) — this requires the real Identity Server codebase, which isn't part of this exercise.
- **Load testing** to validate the ECS/RDS sizing decisions against real traffic rather than the stated req/min figures.
- **A JWKS diff script** to automate the pre/post-migration signing-key verification called out in the ADR's Definition of Done — described as a check to run, not built as a script here.
- **Secrets Manager rotation and the ALB's ACM certificate/HTTPS listener** — specified in the ADR (rotation deferred until after the migration stabilizes, per the Migration Architecture section) but not implemented as actual configuration.
