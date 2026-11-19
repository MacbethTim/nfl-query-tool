# NFL Query Tool — Requirements (Phase 1)
 ---
## 1. Purpose

The purpose of this project is to gain hands-on skill using indusrty standard tools in analytics engineering in a cloud environment. With the paradigm shift towards ai-driven development, it is imperative to have the proper guardrails in place to ensure that agents adhere to the intended motivations behind requirements and design considerations. This project is not just about "how to build a DAG in airflow" or "how to deploy an EC2 instance in AWS", but how to get agents to reliably complete this work that results in a functional product.

This initial phase - a database hosted in the cloud - is meant to be simple so that I can learn the fundamentals of deploying analytics resources in the could, AI-driven development, and the foundations of AI Engineering.

## 2. Users & use

**Who:** Me, as the only human user. Agents also read production, but only through a read-only login (C14).

**How:** Exploratory, ad hoc SQL from a standard client on my laptop (R5), against the modeled tables. I save my own views in a personal workspace (R6) and promote the ones worth keeping (R7). The questions are after-the-fact analysis, not live game tracking.

**Typical questions:**
- <player-season: e.g. Caleb Williams' completion % and yards/attempt by season>
- <game/situational: e.g. Bears 4th-down attempts and conversion rate, 2024>
- <career/leaderboard: e.g. top 10 career receiving yards, 1999–present>

**Looking ahead:** In phase 2, Claude queries the modeled layer through an MCP server. Phase 1 table and column design should assume a second reader that can't guess at meaning.


## 3. Functional

**R1 — The database contains play-by-play, schedules, player stats, and rosters/players from nflverse for every regular-season and postseason game from the 1999 season through the most recent completed game.**
Verify: for each season 1999–present, distinct game count in play-by-play matches the schedule's regular-season and postseason games; player stats and rosters have rows for every season in range.

**R2 — New and revised data is added to the database automatically, with no manual steps, after nflverse publishes it.**
Verify: over one full NFL week after deployment, every completed game appears without anyone triggering a load; a value changed upstream in the current season is reflected after the next refresh.

**R3 — When a refresh revises existing data, I can query what changed: the previous value, the new value, and when it changed.**
Verify: after a known upstream correction, a query returns both versions of the affected rows and the refresh in which the change happened.

**R4 — Common analytical questions can be answered by querying modeled tables only, without touching raw data or rebuilding logic from it.**
Verify: each question in Appendix A is answered by a query that uses only modeled tables, and the result matches a trusted reference (e.g. Pro Football Reference).

**R5 — I can connect to the database from my own machine using a standard SQL client and run queries against the modeled tables.**
Verify: from my laptop, outside the cloud network, I connect with a standard client and run an Appendix A query successfully.

**R6 — I can create my own views and tables in a personal workspace without being able to change modeled data. They keep working through refreshes and non-breaking rebuilds. When a model change breaks one, it is never silently lost: I'm told which objects were affected and can recover their definitions.**
Verify: a write to a modeled table from my login is rejected; a workspace view survives a routine refresh and a full reload; after a model change that drops a column the view uses, I'm notified that the view is affected and can retrieve its original definition.

**R7 — A workspace view can be promoted into the modeled layer, after which the pipeline maintains it. A promoted view can later be rebuilt as a Python pipeline that produces a table, when warranted.**
Verify: a promoted view exists in the modeled layer, is defined in the repo, and is rebuilt by a refresh.

**R8 — I can query when the data was last refreshed, what date it covers, and whether the most recent refresh succeeded.**
Verify: after a successful refresh, a query returns its timestamp and covered date; after a forced failure, the same query shows the failure and still reports the last successful refresh.


## 4. Quality

**Q1 — Data nflverse publishes is queryable within 12 hours of publication, whatever day the game was played, unless that data is rejected under Q4.**
Verify: over one full week that includes Thursday, Sunday, and Monday games, the gap between each nflverse release timestamp and the matching load timestamp (R8) is never more than 12 hours, excluding releases rejected under Q4.

**Q2 — After failed or missed refreshes, however many, the next successful refresh catches up on everything missed with no manual steps, and re-running a refresh never changes the result.**
Verify: after a forced refresh failure, and separately after refreshes are stopped for several days that include games, the next successful refresh loads every missed game and correction without help; re-running a refresh over data already loaded leaves modeled tables identical and adds no entries to the change history (R3).

