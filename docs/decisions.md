# NFL Query Tool — Decisions

*Status: draft. Platform, infrastructure, and deployment decisions, made before specs so specs don't carry "TBD: depends on hosting." Each entry lists the options, the pick, and the reason. Items marked **[Tim to decide]** still need a call; **Accepted** items are settled; **Proposed** items stand unless Tim pushes back.*

*Companion to `requirements.md`. Once a decision is accepted, don't reopen it without a new entry that supersedes it.*

---

## Guiding stance

The project is a portfolio piece for a **product-engineering + analytics** role as well as a skills refresh. That sets the bar for infrastructure:

> **Robust, not complicated.** Every component should be standard practice at a real company (reproducible, tested, deployed automatically, observable), but sized to the problem: one user, a few GB of data, one nightly job.

Over-engineering for its own sake is a negative signal for a product-engineering audience, which hires for judgment. Where a heavier option was considered and rejected, the rejection is recorded below. That record is part of the showcase.

Tie-breaker, from requirements principle 1: when two options are otherwise equal, pick the one that exercises a rusty skill (orchestration, Linux, cloud deployment, dimensional modeling, MCP tool design).

---

## Target architecture (one picture)

```
                        Claude (web / desktop / mobile)
                                      │  HTTPS, authenticated
                                      │  https://nfl.timfox.dev
                                      ▼
┌──────────────────────── AWS EC2 VM (Linux, ARM) ───────────────────┐
│  Caddy (reverse proxy, automatic TLS)                              │
│     │                                                              │
│     ▼                                                              │
│  MCP server (Python)  ──reads──►  Postgres  ◄──writes──  Airflow   │
│     │                             (dimensional model)    (nightly  │
│     └── glossary.md, feedback queue                       DAG)     │
│                                                              │     │
│  All services run in Docker Compose                          ▼     │
└──────────────────────────────────────────────────── nflverse data ─┘
        ▲                                     │
        │ Pulumi, Python (provision AWS       ├─► S3 (nightly DB backups)
        │   + Cloudflare DNS record;          └─► CloudWatch / alerts / AWS Budgets
        │   state in Pulumi Cloud)
        │ GitHub Actions (test + deploy)
```

---

## D1. Data source

- **Options:** nflverse (nflreadr / nflverse-data releases); scraping NFL.com or ESPN directly; a paid feed (e.g. Sportradar).
- **Decision:** **nflverse** play-by-play and player-stats releases.
- **Why:** Free, well documented, and play-by-play coverage starts in 1999, which matches the v1 history scope. Play-by-play files are updated nightly during the season, which matches "current through last night." Paid feeds are out per the cost stance, and scraping is fragile.
- **Spec-time questions:** what time the nightly release usually lands (this sets the DAG schedule); how to detect a stale or failed upstream release; whether to load the parquet releases directly or go through a client library.
- **Status:** Proposed.

## D2. Hosting and cost ceiling

- **Options:** a single VM (EC2); serverless (Lambda + API Gateway + managed DB); managed containers (ECS/Fargate); Kubernetes (EKS); a non-AWS VPS (DigitalOcean, Hetzner).
- **Decision:** **one EC2 Linux VM** (Ubuntu LTS) in us-east-2 (Ohio, near Chicago).
- **Cost ceiling:** **$20/month, all-in** for AWS. The domain is billed separately by Cloudflare (D9). Accepted.
- **What the ceiling means:** Airflow plus Postgres wants about 4 GB of RAM. An on-demand x86 4 GB instance alone runs over $20/month, so:
  - **Instance:** **t4g.medium** (ARM/Graviton, 4 GB). It's cheaper than the x86 equivalent, but on-demand it is still close to the ceiling. Commit to a **1-year no-upfront Compute Savings Plan** once the skeleton proves the size is right, to get it comfortably under. Fallback: **t4g.small** (2 GB) plus swap, with Airflow trimmed to the minimum (D6), if the larger instance doesn't fit.
  - **Other line items to budget:** EBS storage (about 20–30 GB gp3), the public IPv4 address charge (AWS bills every public IPv4 by the hour, including Elastic IPs), CloudWatch Logs (keep retention short), and S3 backups (negligible). There's no Route 53 hosted-zone charge, because DNS stays on Cloudflare (D9).
  - **ARM consequence:** every image must be built for or published as `arm64`. Postgres, Airflow, and Caddy publish arm64 images, and CI builds the MCP server image for arm64.
  - **Guardrail:** an **AWS Budgets** alert at $15 (forecasted) and $20 (actual), provisioned in Pulumi (D3).
  - **Verify before committing:** run the instance + EBS + IPv4 combination through the AWS Pricing Calculator. Prices here are approximate.
