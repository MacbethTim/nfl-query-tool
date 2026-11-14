# NFL Query Tool — Requirements

*Status: draft. Motivation section written; v1 scope captured from brainstorm; architecture direction set (MCP server, no separate API bill); specs and tasks to follow.*

---

## 1. Why this project exists

I came into data science out of economics grad school and spent my early career building models, at West Monroe Partners, then Walgreens, then WMP again. During my second stint at WMP I grew into more of a generalist, and that accelerated at Relevant Radio, where I'm effectively the whole analytics function.

The generalist role gave me breadth, but I didn't build the depth myself. I designed a configuration-driven Python ETL framework, but a new hire learned and implemented the Airflow orchestration, so I couldn't write a DAG today without leaning on AI. The pipelines run on a Linux server I don't really know my way around. We've started deploying data apps to the cloud, and I've been hands-off there too. Our SQL Server instance is really a data lake with no real dimensional modeling. I'm its only serious user, so it has never been forced to be well designed. On top of all that, most of my time goes to manual processes and one-off requests, which leaves no room for the work that would keep my skills sharp.

So the drift is specific. I can architect and direct modern data work, but I've handed off or skipped the hands-on parts: orchestration, Linux, cloud deployment, proper data modeling, and the newer layer of LLM and agentic tooling. My day job won't close that gap on its own, because the gap comes from how my day job works.

This project is where I do the hands-on work myself. I picked NFL stats because I care about the subject and will keep coming back to it. I'm keeping v1 narrow because the point is to build every layer myself, not to answer hard football questions. Those questions are the roadmap for later versions.

> *I became the architect of systems other people built. This project makes me the builder again.*

### 1.1 Learning AI-driven development

The second goal is to learn how software actually gets built now. AI-driven development is real, and I want to practice it properly, not as "chat and ask how to code this in Python," but as a disciplined pipeline:

**requirements → specs → tasks → agents execute → I review**

This document is step one of that pipeline. The project is as much about practicing the workflow (writing requirements clear enough to become specs, specs precise enough to become tasks, and tasks bounded enough that an agent can execute them and I can verify the result) as it is about the tool itself. The same skill applies directly at work, where I want to turn a brain dump into well-formed tasks that an agent can execute with less of me in the loop.

> *Old drift: I directed people who built things. The fix isn't to stop directing; it's to get good at directing agents, while being able to check their work at every layer.*

---

## 2. Guiding principles

1. **Favor the rusty skills.** Build choices should favor the skills I'm rusty on, even when an easier path exists. Specifically, v1 should touch:
   - A real dimensional model for the play-by-play data, not another lake
   - An Airflow DAG I write myself for the nightly refresh
   - Deployment on a Linux box or VM, operated from the command line
   - A cloud-hosted **remote MCP server** (with authentication), even though it's single-user
   - Designing tools for an AI client to use (the MCP tool layer: tool definitions, descriptions, and responses)
2. **Spec before code.** Nothing gets built without a spec, and nothing gets handed to an agent without a bounded task and acceptance criteria.
3. **Understand what the agents build.** Agents can write the code, but I should be able to read, run, debug and operate everything they produce. "Vibing it" is fine as a first draft, not as the final state.
4. **Narrow v1 on purpose.** Scope stays small so the full pipeline (requirements through deployment) gets exercised end to end.
5. **No separate API bill.** All language-model work runs through my existing Claude subscription. Nothing I build calls an LLM API directly.

---

## 3. Architecture direction

Instead of a standalone query app, v1 is a **data model with an MCP server on top of it**, queried from Claude.

- **Claude is the interface.** Claude handles the conversation, follow-ups, context tracking, and translating plain English into tool calls. I don't build a chat UI or a natural-language-to-query layer.
- **The server is the contract.** Claude only sees what the server exposes, so the data model, tool design, and definitions must be good enough that answers are right without guessing.
- **The server never calls an LLM.** It serves data, definitions, and the feedback queue. All reasoning happens in the Claude session (see principle 5).
- **Remote, not local.** The server is hosted in the cloud and reachable from Claude on any device, so the project exercises real deployment, authentication, and uptime.
- **Trade-off acknowledged:** I give up building the natural-language layer myself. In exchange I learn to design tools for an AI client, which is the more current skill.
- **A standalone app remains possible later** on top of the same server, but is not planned.

---

## 4. v1 scope

- **Use mode:** analyst-style, after-the-fact querying (not a live booth tool), though the feel should be "as if calling the game" and needing facts fast
- **Data freshness:** current through last night
- **History:** play-by-play era (~1999–present)
- **Metrics:** standard stats only (counting and rate stats); no advanced metrics (e.g. EPA) in v1
- **Interaction:** conversational, with follow-up questions, via Claude
- **Users:** single user (decided against shareable for v1)
- **Show the interpretation:** every tool response includes the filters, definitions, and season/game scope it applied, so the interpretation comes from the server, not from Claude's description of it
- **No verdicts:** answers what's measurable; says when a question needs judgment the tool can't make yet
- **Unanswerable questions:** the tool says what it would need (stat, data source, capability) and the question is logged as a roadmap item
- **Glossary:** a markdown file of stat/term definitions in the project directory next to the code, tracked with version control, and exposed by the server so answers use my definitions. Fuzzy terms (e.g. "elite QB") get defined only once I actually use them; undefined terms get a provisional definition that goes to the review queue.
- **Feedback loop:** a feedback tool on the server. When I tell Claude an answer is wrong, it logs the item (question, answer, failure category, correction) to a review queue. I approve every change. Approved items update the test set, glossary, and worked examples.
  - Failure categories: misread the question; wrong definition; stale context; data problem; can't answer

### Out of scope for v1
- Live / in-game data
- Sharing with other users
- Advanced or modeled metrics
- Charting data (drops, catchable balls, pressures)
- Transactions, contracts, draft capital
- Injuries and roster status
- Predictions and betting
- A standalone UI

### Deferred
- **v1.5:** AI triage of the feedback queue, once v1 has collected ~30 triaged items. Per principle 5, triage runs **in a Claude session** that reads the queue through the MCP server and proposes fixes; it is not an automated server-side process. Claude proposes, I approve.
- **v2:** published advanced metrics (efficiency, win probability)
- **v3:** evidence building — breaking judgment questions into measurable parts
- **v4+:** new data sources (charting, transactions, injuries)
- Judgment and predictive questions (e.g. Caleb vs. Jordan Love, Bears 4th-down calls, drops/retargets) form the roadmap; see `brainstorm-summary.md` for the starter question list mapped to versions.

---

## 5. Open items
- Further concerns to add to the motivation section (TBD)
- Tool design: a general query tool, curated tools (player stats, team splits, leaderboards), or both — decide at spec time
- Testing approach: deterministic tests of server tools, plus an end-to-end question set run through Claude (with tolerance for non-repeatable responses)
- Product success criteria
- Operating constraints: response time, behavior when the nightly refresh fails, hosting cost ceiling
- Specs and the task breakdown for each component in §2
