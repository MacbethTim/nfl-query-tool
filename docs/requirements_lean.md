# NFL Query Tool — Requirements (Phase 1)
 ---
## 1. Purpose

The purpose of this project is to gain hands-on skill using indusrty standard tools in analytics engineering in a cloud environment. With the paradigm shift towards ai-driven development, it is imperative to have the proper guardrails in place to ensure that agents adhere to the intended motivations behind requirements and design considerations. This project is not just about "how to build a DAG in airflow" or "how to deploy an EC2 instance in AWS", but how to get agents to reliably complete this work that results in a functional product.

This initial phase - a database hosted in the cloud - is meant to be simple so that I can learn the fundamentals of deploying analytics resources in the could, AI-driven development, and the foundations of AI Engineering.

## 2. Users & use      (who queries, how, typical questions)

I am the primary user of this product. Phase 1 will enable me to write SQL queries against this data for exploratory purposes. This phase will allow me to ask question and write the corresponding to retrieve the required data to answer my questions.


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

**R7 — A workspace view can be promoted into the modeled layer, after which the pipeline maintains it.**
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


## 5. Constraints      (C1… — fixed tools/cost/rules + one-line reason)
## 6. Success criteria (how you'll know phase 1 is done)
## 7. Out of scope / Phase 2
## 8. Open questions

## Appendix A. Benchmark questions
<!-- 5–8 questions spanning grains: player-season, play-level, career, game-level -->