- **Note:** Claude connects to remote MCP servers from Anthropic's cloud, not from my device, so the server must be reachable on the public internet over HTTPS.
- **Status:** Accepted (VM, region, ceiling); instance size Proposed pending the pricing check.

## D3. Provisioning: infrastructure as code

- **Options:** click through the AWS console; AWS CloudFormation/CDK; Terraform; Pulumi.
- **Decision:** **Pulumi, written in Python**. It provisions the AWS resources (VPC/subnet, security groups, EC2 instance, IAM role, S3 backup bucket, Elastic IP, AWS Budgets alert) and, through the **Pulumi Cloudflare provider**, the `nfl.timfox.dev` DNS record.
- **Why:** Tim's call. It also fits the project well: infrastructure is written in the same language as the pipeline and server, with real types, tests, and functions instead of a separate configuration language, and principle 3 (understand what the agents build) is easier when everything is Python.
- **State backend:** **Pulumi Cloud, free individual tier.** Accepted.
  - *What state is:* Pulumi's record of every resource it has created and its current settings. On each run, Pulumi compares the code against this record to decide what to create, change, or delete. That's the same idea as the pipeline's load-state record (D12).
  - *Why Pulumi Cloud:* setup is just `pulumi login`. It includes deployment history, locking so two runs can't collide, and built-in encryption for config secrets. It keeps the skeleton focused on the skills the project is meant to exercise.
  - *CI access:* GitHub Actions authenticates with a Pulumi access token stored as a GitHub Actions secret (`PULUMI_ACCESS_TOKEN`), never in the repo.
  - *Exit path:* `pulumi stack export` / `import` can move the state to a self-managed S3 backend later if wanted.
- **Cloudflare access:** use a Cloudflare API token scoped to **DNS edit on the `timfox.dev` zone only**, stored as a Pulumi encrypted config secret, never in the repo.
- **Scope of ownership:** this project's Pulumi code owns only the `nfl` record. The `timfox.dev` zone itself, and records for other projects, are managed outside this repo.
- **Stacks:** one `prod` stack to start; a `dev` stack only if it earns its cost.
- **Scope guard:** Pulumi owns the cloud resources. It doesn't configure software on the box; Docker Compose does that (D4).
- **Status:** Accepted.

## D4. Runtime: containers

- **Options:** install everything directly on the VM; Docker Compose; Kubernetes (k3s or EKS).
- **Decision:** **Docker Compose** on the VM: `postgres`, `airflow` (scheduler, webserver, and so on), `mcp-server`, `caddy`.
- **Why:** Each service is isolated, versioned, and portable, and the same Compose file runs on my laptop for local development. Moving a service to ECS or the database to RDS later is a configuration change, not a rewrite.
- **Note:** set memory limits per container so Airflow can't starve Postgres on a 4 GB box.
- **Status:** Proposed.

## D5. Database

- **Options:** Postgres; DuckDB; SQL Server (my day-job comfort zone).
- **Decision:** **Postgres 16+** in a container, with a volume on the VM's disk.
- **Why:** A real server database suits a dimensional model with one writer (Airflow) and one reader (MCP server) running at the same time. Running and backing it up is a transferable skill. DuckDB is great for analytics but teaches less about operating a database. SQL Server would add the least new learning.
- **Spec-time questions:** star schema grain (play, drive, game, player-game); slowly changing dimensions for players and teams; the separation between raw/staging and modeled schemas.
- **Status:** Proposed.

## D6. Orchestration

- **Options:** cron; Airflow; Dagster or Prefect.
- **Decision:** **Airflow**, the official Compose setup trimmed to the minimum (LocalExecutor, no Celery or Redis), which matters for the memory budget in D2.
- **Why:** The requirements say "an Airflow DAG I write myself." Dagster is arguably nicer for asset-based data work, but Airflow is the gap I named.
- **Refresh failure and recovery:** see D12. The DAG is built around a **watermark**: it records the last successful load, and every run loads from that point forward, so a missed night recovers itself.
- **Status:** Proposed.

## D7. MCP server stack

- **Options:** Python with the official MCP SDK; TypeScript with the official MCP SDK.
- **Decision:** **Python**, using the official MCP Python SDK, served over Streamable HTTP.
- **Why:** Python is my strongest language, so I can read and debug everything agents write (principle 3). The learning is in the tool design, not the language.
- **Spec-time questions:** curated tools vs. a general query tool (requirements §5); the response envelope that always carries the applied filters, definitions, scope, and data as-of date; how the glossary and feedback queue are exposed.
- **Status:** Proposed.

## D8. Authentication

