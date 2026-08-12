# RecruiterEye

> See your GitHub the way a recruiter does — before they do.
> Paste a **GitHub profile** or a **repo/PR URL** → get an HR-lens review of your code, commit history, and project structure. **100% free.**

> ⚠️ **Status: Idea / early planning** — this repo is a working notebook while the architecture takes shape, not a build yet.

---

## Why

Students build projects for their resumes but rarely see what a recruiter actually notices: whether the commit history looks like real work or a one-shot upload, whether the code is actually clean, and whether the repo is organized like a real project. CodeLens Agent automates that first pass, explains *why* something's flagged, and tells the student what to fix — a free second pair of eyes before they apply.

---

## What It Reviews

Four agents, each covering one thing a reviewer would look at:

1. **Commit History** — does the history look incremental and real, or dumped in one go?
2. **Code Quality** — anti-patterns, style, cited against real guidelines (Clean Code, Google Style Guide, etc.)
3. **Structure** — is the repo organized like a real project?
4. **Authenticity** — takes the top 2–3 projects and checks their commit timing/diff sizes to flag "written elsewhere and pasted in" vs. built over time.

A **free chatbot** (Llama via Groq) sits on top so students can ask follow-up questions about their review.

---

## Rough Data Flow

```
GitHub URL (profile or repo/PR)
   → fetch data (GitHub API)
   → parse diffs (function-level)
   → run the 4 review agents (RAG-backed for citations)
   → authenticity check on top projects
   → LLM synthesizes one review (streamed via SSE)
   → saved to Postgres
   → chatbot answers follow-ups on the result
```

---

## LangGraph Structure

```
START
  │
  ▼
fetch_github        # pull profile/repo data via GitHub API
  │
  ▼
parse_diff           # function-level diff parsing
  │
  ├──► commit_history_reviewer ──┐
  ├──► code_quality_reviewer ────┼──► (parallel branch, fan-out)
  └──► structure_reviewer ───────┘
  │
  ▼
rag_retrieval         # code_quality_reviewer calls this for cited guidelines
  │
  ▼
select_top_projects   # pick 2-3 projects flagged by the branch above (fan-in)
  │
  ▼
authenticity_evaluator  # commit timing/diff-size check on those projects
  │
  ▼
llm_review             # synthesize everything into one review, stream via SSE
  │
  ▼
END
```

**State object** (rough, will refine): `github_url`, `mode` (profile/repo), `raw_data`, `parsed_diffs`, `commit_findings`, `quality_findings`, `structure_findings`, `top_projects`, `authenticity_findings`, `final_review`.

Notes to self:
- `commit_history_reviewer`, `code_quality_reviewer`, `structure_reviewer` can run as parallel nodes off the same state since they read different parts of it — LangGraph fan-out/fan-in.
- `rag_retrieval` is really a tool the `code_quality_reviewer` node calls, not a separate graph node — might model it as a subgraph or just a tool call inside that node.
- `select_top_projects` is the fan-in point — needs findings from all 3 review nodes before it can run.
- Chatbot is a separate graph/entry point, not part of this main flow — it reads `final_review` + RAG index as context.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Agent Orchestration | LangGraph |
| LLM | Groq — `llama-3.3-70b-versatile` (free tier) |
| Backend | FastAPI |
| Database | PostgreSQL + SQLAlchemy async |
| Vector Store | ChromaDB |
| Retrieval | BM25 + dense embeddings (hybrid, RRF) |
| GitHub Data | PyGithub (API) + lightweight scraping fallback |
| Streaming | Server-Sent Events (SSE) |
| Eval | RAGAS |
| Deploy | Docker + Docker Compose |

---

## Project Structure

```
codelens-agent/
├── backend/
│   ├── main.py
│   ├── routers/     # review.py, chat.py, health.py
│   ├── agent/       # state.py, graph.py, nodes.py, tools.py
│   ├── rag/         # ingest.py, retriever.py, eval.py
│   ├── github/      # fetch.py
│   ├── db/          # models.py, session.py
│   └── core/        # config.py
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
- [ ] GitHub fetch layer (API)
- [ ] LangGraph agent — fetch → parse → review (commit history / code quality / structure)
- [ ] RAG pipeline — ChromaDB + BM25 hybrid retrieval, cited guidelines
- [ ] Authenticity check on top projects
- [ ] Groq LLM + SSE streaming
- [ ] RAGAS evaluation
- [ ] Free chatbot for follow-up Q&A
- [ ] Web UI

---

## Author

**Sutikshan Upman** — B.Tech CSE, IIIT Nagpur
[LinkedIn](https://linkedin.com/in/sutikshanupman) · [GitHub](https://github.com/SutikshanUpman)
