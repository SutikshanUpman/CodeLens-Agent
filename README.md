# CodeLens Agent 🔍

> AI-powered code review agent that thinks like a senior engineer.
> Paste a GitHub PR URL → get line-specific, pattern-aware review comments with engineering guidelines cited as sources.

> ⚠️ **Status: In Development** — architecture planned, build starting soon.

---

## Idea

Most AI code review tools give generic suggestions. CodeLens Agent is different:

- Parses PR diffs at **function level**, not just line-by-line
- Detects specific anti-patterns — N+1 loops, deep nesting, duplicate logic, missing edge cases
- Retrieves relevant engineering guidelines (Google Style Guide, Effective Java, Clean Code) via RAG and **cites the source** in every comment
- Streams review comments in real time via SSE
- Persists all reviews in PostgreSQL — queryable by repo, author, pattern type

---

## Planned Architecture

```
PR URL
  │
  ▼
FastAPI Backend (POST /review → SSE stream)
  │
  ▼
LangGraph Agent
  [fetch_pr] → [parse_diff] → [pattern_scan] → [rag_retrieval] → [llm_review]
  │                                                    │
  ▼                                                    ▼
GitHub API                                        ChromaDB
(diffs, file trees)                          (engineering guidelines
                                              BM25 + dense hybrid)
  │
  ▼
PostgreSQL
(review history, pattern stats)
```

---

## Tech Stack

| Layer | Tool |
|-------|------|
| Agent | LangGraph |
| LLM | Groq — `llama-3.3-70b-versatile` (free tier) |
| Backend | FastAPI |
| Database | PostgreSQL + SQLAlchemy async |
| Vector Store | ChromaDB |
| Retrieval | BM25 + dense embeddings (hybrid, RRF) |
| GitHub | PyGithub |
| Streaming | Server-Sent Events (SSE) |
| Eval | RAGAS |
| Deploy | Docker + Docker Compose |

---

## Project Structure

```
codelens-agent/
├── backend/
│   ├── main.py
│   ├── routers/        # review.py, health.py
│   ├── agent/          # state.py, graph.py, nodes.py, tools.py
│   ├── rag/            # ingest.py, retriever.py, eval.py
│   ├── db/             # models.py, session.py
│   └── core/           # config.py, github.py
├── eval/
│   └── golden_dataset.json
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── .env.example
```

---

## Roadmap

- [ ] FastAPI backend + PostgreSQL + Docker Compose
- [ ] LangGraph agent — fetch → parse → pattern scan
- [ ] RAG pipeline — ChromaDB + BM25 hybrid retrieval
- [ ] Groq LLM integration + SSE streaming
- [ ] RAGAS evaluation on golden dataset
- [ ] GitHub webhook auto-trigger
- [ ] Web UI

---

## Author

**Sutikshan Upman** — B.Tech CSE, IIIT Nagpur
[LinkedIn](https://linkedin.com/in/sutikshanupman) · [GitHub](https://github.com/SutikshanUpman)