- **Options:** (a) a static secret sent in a request header; (b) full OAuth 2.1 with the server acting as its own authorization server; (c) OAuth through a hosted identity provider (AWS Cognito, Auth0, etc.).
- **Decision:** **Phase it.** Start the walking skeleton with **(a)** request-header auth to get the end-to-end path working, then move to **OAuth via a hosted provider (c)** as its own spec.
- **Why:** Claude's custom connectors support both OAuth and request-header auth. OAuth is the standard for remote MCP and the more marketable skill, but a self-hosted OAuth flow is a known trouble spot. A hosted identity provider teaches the protocol without hand-rolling an authorization server. Cognito's free tier keeps it inside the D2 budget.
- **Check before implementing:** Claude's current connector authentication docs (support.claude.com and claude.com/docs/connectors), since this area changes.
- **Status:** Proposed.

## D9. Domain, DNS, ingress, and TLS

- **Domain:** **`timfox.dev`**, registered with **Cloudflare Registrar** (recommended by Brian Stauff). It's a personal umbrella domain: each project gets its own subdomain, and the root can later hold a portfolio page.
- **This project's address:** **`nfl.timfox.dev`**. The Claude custom-connector URL will be `https://nfl.timfox.dev/<mcp path>`, with the path set in the server spec.
- **DNS:** stays on **Cloudflare** (free), and the `nfl` A record points at the EC2 Elastic IP, managed in Pulumi (D3).
  - **Start with the record set to "DNS only" (grey cloud), not proxied.** Then Caddy gets its own Let's Encrypt certificate with no configuration, and Claude talks straight to the server. Cloudflare's proxy (orange cloud: DDoS shielding, hides the origin IP) is a possible later hardening step, but it changes how TLS and timeouts work, so it gets its own small spec if wanted.
  - `.dev` domains are HTTPS-only in browsers (HSTS preloaded). That's no issue here, since everything is served over TLS anyway.
- **Ingress options:** Caddy; Nginx + Certbot; an AWS Application Load Balancer with an ACM certificate.
- **Ingress decision:** **Caddy** as a reverse proxy in front of the MCP server, with automatic Let's Encrypt certificates. Only ports 80 and 443 are open to the internet. SSH is restricted to my IP, or replaced with AWS SSM Session Manager.
- **Why:** HTTPS with almost no configuration. A load balancer would cost more than the whole $20 budget allows.
- **Status:** Accepted (domain, registrar, DNS host, subdomain); Caddy Proposed.

## D10. CI/CD

- **Options:** deploy by hand (SSH, git pull); GitHub Actions.
- **Decision:** **GitHub Actions.** Every push runs lint, type checks, unit tests for the server tools, DAG import tests, and secret scanning (D14). A merge to `main` builds arm64 images and deploys to the VM. Pulumi changes run `pulumi preview` on PRs (state read from Pulumi Cloud, D3) and `pulumi up` by hand.
- **Why:** The clearest signal of engineering maturity in a public repo, and it keeps me honest about tests.
- **Sequencing:** manual deploys are fine for the walking skeleton; automate in the next slice.
- **Status:** Proposed.

## D11. Secrets

- **Options:** a `.env` file on the box; AWS Systems Manager Parameter Store; AWS Secrets Manager; Pulumi config secrets.
- **Decision:** **SSM Parameter Store** (SecureString) for **runtime** secrets (database password, auth token), read at deploy time through the VM's IAM role. **Pulumi encrypted config** (encrypted by Pulumi Cloud, D3) for **provisioning-time** secrets only, such as the Cloudflare API token. **GitHub Actions secrets** for CI credentials, such as `PULUMI_ACCESS_TOKEN`. Nothing secret in the repo, ever, and `.env` files are generated and git-ignored.
- **Why:** Free at this scale and the standard AWS pattern. Secrets Manager adds rotation I don't need, plus a per-secret monthly charge.
- **Status:** Proposed.

## D12. Observability, alerting, and refresh failure

- **Refresh-failure behavior:** **Accepted.**
  - The pipeline keeps a **load-state record** in Postgres: last successful refresh timestamp, the data date it covers, and the status and error of the most recent attempt.
  - Each run **backfills from the last successful data date** forward, not just "yesterday," so a failed night is picked up automatically the next time it succeeds. Loads must be idempotent (re-running a date range gives the same result).
  - The MCP server keeps serving the last good data and includes the **data as-of date** in every tool response, so staleness shows up in Claude's answers instead of being hidden.
- **Monitoring:**
  - Health-check endpoint on the MCP server, plus an external uptime check against `nfl.timfox.dev`.
  - An Airflow failure callback that notifies me (email or a push service) when a refresh fails.
  - Container logs sent to CloudWatch Logs with short retention (cost, D2).
  - AWS Budgets alerts (D2).
- **Why:** How you show you can run a system, not just deploy one.
- **Status:** Accepted (refresh behavior); monitoring Proposed.

## D13. Backups and recovery

