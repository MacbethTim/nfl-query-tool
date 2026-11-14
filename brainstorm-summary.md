# NFL Stats Tool: Brainstorm Summary

*Session: November 10, 2026*

## Purpose

A side project to keep my skills current (main goal) by building a tool that answers NFL questions in plain English (secondary goal).

## v1 Scope

- Single user (me), analyst mode, after the fact
- Play-by-play era (~1999–present), current through the previous night
- Counting and rate stats only; no advanced or modeled metrics
- **Conversational:** follow-ups carry context forward; every answer shows how the question was interpreted and which filters are active
- **No verdicts:** answers what's measurable and says when a question needs judgment it can't make yet
- **Unanswerable questions:** the tool says what it would need (stat, data source, capability) and logs the question as a roadmap item
- **Fuzzy terms** ("elite," "terrible game"): defined in a glossary only after I actually use them. Undefined terms get a provisional definition that goes into the review queue.
- **Feedback:** right/wrong flag, failure category, correction → review queue → I approve → updates the test set, glossary, and worked examples

### Feedback failure categories

- Misread the question
- Wrong definition
- Stale context (filter carried forward when it shouldn't have)
- Data problem
- Can't answer

## Out of Scope for v1

- Live / in-game data
- Sharing with other users
- Advanced or modeled metrics
- Charting data (drops, catchable balls, pressures)
- Transactions, contracts, draft capital
- Injuries and roster status
- Predictions and betting

## Roadmap

| Version | Adds | Questions it opens up |
|---|---|---|
| v1 | Counting and rate stats, conversational follow-ups, manual feedback queue | 3, 4, 5, descriptive part of 10 |
| v1.5 | AI triage of the feedback queue (classify, propose fix, verify, check side effects, group, flag conflicts). AI proposes, never applies. Starts once there are ~30 triaged items. | (quality, not new questions) |
| v2 | Published advanced metrics (efficiency, win probability) | 1, 6, 8, stronger 11 |
| v3 | Evidence building: break judgment questions into measurable parts and summarize what the evidence supports | 1, 6, 11 as real answers; 2 and 10 as analysis |
| v4+ | New data sources: charting, transactions, injuries | 7, 9, 12 |

## Starter Question List

1. Who's the better quarterback - Caleb or Jordan Love *(v2/v3)*
2. Was Drake Maye's turnover issues this season predictable *(v3)*
3. Which other elite quarterbacks had 4 interception games *(v1)*
4. After a terrible game, how has <qb name> rebounded in the past *(v1)*
5. Which defense is best in 3rd down situations *(v1)*
6. Is the Bears offense elite *(v2/v3)*
7. Did the Bears give up too much to get Jeffery Simmons *(v4+: transaction data)*
8. With the Bears defense playing this way, should Ben Johnson go for it on 4th and 5 from his own 20 *(v2: win probability)*
9. Will the Packers attack through the air with deep shots against the Bears with their depleted secondary *(v4+: injury data)*
10. Is the running game really becoming more secondary in the NFL or is it just as important as before *(v1 descriptive; v3 analysis)*
11. Who would win - the 2026 Bears or the Greatest Show on Turf *(v2/v3)*
12. What is Caleb Williams' catchable ball rate - how often do his receivers drop these balls - does he retarget them *(v4+: charting data)*

## Parked: Glossary

- Markdown file next to the code, tracked with version control (no separate database)
- Entry structure: term, aliases, definition, why, example, status (draft/approved), added/last changed
- Starting entries:
  - **Foundational:** season (regular season by default), "this season" / "last year" (incl. January–February rule), what counts as a game for a QB, minimum sample sizes
  - **From starter questions:** elite QB, terrible game, rebound, best 3rd-down defense

## Still Open

1. **Learning goal:** what "not drifting" means and how to measure it (next thread)
2. **Product success criteria:** what makes v1 "done" and "good"
3. **Experience:** device, answer format, saved vs. fresh conversations
4. **Operating constraints:** cost ceiling, response time, handling a failed data refresh
