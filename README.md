# MiroFish-Offline

A fork of [MiroFish](https://github.com/nikmcfly/MiroFish-Offline) — a multi-agent social media simulation platform with GraphRAG, adapted for fully offline/local LLM inference on a dual-machine AI cluster.

## Overview

MiroFish-Offline simulates public debates on social media using AI agents powered by local LLMs. Starting from a source document (a "reality seed"), the system:

1. **Builds a Knowledge Graph** (GraphRAG) using Neo4j — extracting entities, relationships, and ontologies from the input document
2. **Generates Agent Personas** — each agent gets a rich, LLM-generated personality profile based on the knowledge graph
3. **Runs Parallel Simulations** on two platforms:
   - **Info Plaza** (Twitter-like) — short posts, likes, reposts, follows
   - **Topic Community** (Reddit-like) — longer posts, comments, upvotes, debates
4. **Generates a Prediction Report** — the system interviews agents and produces a structured analytical report with future predictions

## Hardware Setup (Dual-Machine Cluster)

| Machine | Hardware | Role |
|---|---|---|
| **AI-STATION-SH** | GMKtek AM18, Ryzen AI MAX 395, 128GB RAM | Main LLM (ontology, GraphRAG, report, Reddit) |
| **AI-STATION-R9** | MINISFORUM AI X1 PRO, Ryzen AI 9, 64GB RAM | Boost LLM (Twitter simulation) + Embeddings + Zep stack |

### Services
- **SH**: LM Studio (port 1234) + Flask backend (port 5001) + Vite frontend (port 3000)
- **R9**: LM Studio (port 1234) + Docker stack (Zep CE, Neo4j, pgvector, Graphiti)

## Recommended Models

| Role | Model | Speed |
|---|---|---|
| Main LLM (SH) | `qwen/qwen3.5-35b-a3b` | ~68-70 t/s |
| Boost LLM (R9) | `nemotron-cascade-2-30b-a3b-i1` | ~35 t/s |
| Embeddings (R9) | `text-embedding-nomic-embed-text-v1.5` | — |

Both models are MoE (Mixture of Experts) architectures, providing high quality at fast inference speeds.

## Environment Configuration (.env)

```env
# Main LLM (SH) — used for ontology, GraphRAG, agent profiles, report
LLM_API_KEY=lm-studio
LLM_BASE_URL=http://<SH_IP>:1234/v1
LLM_MODEL_NAME=qwen/qwen3.5-35b-a3b

# Boost LLM (R9) — used for Twitter/Info Plaza simulation
LLM_BOOST_API_KEY=lm-studio
LLM_BOOST_BASE_URL=http://<R9_IP>:1234/v1
LLM_BOOST_MODEL_NAME=nemotron-cascade-2-30b-a3b-i1

# Embeddings (R9)
EMBEDDING_MODEL=text-embedding-nomic-embed-text-v1.5
EMBEDDING_BASE_URL=http://<R9_IP>:1234/v1

# Neo4j (R9)
NEO4J_URI=bolt://<R9_IP>:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password

# CAMEL/OASIS — MUST point to SH, not R9!
# If this points to R9, CAMEL will trigger unwanted model loading on R9 during agent preparation
OPENAI_API_KEY=lm-studio
OPENAI_API_BASE_URL=http://<SH_IP>:1234/v1

# CAMEL model timeout — increase for slower models
MODEL_TIMEOUT=600
```

> ⚠️ **Critical**: `OPENAI_API_BASE_URL` must point to SH. If it points to R9, CAMEL/OASIS will send requests to R9 during agent generation and force-load the default model, ignoring `LLM_BOOST_MODEL_NAME`.

## Dual-LLM Routing

The simulation splits workload across two machines:

| Platform | LLM | Machine | Reason |
|---|---|---|---|
| Topic Community (Reddit) | Main LLM | SH | More actions, longer posts — needs more power |
| Info Plaza (Twitter) | Boost LLM | R9 | Short posts, faster rounds |
| Agent preparation | Main LLM | SH | Always on SH regardless of boost config |
| Report generation | Main LLM | SH | Always on SH |

## Fixes Applied (vs Original MiroFish)

### Bug Fixes
| File | Fix |
|---|---|
| `app/utils/llm_client.py` | Removed `response_format=None` (caused `{"type": null}` errors with LM Studio) |
| `app/services/oasis_profile_generator.py` | Same fix — was causing all agent persona generation to fail |
| `app/services/simulation_config_generator.py` | Same fix |
| `app/storage/embedding_service.py` | Fixed endpoint `/api/embed` → `/embeddings`, added OpenAI format support |

### Performance & Stability
| File | Fix |
|---|---|
| `app/services/simulation_ipc.py` | IPC timeout 120s → 600s |
| `app/services/simulation_runner.py` | Batch chunk size 4 → 2 (prevents interview timeouts), global timeout 180s → 600s |
| `app/services/graph_tools.py` | Hardcoded timeout 180s → 600s |
| `scripts/run_parallel_simulation.py` | Added `load_dotenv(override=True)` to ensure subprocess reads updated .env values |

### Dual-LLM Support
| File | Fix |
|---|---|
| `scripts/run_parallel_simulation.py` | Inverted `use_boost` routing: Reddit→SH (False), Twitter→R9 (True) |
| `.env` | Added `LLM_BOOST_*` variables for R9 model, `MODEL_TIMEOUT=600` for CAMEL |

## Quick Start

### R9 — Start Docker stack
```bash
cd /path/to/zep-storage
docker compose up -d
```

### SH — Start backend + frontend
```bash
# Terminal 1 - Backend
cd /path/to/MiroFish-Offline
source venv/bin/activate
cd backend && python run.py

# Terminal 2 - Frontend
cd /path/to/MiroFish-Offline/frontend
npm run dev
```

### Cluster health check
```bash
bash test_cluster.sh
```

## Tips for Best Results

- **Source document**: Include explicit named entities (people, organizations) with roles and relationships. The GraphRAG extracts these as simulation agents.
- **Prompt**: Explicitly list character names and their roles — this generates richer, more distinct agent personalities.
- **Context window**: 128k is sufficient for most simulations and faster than 262k.
- **Custom rounds**: Use the Custom mode (15-30 rounds) for quick tests before running full 48-72 round simulations.

## Architecture

```
Frontend (Vite) ──► Backend (Flask)
                        │
                        ├── GraphRAG Builder ──► Neo4j (R9)
                        ├── Agent Persona Generator ──► LM Studio SH
                        ├── Simulation Runner
                        │       ├── run_parallel_simulation.py
                        │       │       ├── Twitter ──► LM Studio R9 (Boost)
                        │       │       └── Reddit  ──► LM Studio SH (Main)
                        │       └── IPC (filesystem-based commands/responses)
                        ├── Report Agent ──► LM Studio SH
                        └── Embedding Service ──► LM Studio R9
```

## Original Project

This is a fork of [MiroFish-Offline](https://github.com/nikmcfly/MiroFish-Offline).  
All original credits go to the original authors.