- **Decision:** nightly `pg_dump` to S3 with a lifecycle rule that keeps about 14 days. A documented restore runbook, tested once. Because the warehouse can be rebuilt from nflverse, the backups mainly protect the **feedback queue, load-state history, and glossary history** (the glossary is also in git).
- **Status:** Proposed.

## D14. Repository

- **Layout:** **one repo**, with top-level folders for `infra/` (Pulumi), `pipeline/` (Airflow DAGs and transforms), `server/` (MCP server), `glossary/`, `docs/` (requirements, decisions, specs, runbooks), and `docker-compose.yml`.
- **Visibility:** **Public.** Accepted.
- **What public requires from day one:**
  - Secret scanning: GitHub secret scanning and push protection turned on, plus a `gitleaks` pre-commit hook and CI step.
  - No secrets or environment-specific values in code: IPs and account-specific settings live in Pulumi stack config or Parameter Store. The public hostname `nfl.timfox.dev` isn't a secret and can appear in docs.
  - A license file (MIT or similar) and a README written for an outside reader from the start.
  - If a secret is ever committed: rotate it first, then clean history. Rotation is what actually fixes it.
  - Data note: the README credits nflverse as the data source, and raw data files aren't committed (they're pulled by the pipeline).
- **Status:** Accepted.

## D15. Showcase

The infrastructure won't be what sets this project apart. The product and data decisions are: tool design for an AI client, responses that show their interpretation, the governed glossary, and the feedback loop that turns errors into test cases. Make those visible:

- A README with the architecture diagram, a "why it's built this way" section, and links to this file
- A short recorded demo (2–3 minutes) of real questions asked from Claude on a phone, including a wrong answer being logged and fixed
- Since v1 is single-user, the public repo and the demo *are* the showcase. Read-only demo access is a possible later addition, not v1.
- Later: a simple portfolio page at the root `timfox.dev` linking to this project and future ones.
- **Status:** Proposed.

---

## Considered and rejected

| Option | Why not (for v1) |
|---|---|
| Terraform | Tim chose Pulumi: infrastructure in Python alongside the rest of the codebase. Terraform remains the more common tool in job postings; the concepts carry over. |
| Pulumi state in a self-managed S3 bucket | More setup (state bucket created by hand, own encryption key) for no v1 benefit. Pulumi Cloud can be exported to S3 later. |
| Pulumi state as a local file | Lost if the laptop dies, and CI can't read it to run previews. |
| Route 53 for DNS | The domain is registered at Cloudflare, whose DNS is free; Pulumi manages Cloudflare records just as well. |
| Cloudflare proxy (orange cloud) at launch | Changes the TLS setup and adds a layer to debug during the skeleton; revisit as hardening. |
| Kubernetes (EKS or k3s) | Orchestrating containers for one user on one box adds operational load with no benefit. EKS alone exceeds the $20 budget. |
| Serverless (Lambda + API Gateway) | Airflow and a long-running Postgres don't fit serverless, and it would skip the Linux operations work I need. |
| Managed Postgres (RDS) | Hides the database operations I want to learn and would take most of the $20 budget by itself. |
| ECS/Fargate | Reasonable "next step up"; v1 doesn't need it, and always-on Fargate tasks strain the budget. |
| Application Load Balancer | Its monthly base charge alone takes most of the budget; Caddy does the job for free. |
| Multi-AZ / high availability | A single user tolerates downtime; the nightly refresh and the backups are what matter. |
| Microservices | One server, one database. Splitting would add network hops and deployment surface for nothing. |
| Dagster / Prefect | Good tools, but the named skill gap is Airflow. |
| SQL Server | Day-job comfort zone; least new learning. |
| Self-hosted OAuth authorization server | Error-prone and not the learning goal; a hosted identity provider teaches the protocol with less risk. |

---

## Sequencing (walking skeleton first)

1. **Skeleton:** Pulumi (state in Pulumi Cloud) for the VM, network, budget alert, and the `nfl.timfox.dev` record → Compose with Postgres, MCP server, and Caddy → one season loaded by a manual script → one MCP tool → header auth → Claude answers one real question from my phone.
2. **Pipeline:** Airflow DAG with the watermark/backfill pattern replaces the manual load; full 1999–present history; first pass at the dimensional model.
3. **Hardening:** CI/CD, secrets in Parameter Store, alerting, backups plus a tested restore.
4. **Product layer:** glossary tool, feedback queue, response envelope with interpretation and as-of date, curated tools.
5. **Auth upgrade:** OAuth via a hosted identity provider.
6. **Showcase:** README, demo recording.

Each step gets its own spec and task list before any agent work starts.

---

## Open decisions for Tim

1. Instance size, after the pricing-calculator check (D2); can be confirmed during the skeleton spec
