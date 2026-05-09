# MiroShark setup for `soon`

> Our fork-specific configuration. The upstream `README.md` next to this file is preserved as-is for reference.
>
> See also: `docs/SLACK-BRIDGE.md`, `docs/PIPELINE.md` stage 5, `research/codebase-deep-dive.md`.

---

## Prereqs

- **Docker Desktop** running (Windows or macOS)
- An **OpenRouter API key** (`sk-or-v1-...` from <https://openrouter.ai/keys>)
- ~5 GB free disk for Docker images + Neo4j volume

We do *not* run Ollama. Cloud-only via OpenRouter.

---

## Boot sequence

```powershell
# from soon/services/miroshark/
copy .env.example .env
notepad .env       # paste OpenRouter key into the slots noted below
docker compose up -d
```

That brings up three containers:
- `miroshark-neo4j` — graph database on `:7474` (browser) and `:7687` (bolt)
- `miroshark-ollama` — declared in compose but unused; we don't pull any models
- `miroshark` — the MiroShark backend on `:5001`, frontend on `:3000` (we ignore the frontend)

First boot pulls images (~2 min) and starts Neo4j (~30 s). Subsequent boots are ~5 s.

Verify with:

```powershell
curl http://localhost:5001/api/docs
```

Should return MiroShark's OpenAPI spec.

---

## `.env` — recommended configuration

Edit `services/miroshark/.env` (start from their `.env.example`):

```env
# ── LLM (OpenRouter, single key fills all five slots) ────────────────────────
LLM_PROVIDER=openai
LLM_API_KEY=<your OpenRouter key>
LLM_BASE_URL=https://openrouter.ai/api/v1
LLM_MODEL_NAME=xiaomi/mimo-v2-flash       # bulk agent loop — cheap, fast

SMART_PROVIDER=openai
SMART_API_KEY=<same OpenRouter key>
SMART_BASE_URL=https://openrouter.ai/api/v1
SMART_MODEL_NAME=x-ai/grok-4.1-fast       # smart calls (reports, ontology)

NER_API_KEY=<same OpenRouter key>
NER_BASE_URL=https://openrouter.ai/api/v1
NER_MODEL_NAME=x-ai/grok-4.1-fast         # entity extraction needs reliable JSON

WONDERWALL_MODEL_NAME=xiaomi/mimo-v2-flash   # the agent loop

OPENAI_API_KEY=<same OpenRouter key>
OPENAI_API_BASE_URL=https://openrouter.ai/api/v1

EMBEDDING_PROVIDER=openai
EMBEDDING_MODEL=openai/text-embedding-3-large
EMBEDDING_BASE_URL=https://openrouter.ai/api
EMBEDDING_API_KEY=<same OpenRouter key>
EMBEDDING_DIMENSIONS=768

WEB_SEARCH_MODEL=x-ai/grok-4.1-fast:online   # Perplexity-style live web for persona enrichment

# ── Features we KEEP (intentionally) ─────────────────────────────────────────
RERANKER_ENABLED=true                       # one-time ~1 GB download, then free
ENTITY_RESOLUTION_ENABLED=true
ENTITY_RESOLUTION_USE_LLM=true
COMMUNITY_MIN_SIZE=3
COMMUNITY_MAX_COUNT=30
REASONING_TRACE_ENABLED=true
LLM_DISABLE_REASONING=false                 # CoT on for smart calls (quality lever)
LLM_PROMPT_CACHING_ENABLED=true             # auto-no-ops on non-Anthropic
MCP_AGENT_TOOLS_ENABLED=true                # 1–2 analyst personas with web tools
GRAPH_SEARCH_ENABLED=true
GRAPH_SEARCH_HOPS=1
GRAPH_SEARCH_SEEDS=5

# ── Features we SKIP (intentionally) ─────────────────────────────────────────
CONTRADICTION_DETECTION_ENABLED=false       # rarely fires for single-doc input
ORACLE_SEED_ENABLED=false                   # niche financial/regulatory feeds

# ── Neo4j (Docker service hostnames) ─────────────────────────────────────────
NEO4J_URI=bolt://neo4j:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=miroshark

# ── Embedding batch ──────────────────────────────────────────────────────────
EMBEDDING_BATCH_SIZE=128
```

The same `OPENROUTER_API_KEY` from our root `.env.local` works here — paste it into all six slots above.

---

## Pre-demo: warm a run

The night before the demo, cache a full simulation of Red Bull Caffeinated Gum so the live demo only triggers `/start`. From the soon repo root:

```powershell
# Boot MiroShark + our Next.js app
docker compose -f services/miroshark/docker-compose.yml up -d
npm run dev

# In a second terminal, fire one full pipeline run
curl -X POST http://localhost:3000/api/pipeline `
  -F companyName="Red Bull" `
  -F idea="Caffeinated Chewing Gum" `
  -F documents=@./redbull-annual-report.pdf
```

The graph build, persona generation, and one full simulation will execute. After it completes, MiroShark's project + simulation state are persisted to the Neo4j volume and `services/miroshark/backend/uploads/`. Demo day, you start a fresh run that reuses the cached graph.

---

## Wall-clock expectations (per run, after pre-warm)

| Phase                                | Time         |
|--------------------------------------|--------------|
| Brand brief summarization (our backend) | ~5 s         |
| Ontology generate                    | ~10 s        |
| Graph build (1 chunk, post-summarize) | ~15 s        |
| Profile generation (~30 personas)    | ~30 s        |
| Prepare                              | ~10 s        |
| Agent loop (5 rounds, parallel)      | ~60–90 s     |
| Slack streaming                      | concurrent   |
| Report generation                    | ~30 s post-loop |
| **Total**                            | **~3 min**   |

Demo-stage acceptable. Cost per run ≈ $0.50–1.00.

---

## Health checks

```powershell
# MiroShark up?
curl http://localhost:5001/api/docs

# Neo4j up?
docker logs miroshark-neo4j --tail 20

# Reranker downloaded?
docker exec miroshark ls -lh /root/.cache/torch/sentence_transformers
```

---

## Caveats

- **First simulation per session is slower** by ~30 s — reranker model loads into memory on first call.
- **OpenRouter rate limits** can throttle bulk NER if your account tier is low. Tier 2+ recommended.
- **`enable_twitter: false, enable_polymarket: false`** in `/api/simulation/create` — set explicitly in our client. Don't run `parallel` mode unless you also want Twitter/Polymarket data.
- **MCP tools** are wired by configuring `tools_enabled: true` on specific persona profiles + `MCP_AGENT_TOOLS_ENABLED=true` in `.env`. See MiroShark's `config/mcp_servers.yaml` for tool registration.