**Q3 — Queries never see a partially applied refresh: modeled tables always reflect either the last complete refresh or the new one, never a mix.**
Verify: record row counts and a content hash for every modeled table; (a) force a refresh to fail after it has updated at least one modeled table, and confirm every fingerprint is unchanged; (b) pause a refresh at the same point, confirm a separate session still sees the original fingerprints for every table, then let it finish and confirm every table reflects the new refresh.

**Q4 — A refresh that would publish incomplete or internally inconsistent data is rejected, and the last good data stays queryable.**
Verify: feed a test refresh known-bad inputs — a scheduled game missing from play-by-play, a duplicated play, a player stat row for a game that doesn't exist, player stat totals that disagree with play-by-play for a game — and confirm each is rejected, modeled tables are unchanged (Q3 fingerprints), and R8 reports the failure and its reason.

**Q5 — I'm notified within 1 hour when a refresh fails or is rejected, and when published nflverse data has gone unloaded for longer than Q1 allows, without having to check.**
Verify: a forced refresh failure and a Q4 rejection each produce a notification within 1 hour that states the reason; with the server shut down entirely, a notification arrives once newly published nflverse data passes the Q1 limit; no staleness notification fires during a week with no new nflverse data.

**Q6 — I can declare a planned outage, with an end time or open-ended, that suppresses Q5 staleness notifications. A timed outage resumes notifications at its end time; an open-ended one sends me a monthly reminder until I end it.**
Verify: with the server shut down under a timed outage, no staleness notification fires before the end time and one fires after it if data is still unloaded; under an open-ended outage, no staleness notifications fire but a reminder arrives each month; ending the outage restores normal Q5 behavior.

**Q7 — Only I can connect to the database, and only from networks I've approved; connections are encrypted, no login has more access than its job needs, and no credential appears in the public repo.**
Verify: a connection attempt without valid credentials is refused, and so is an unencrypted one; a connection from an unapproved network is refused even with valid credentials; after I approve a new network, I can connect from it; my everyday login cannot create or change logins or alter modeled tables (write rejection per R6); the pipeline's login cannot read or change my personal workspace; a secret scan of the full repo history finds no credentials.

**Q8 — If the server or its disk is lost, my workspace objects, change history (R3), and load state can be restored with no more than 24 hours of changes lost.**
Verify: restore the most recent backup onto a fresh server, then confirm workspace objects, change history, and load state match their state as of that backup, the backup is less than 24 hours old, and a refresh afterward catches up per Q2.

**Q9 — The entire system can be torn down and rebuilt from the repo and the latest backup, with no manual steps beyond running the provisioning and deploy commands.**
Verify: run `pulumi destroy`, confirm no billable resources remain except the backup storage, then rebuild from a clean checkout; confirm workspace objects, change history, and load state match the last backup, I can connect per R5, and the next refresh catches up per Q2.


## 5. Constraints

**C1 — Hosted on AWS, on Linux that I administer from the command line.**
Why: AWS is what my employer uses and what shows up in job postings; Linux operations are a named skill gap.

**C2 — AWS spend stays under \$35/month, all-in. The domain is billed separately.**
Why: it's a personal project; the ceiling sizes the system to the problem, and on-demand pricing avoids committing to an instance size before the skeleton proves it.

**C3 — Non-AWS services add no more than \$15/month to this project; free tiers are the default.**
Why: total cost stays near \$50/month, and a small paid tool is acceptable when it's clearly the better option.

**C4 — In phase 1, all data comes from nflverse.**
Why: it's free, covers 1999 onward, and updates nightly. A paid feed would blow the cost ceilings, and scraping is fragile.

**C5 — Airflow orchestrates all scheduled work on the server.**
Why: it's the long-standing industry standard for orchestration.

**C6 — All of this project's cloud resources are provisioned with Pulumi in Python; nothing is created by hand in the console.**
Why: infrastructure stays reproducible (Q9) and in the same language as the rest of the codebase.

**C7 — Source-to-target pipelines are Python; derived data (views over modeled tables) is SQL.**
Why: I need to read, run, and debug everything agents write; pipelines will move onto my own ETL framework (separate project) once it exists.

