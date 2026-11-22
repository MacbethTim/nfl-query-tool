# NFL Query Tool — Phase 1 Roadmap

High-level steps for phase 1, in rough order. Not tasks, just the outline of the work and the decisions each step raises. Local before cloud (C14).

| # | Step | Decisions it raises |
|---|---|---|
| 1 | **Finish requirements**: fill in Appendix A reference values and typical-question examples | — |
| 2 | **Minimal project setup**: repo layout, decisions.md, spec template, ClickUp space, branch/PR workflow; everything agents need before any work (C10) | Repo layout; spec location and naming; spec format; task ↔ spec ↔ R/Q linking; branch/PR workflow |
| 3 | **Source data exploration** (laptop/notebook, no infra): how nflverse publishes and refreshes (release timing, file formats, how revisions appear), keys and grain of the four datasets, quirks, data volume | Pull mechanism; detecting new/revised data (Q1, R2); what "release timestamp" means; whether requirements need adjusting |
| 4 | **Requirements→spec skill** (C16): built from the patterns in the hand-written exploration specs | What the skill takes in and produces; how it traces specs to R/Qs; where it lives in the repo |
| 5 | **Landing & data model design**: where raw data lands; raw/staging/modeled layers; change history (R3); workspace schema (R6); load-state metadata (R8); in-database docs (R9) | Landing zone (S3, disk, raw schema); schema layout; history approach; doc storage and enforcement; naming conventions |
| 6 | **Accounts & access**: AWS account lockdown (root MFA, admin identity), billing alarms, Pulumi state backend, Cloudflare API token, GitHub secrets | IAM vs. Identity Center; Pulumi state location; secrets storage and passing |
| 7 | **Local dev**: Docker, Postgres + Airflow containers, Python project setup, CI (secret scan, R9 doc check, notebook output check) | Python tooling; Airflow executor; compose layout; CI platform |
| 8 | **Pipelines**: extract/land/load, validation (Q4), atomic publish (Q3), idempotent catch-up (Q2) | Full vs. incremental loads; atomic swap mechanism; validation rules |
| 9 | **Orchestration & monitoring**: DAGs and schedules (Q1), failure/staleness alerts (Q5), planned outages (Q6) | Schedule cadence; notification channel; off-box staleness detection |
| 10 | **Pulumi infrastructure**: network, security groups, EC2/Linux, storage, S3 (landing if chosen, backups), DNS; dev vs. prod stacks (C14) | Instance size and OS; network design and allowed IPs; stack/config structure; cost vs. C2 |
| 11 | **Server configuration**: Linux hardening, Docker, Postgres + Airflow containers, TLS, logins/roles (Q7), backups (Q8) | Config approach (cloud-init, scripts, Ansible); Postgres role model; backup tool and schedule |
| 12 | **Deploy & backfill**: first production deploy, historical load 1999–present | Backfill method (bulk vs. normal pipeline) |
| 13 | **Verify**: every R/Q Verify step, teardown/rebuild (Q9), S2 exploration, S3 baseline | Exception criteria under S1 |

## Notes
- Exploration specs are written by hand; they're the practice runs the step 4 skill is built from.
- Steps 3 and 5 drive most of what follows (pipelines, orchestration, storage) and may feed back into requirements.
- Steps 10–11 can run in parallel with 8–9 once step 7 is done, against the dev stack only.
