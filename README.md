# CodeLens Agent 🔍

> **An AI-powered code review agent that thinks like a senior engineer.**
> 
> Paste a GitHub PR URL → get back line-specific, pattern-aware review comments covering performance pitfalls, duplication, missing edge cases, and style violations — with sources cited from real engineering guidelines.

---

## What It Does

Most AI code review tools give you generic suggestions. CodeLens Agent is different — it:

- **Parses the PR diff at the function level**, not just line-by-line
- **Detects specific anti-patterns** (N+1 loops, unnecessary nested iteration, null pointer risks, duplicate logic across files)
- **Retrieves relevant engineering guidelines** via RAG (Google Style Guide, Effective Java, clean code principles) and **cites the source** alongside each comment
- **Streams the review back** via SSE — you see comments appear in real time as the agent reasons through each changed file
- **Stores every review in PostgreSQL** — you can query review history per repo, per author, per pattern type

---

## Architecture

```
PR URL Input
     │
     ▼
┌─────────────────────────────────────────────────────┐
│                   FastAPI Backend                    │
│  POST /review  →  SSE stream  ←  GET /history       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              LangGraph Agent (StateGraph)            │
│                                                      │
│  [fetch_pr] → [parse_diff] → [pattern_scan]         │
│       → [rag_retrieval] → [llm_review]              │
│       → [format_output]                             │
└──────────┬───────────────────────┬──────────────────┘
           │                       │
           ▼                       ▼
  ┌────────────────┐    ┌─────────────────────┐
  │   GitHub API   │    │  ChromaDB (RAG)      │
  │  (PR diffs,    │    │  Engineering guides  │
  │   file trees)  │    │  BM25 + dense hybrid │
  └────────────────┘    └─────────────────────┘
           │
           ▼
  ┌────────────────┐
  │  PostgreSQL    │
  │  Review logs   │
  │  Pattern stats │
  └────────────────┘
```

### Agent Nodes (LangGraph)

| Node | What it does |
|------|-------------|
| `fetch_pr` | Calls GitHub API, fetches diff + file metadata |
| `parse_diff` | Splits diff into function-level chunks with context |
| `pattern_scan` | Rule-based pre-scan: flags N+1 loops, deep nesting, duplication candidates |
| `rag_retrieval` | Hybrid BM25 + dense retrieval from engineering guidelines corpus |
| `llm_review` | Groq `llama-3.3-70b-versatile` generates line-specific comments with citations |
| `format_output` | Structures output into per-file, per-line review objects |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Agent orchestration | LangGraph |
| LLM | Groq API — `llama-3.3-70b-versatile` (free tier) |
| Backend | FastAPI + async SQLAlchemy |
| Database | PostgreSQL |
| Vector store | ChromaDB |
| Retrieval | Hybrid: BM25 (rank-bm25) + dense embeddings (sentence-transformers) |
| GitHub integration | PyGithub |
| Streaming | Server-Sent Events (SSE) |
| Evaluation | RAGAS on 20-PR golden dataset |
| Deployment | Docker + Docker Compose |

---

## Project Structure

```
codelens-agent/
├── backend/
│   ├── main.py                  # FastAPI app entry point
│   ├── routers/
│   │   ├── review.py            # POST /review, GET /history
│   │   └── health.py            # GET /health
│   ├── agent/
│   │   ├── state.py             # LangGraph state schema (TypedDict)
│   │   ├── graph.py             # StateGraph definition + compilation
│   │   ├── nodes.py             # All 6 agent nodes
│   │   └── tools.py             # GitHub fetch tool, pattern scan tool
│   ├── rag/
│   │   ├── ingest.py            # Load guides → chunk → embed → store
│   │   ├── retriever.py         # Hybrid retrieval + reranking
│   │   └── eval.py              # RAGAS evaluation suite
│   ├── db/
│   │   ├── models.py            # SQLAlchemy: Review, FileReview, Comment
│   │   └── session.py           # Async DB session
│   └── core/
│       ├── config.py            # Pydantic settings (env vars)
│       └── github.py            # GitHub API client wrapper
├── eval/
│   └── golden_dataset.json      # 20 PRs with expected review patterns
├── docs/
│   └── architecture.md          # Extended architecture notes
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── .env.example
└── .gitignore
```

---

## Quickstart

### Prerequisites
- Docker + Docker Compose
- [Groq API key](https://console.groq.com) (free, no credit card)
- [GitHub Personal Access Token](https://github.com/settings/tokens) (read:repo scope)

### Setup

```bash
# 1. Clone the repo
git clone https://github.com/SutikshanUpman/codelens-agent.git
cd codelens-agent

# 2. Configure environment
cp .env.example .env
# Fill in GROQ_API_KEY and GITHUB_TOKEN in .env

# 3. Start everything
docker compose up --build

# 4. Ingest the engineering guidelines corpus
docker compose exec app python -m backend.rag.ingest

# 5. API is live at http://localhost:8000
# Docs at http://localhost:8000/docs
```

### Run a Review

```bash
curl -X POST http://localhost:8000/review \
  -H "Content-Type: application/json" \
  -d '{"pr_url": "https://github.com/owner/repo/pull/42"}'
```

**Example response (streamed):**

```json
{
  "file": "src/UserService.java",
  "line": 47,
  "pattern": "N+1 Query",
  "severity": "high",
  "comment": "getUserOrders() is called inside a loop over users — this will fire one DB query per user. Batch the fetch outside the loop using findAllByUserIds().",
  "source": "Effective Java, Item 50 — Avoid unnecessary object creation in hot paths"
}
```

---

## Evaluation

RAGAS evaluation runs on a golden dataset of 20 real PRs with manually annotated expected review patterns.

```bash
docker compose exec app python -m backend.rag.eval
```

| Metric | Score |
|--------|-------|
| Faithfulness | 0.81 |
| Answer Relevance | 0.78 |
| Context Precision | 0.74 |

*Scores updated after each major retrieval change.*

---

## RAG Corpus

Engineering guidelines indexed in ChromaDB:

- Google Java Style Guide
- Effective Java (3rd ed.) — key items
- Clean Code principles (Robert Martin)
- OWASP Top 10 — security anti-patterns
- Python PEP 8 + PEP 20

Retrieval: **Hybrid BM25 + dense** with Reciprocal Rank Fusion — retrieve top 20, rerank to top 5 before feeding to LLM.

---

## Roadmap

- [x] Core LangGraph agent pipeline
- [x] FastAPI backend with SSE streaming
- [x] ChromaDB RAG with hybrid retrieval
- [x] PostgreSQL review persistence
- [x] Docker Compose deployment
- [x] RAGAS evaluation suite
- [ ] GitHub webhook auto-trigger (POST PR → auto-review)
- [ ] Web UI for review visualization
- [ ] PR comment posting back to GitHub

---

## Why This Project

Code review is one of the highest-leverage engineering activities — and one of the most context-dependent. Generic AI suggestions don't help because they don't know *why* a pattern is bad in *this* codebase. CodeLens Agent solves this by combining:

1. **Structured diff parsing** — working at function granularity, not just line diffs
2. **Pattern-first analysis** — rule-based pre-scan before LLM, so the LLM reasons about real candidates
3. **Grounded generation** — every comment cites a retrieved engineering principle, not hallucinated advice

Built as a learning project while preparing for AI engineer internships — every component written from scratch to understand the internals.

---

## Author

**Sutikshan Upman** — B.Tech CSE, IIIT Nagpur  
[LinkedIn](https://linkedin.com/in/sutikshanupman) · [GitHub](https://github.com/SutikshanUpman)
