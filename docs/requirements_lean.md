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


## 4. Quality          (Q1… — freshness, recovery, accuracy, security)
## 5. Constraints      (C1… — fixed tools/cost/rules + one-line reason)
## 6. Success criteria (how you'll know phase 1 is done)
## 7. Out of scope / Phase 2
## 8. Open questions

## Appendix A. Benchmark questions
<!-- 5–8 questions spanning grains: player-season, play-level, career, game-level -->