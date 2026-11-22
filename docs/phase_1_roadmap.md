# NFL Query Tool — Phase 1 Roadmap

High-level steps for phase 1, in rough order. Not tasks, just the outline of the work and the decisions each step raises. Local before cloud (C14).

| # | Step | Decisions it raises |
|---|---|---|
| 1 | **Finish requirements**: fill in Appendix A reference values and typical-question examples | — |
| 2 | **Source data exploration** (laptop/notebook, no infra): how nflverse publishes and refreshes (release timing, file formats, how revisions appear), keys and grain of the four datasets, quirks, data volume | Pull mechanism; detecting new/revised data (Q1, R2); what "release timestamp" means; whether requirements need adjusting |
| 3 | **Landing & data model design**: where raw data lands; raw/staging/modeled layers; change history (R3); workspace schema (R6); load-state metadata (R8); in-database docs (R9) | Landing zone (S3, disk, raw schema); schema layout; history approach; doc storage and enforcement; naming conventions |
| 4 | **Set up the project**: repo structure, ClickUp space, spec template, requirements→spec skill (C16), decisions.md | Repo layout; spec format; task ↔ spec ↔ R/Q linking; branch/PR workflow |
| 5 | **Accounts & access**: AWS account lockdown (root MFA, admin identity), billing alarms, Pulumi state backend, Cloudflare API token, GitHub secrets | IAM vs. Identity Center; Pulumi state location; secrets storage and passing |
| 6 | **Local dev**: Docker, Postgres + Airflow containers, Python project setup, CI (secret scan, R9 doc check) | Python tooling; Airflow executor; compose layout; CI platform |
| 7 | **Pipelines**: extract/land/load, validation (Q4), atomic publish (Q3), idempotent catch-up (Q2) | Full vs. incremental loads; atomic swap mechanism; validation rules |
| 8 | **Orchestration & monitoring**: DAGs and schedules (Q1), failure/staleness alerts (Q5), planned outages (Q6) | Schedule cadence; notification channel; off-box staleness detection |
| 9 | **Pulumi infrastructure**: network, security groups, EC2/Linux, storage, S3 (landing if chosen, backups), DNS; dev vs. prod stacks (C14) | Instance size and OS; network design and allowed IPs; stack/config structure; cost vs. C2 |
| 10 | **Server configuration**: Linux hardening, Docker, Postgres + Airflow containers, TLS, logins/roles (Q7), backups (Q8) | Config approach (cloud-init, scripts, Ansible); Postgres role model; backup tool and schedule |
| 11 | **Deploy & backfill**: first production deploy, historical load 1999–present | Backfill method (bulk vs. normal pipeline) |
| 12 | **Verify**: every R/Q Verify step, teardown/rebuild (Q9), S2 exploration, S3 baseline | Exception criteria under S1 |

## Notes
- Steps 2–3 drive most of what follows (pipelines, orchestration, storage) and may feed back into requirements.
- Step 4 can run alongside 2–3; it doesn't depend on them.
- Steps 9–10 can run in parallel with 7–8 once step 6 is done, against the dev stack only.
