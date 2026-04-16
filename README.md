<p align="center">
  <img src="logo.png" alt="Command Claw Memory" height="88">
</p>

<h1 align="center">Command Claw Memory</h1>

<p align="center">
  <strong>The recall machinery for CommandClaw agents.</strong><br>
  <em>FastAPI service that owns the wiki repo, validates writes, indexes with LanceDB + BM25, runs distillation, and exposes hybrid retrieval.</em><br>
  <sub>Discipline layer for the LLM Wiki. Built for cheap-model safety.</sub>
</p>

---

> [!WARNING]
> **🚧 Stub Repository** — This repo is scaffolded but not yet implemented. The service, schema, distillation worker, and integration shim will land per [PLAN.md](https://github.com/FnSK4R17s/commandclaw/blob/main/guiding_docs/PLAN.md) in the main `commandclaw` repo. Watch this space.
>
> 💬 **Have feedback or found a bug?** Reach out at [**@_Shikh4r_** on X](https://x.com/_Shikh4r_)

## What this is

`commandclaw-memory` is the FastAPI microservice that gives CommandClaw agents a real memory layer. It owns a working clone of [commandclaw-wiki](https://github.com/FnSK4R17s/commandclaw-wiki), validates every write against a Pydantic schema, indexes content with LanceDB (vectors) + Tantivy (BM25), runs an asynchronous distillation worker, and exposes hybrid retrieval over a REST API.

It is the **discipline layer**. The wiki content lives in git (auditable, portable, human-readable). The service is what makes that content reliable to write and fast to search — even when the primary agent is a cheap or weak model that can't be trusted to self-maintain a knowledge base.

The architectural rationale, design decisions, and full ten-dimension specification live in the [memory service whitepaper](https://github.com/FnSK4R17s/commandclaw/blob/main/whitepaper-output/commandclaw-memory-service/whitepaper/commandclaw-memory-service-whitepaper.md) and the implementation [PLAN.md](https://github.com/FnSK4R17s/commandclaw/blob/main/guiding_docs/PLAN.md).

## What it does

| Capability | How |
|---|---|
| **Owns the wiki repo** | Clones `commandclaw-wiki` on startup. Single-writer. Sync git commit on every successful write, async push on a configurable interval. |
| **Schema enforcement** | Pydantic models per page type. Cross-reference validation. Malformed writes rejected at the REST boundary. |
| **Hybrid retrieval** | LanceDB for vector search (BGE-small-en-v1.5), Tantivy for BM25, RRF fusion. Sub-60ms warm retrieval target on loopback. |
| **Distillation worker** | Async background asyncio task. Configurable LLM (default gpt-4o-mini) or rule-based fallback. Off the hot path. |
| **Auth and RBAC** | Phantom token + HMAC + Cerbos, identical to the [commandclaw-mcp](https://github.com/FnSK4R17s/commandclaw-mcp) gateway pattern. |
| **Graceful degradation** | Circuit breaker → hot-tier fallback → offline write buffer. Agents stay responsive when the service is down. |

## Architecture in one diagram

```
   Agent (LangGraph)                   commandclaw-memory (FastAPI :8284)
   ├─ wiki_search ──────HTTP─────►     ├─ Schema validator
   ├─ wiki_read         (loopback)     ├─ LanceDB store (vector)
   ├─ wiki_ingest                      ├─ Tantivy store (BM25)
   ├─ wiki_lint                        ├─ Distill worker (async)
   └─ MemoryServiceStore               ├─ Cerbos client
      (BaseStore shim)                 │
                                       └─ Wiki clone ──git commit──►  commandclaw-wiki
                                                       (sync write,    (separate repo,
                                                        async push)     service-managed)
```

The agent never touches the wiki repo directly. The memory service never touches per-agent vaults. Clean boundaries on both sides.

## REST API surface (planned)

```
POST   /wiki/pages                  Create a wiki page (validated, committed, indexed)
GET    /wiki/pages/{type}/{slug}    Read a page
PATCH  /wiki/pages/{type}/{slug}    Update a page
DELETE /wiki/pages/{type}/{slug}    Delete a page (admin/owner only)

GET    /wiki/search                 Hybrid BM25 + vector + RRF
GET    /wiki/index                  index.md as structured JSON

POST   /ingest/source               Trigger distillation of a raw source
POST   /distill/session             Trigger distillation of a session transcript
POST   /lint                        Health-check the wiki (orphans, contradictions)
GET    /jobs/{job_id}               Poll a distillation job

POST   /admin/reindex               Full rebuild from the wiki repo
POST   /admin/wiki/pull             Force-fetch + reset to remote
POST   /admin/wiki/push             Force-push pending commits
POST   /admin/bootstrap             Seed-from-vaults bootstrap
POST   /admin/enroll                Issue agent credentials

GET    /health, /ready, /metrics    Operational
```

Full spec: [PLAN.md §4](https://github.com/FnSK4R17s/commandclaw/blob/main/guiding_docs/PLAN.md).

## Quick start (once implemented)

```bash
# 1. Clone this repo
gh repo clone FnSK4R17s/commandclaw-memory
cd commandclaw-memory

# 2. Set the wiki remote
cp .env.example .env
# Edit .env — set COMMANDCLAW_MEMORY_WIKI_REMOTE to your wiki repo URL

# 3. Start the service
docker compose up -d

# 4. Service is at http://localhost:8284
curl http://localhost:8284/health
```

## Related repos

| Repo | Purpose |
|------|---------|
| [commandclaw](https://github.com/FnSK4R17s/commandclaw) | Agent runtime, Telegram I/O, tracing |
| [commandclaw-vault](https://github.com/FnSK4R17s/commandclaw-vault) | Vault template — cloned per agent workspace |
| [commandclaw-mcp](https://github.com/FnSK4R17s/commandclaw-mcp) | MCP gateway — credential proxy with rotating keys |
| [commandclaw-gateway](https://github.com/FnSK4R17s/commandclaw-gateway) | LLM routing layer — provider credentials, virtual keys, budgets, rate limits, multi-provider fallback |
| [commandclaw-skills](https://github.com/FnSK4R17s/commandclaw-skills) | Skills library — `npx skills add FnSK4R17s/commandclaw-skills` |
| [commandclaw-wiki](https://github.com/FnSK4R17s/commandclaw-wiki) | LLM Wiki — persistent, compounding knowledge base per agent (Karpathy pattern) |
| [commandclaw-observe](https://github.com/FnSK4R17s/commandclaw-observe) | Self-hosted observability — Langfuse tracing + Prometheus + Grafana, one compose |
| [openclaw](https://github.com/FnSK4R17s/openclaw) | Original personal AI assistant — predecessor to CommandClaw |

## License

MIT