**C8 — The repo is public on GitHub.**
Why: it's a portfolio piece.

**C9 — Raw nflverse data is never committed to the repo, and the README credits nflverse per its CC-BY 4.0 license.**
Why: attribution is a license requirement, and the underlying NFL data remains subject to its owners' terms, so the repo holds code, not data.

**C10 — No agent works on anything without a spec and a bounded task whose acceptance criteria trace to an R or Q. Agents never provision or change cloud resources without my approval.**
Why: practicing the requirements → specs → tasks → agents workflow is half the point of the project, and I stay the gate on anything that costs money or touches production.

**C11 — Agents run on my Claude subscription; nothing in the system calls a paid LLM API.**
Why: usage-based pricing has no ceiling; the subscription is a fixed cost.

**C12 — The database is PostgreSQL.**
Why: it's the open-source industry standard, and operating it is a transferable skill.

**C13 — Every service runs in a Docker container.**
Why: containers are the standard unit of deployment, and the same images run on my laptop and in the cloud.

**C14 — Development and agent work happen locally or in an ephemeral dev stack built from the same Pulumi code as production and destroyed when testing ends; never against production. Agents may read production only through a dedicated read-only login.**
Why: mistakes stay off the live database and cost little; Q9 makes a short-lived cloud copy cheap when local isn't enough.

**C15 — Tasks are tracked in ClickUp; each task links to its spec and the R or Q it serves.**
Why: ClickUp is what I use at work, so this sharpens a skill I use daily.

**C16 — Requirements are translated into specs and tasks by a Claude skill kept in the repo.**
Why: it makes the translation repeatable, and it shows repo readers how I direct AI work, not just the code it produced.

**C17 — Every software dependency (libraries, packages, container images) is free and open source (OSI-approved license).**
Why: the repo is public, and C2/C3 cap paid services but not paid or proprietary software.


## 6. Success criteria

**S1 — Phase 1 is done when every R and Q passes its Verify step against production, and I run the checks myself. The only exceptions are failures caused by things outside the project's control, such as upstream data issues, each listed under Exceptions with a linked task.**

**S2 — The modeled layer doesn't lose what the source data can answer.**
After the build, I explore freely with SQL against the modeled tables and record every question I try. If SQL can't answer a question, I try it in pandas against nflverse directly. If pandas answers it, it's a model gap; if not, it's logged as a roadmap item. Phase 1 succeeds when, out of at least 10 questions the source data can answer, no more than 2 are model gaps. If S2 fails, each model gap becomes a task; after the fixes, S2 is rerun with new questions.

**S3 — The agent workflow held up.**
Every merged change traces to a ClickUp task, a spec and an R or Q. I record the first-review pass rate for all agent tasks as a baseline for future phases. It measures the whole workflow, from requirement to merged code, not just the agent. A task passes if its PR is approved with no "Request changes" reviews and no commits after my first review. A later task that fixes its work marks it failed, and the reason is noted on the original task.

### Exceptions
<!-- Each entry: the R/Q it applies to; what fails and why it's acceptable for now; link to the ClickUp task that fixes it. -->

## 7. Out of scope / Phase 2
- Availability/uptime targets — single user; downtime is acceptable (freshness and recovery are covered by Q1–Q2).
- My ETL framework — built in a separate project. Phase 1 transformations start as plain Python and move onto the framework once it exists.

## 8. Open questions
- Should workspace object definitions (R6) be saved to the repo automatically, as a safety net beyond Q8 backups and a record of exploration?

## Appendix A. Benchmark questions
<!-- 5–8 questions spanning grains: player-season, play-level, career, game-level -->

## Infrastructure notes (temporary — move to the new decisions.md, then delete)
- Connect by hostname (e.g. `db.timfox.dev`); Pulumi updates the Cloudflare record on each rebuild (Q9).
- Postgres TLS uses `sslmode=require` (encryption without hostname verification; meets Q7).
- After a rebuild, clear the old SSH host key from `known_hosts`.
- Backups stay in S3. The bucket is `retain_on_delete` so `pulumi destroy` completes and leaves it in place (Q9); never set `force_destroy` on it.
