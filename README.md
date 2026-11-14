# NFL Query Tool

Ask NFL questions in plain English from Claude, and get answers backed by real play-by-play data. Each answer shows exactly how the question was interpreted.

> **Status: planning.** Requirements and platform decisions are drafted, and no code is written yet. The first milestone is a walking skeleton: one season of data, one MCP tool, and Claude answering one real question from a phone.

## What it is

There's no chat app here. The project is a **data model with an MCP server on top**, connected to Claude as a custom connector:

- **Claude is the interface.** It handles the conversation, follow-up questions, and turning plain English into tool calls.
- **The server is the contract.** Claude sees only what the server exposes, so the data model, tool design, and definitions have to be good enough that answers are right without guessing.
- **The server never calls an LLM.** It serves data, definitions, and a feedback queue, and all reasoning happens in the Claude session.

### v1 scope

- Play-by-play era (1999 to present), current through the previous night
- Counting and rate stats only, with no advanced or modeled metrics yet
- **Shows its interpretation:** every response includes the filters, definitions, and season/game scope applied, plus the data's as-of date
- **No verdicts:** it answers what's measurable and says when a question needs judgment it can't make yet
- **Governed glossary:** fuzzy terms like "elite QB" or "terrible game" get explicit, versioned definitions
- **Feedback loop:** wrong answers get logged with a failure category and a correction. Approved fixes become test cases, glossary updates, and worked examples.

## Architecture

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

| Layer | Choice |
|---|---|
| Data source | [nflverse](https://github.com/nflverse) play-by-play and player stats |
| Database | Postgres 16+, dimensional model |
| Orchestration | Airflow (LocalExecutor), nightly watermark-based refresh |
| MCP server | Python, official MCP SDK, Streamable HTTP |
| Hosting | One ARM EC2 instance running Docker Compose, budgeted under $20/month |
| Infrastructure as code | Pulumi (Python) for AWS and Cloudflare DNS |
| CI/CD | GitHub Actions |

## Why it's built this way

The guiding rule is **robust, not complicated**. Every piece is standard practice at a real company (reproducible, tested, deployed automatically, observable), but sized to the problem: one user, a few GB of data, one nightly job. Heavier options that were considered and rejected (Kubernetes, serverless, managed databases, and others) are written up along with the reasons, in [docs/decisions.md](docs/decisions.md).

The project also practices AI-driven development as a disciplined pipeline: requirements, then specs, then bounded tasks that an agent carries out and I check.

## Repository layout

| Path | Contents |
|---|---|
| [docs/](docs/) | Requirements, decisions, specs, and runbooks |
| `infra/` | Pulumi program *(planned)* |
| `pipeline/` | Airflow DAGs and transforms *(planned)* |
| `server/` | MCP server *(planned)* |
| `glossary/` | Term definitions, version-controlled *(planned)* |

Start with [docs/requirements.md](docs/requirements.md) and [docs/decisions.md](docs/decisions.md). The original brainstorm is in [brainstorm-summary.md](brainstorm-summary.md).

## Data

Data comes from [nflverse](https://github.com/nflverse). Thanks to its maintainers. Raw data files aren't committed to this repo, because the pipeline pulls them.

## License

The code is released under the [MIT License](LICENSE). Data from nflverse is subject to nflverse's own licenses.
