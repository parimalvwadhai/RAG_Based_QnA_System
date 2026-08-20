# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A fully offline / air-gapped RAG Q&A system over corporate policy PDFs, built on Streamlit. Every model runs locally: Llama 3.2 through Ollama for generation, BGE embeddings + a BGE cross-encoder reranker through `sentence-transformers`, and ChromaDB for persistence. No cloud APIs are used anywhere, and changes should preserve that property.

## Commands

```powershell
pip install -r requirements.txt

# Ollama must be running locally and the model pulled before the app works
ollama pull llama3.2
ollama serve                      # generation.py hardcodes http://127.0.0.1:11434

streamlit run app.py              # main application

python evaluate_rag.py            # Ragas benchmark -> ragas_evaluation_results.csv
```

There is no test suite, linter, or build step. Ingestion is not a CLI command — it is triggered from the admin sidebar button inside the running Streamlit app.

First run of `VectorDatabase()` downloads `BAAI/bge-base-en-v1.5` and `BAAI/bge-reranker-base` from HuggingFace (~700MB combined) and caches them; after that the system is genuinely offline.

## Architecture

Request flow for a single user question in [app.py](app.py):

1. **Query rewriting** — `LLMGenerator.rewrite_query` feeds the last 3 chat turns to Llama 3.2 to condense a follow-up ("does it apply to contractors?") into a standalone search query.
2. **Hybrid retrieval** — `VectorDatabase.retrieve_context_hybrid` runs Chroma dense similarity and BM25 lexical search in parallel over `top_k * 3` candidates, merges them with hand-written Reciprocal Rank Fusion (`1/(60 + rank)`), then reranks the merged pool with the BGE cross-encoder and keeps `top_k`.
3. **Relevance gate** — if the best raw *dense distance* exceeds `1.15`, the LLM is bypassed entirely and the app answers `"I do not know."` This threshold is the anti-hallucination mechanism and is duplicated in [evaluate_rag.py](evaluate_rag.py); change both together.
4. **Grounded generation** — `LLMGenerator.generate_response` runs the strict prompt at `temperature=0.0`, returning the answer plus per-chunk citations (source filename, page, score).

Supporting layers:

- **[ingestion.py](ingestion.py)** — PyMuPDF page-by-page text extraction, PII redaction (`clean_pii` regexes for email, Indian mobile, Aadhaar, PAN) applied *before* chunking so redacted text is what gets embedded, then `RecursiveCharacterTextSplitter` at 500/50. Incremental: `data/ingestion_registry.json` stores each PDF's mtime, so re-running only processes new or modified files. Note the DB appends — re-ingesting a modified file duplicates its chunks rather than replacing them.
- **[database.py](database.py)** — owns both indexes. The BM25 index is rebuilt in memory from Chroma's stored documents on every `_load_db()`, so it is always derived from Chroma, never persisted separately.
- **[auth_service.py](auth_service.py)** — offline auth. Users live in `config/users.json` with SHA-256 hashed passwords; a super-admin, the registration invite code, and the allowed corporate email domain come from `st.secrets["admin"]` (`.streamlit/secrets.toml`) or `AUTH_ADMIN_EMAIL` / `AUTH_ADMIN_PASSWORD` / `AUTH_INVITE_CODE` / `AUTH_CORPORATE_DOMAIN` env vars. All are optional and silently disable their check when unset. Only `role == "admin"` sees the ingestion trigger; self-registration always creates `role: "user"`.
- **Caching** — `auth_service`, `vector_db`, and `llm_gen` are `@st.cache_resource` singletons; `doc_processor` is rebuilt each rerun. Editing a cached class requires restarting Streamlit, not just saving the file.

Audit events go to `admin_secure.log` (append-only, created at runtime).

## Known inconsistencies in the current tree

These are real and will bite; check them before assuming a bug is new.

- **Module layout is split.** [app.py](app.py) and [evaluate_rag.py](evaluate_rag.py) import `modules.auth_service`, `modules.ingestion`, `modules.database`, `modules.generation`, but only [modules/generation.py](modules/generation.py) actually lives there — the other three sit at the repo root, and `modules/` has no `__init__.py`. The app cannot start until this is resolved. Moving the three files into `modules/` (plus an empty `__init__.py`) matches the README's documented layout and both call sites.
- **Return-arity mismatch.** `retrieve_context_hybrid` returns a 3-tuple `(docs, latency, absolute_best_distance)`, but [app.py:230](app.py#L230) and [evaluate_rag.py:58](evaluate_rag.py#L58) both unpack two values.
- **Score semantics.** After reranking, the score in each `(doc, score)` pair is a cross-encoder relevance score (higher is better), not a distance. Callers that read `docs[0][1]` and compare against `1.15` are treating it as a distance — the value intended for that comparison is the separately returned `absolute_best_distance`. Citation "Score" in the UI is the cross-encoder score.
- **[evaluate_rag.py:112-113](evaluate_rag.py#L112-L113)** is a syntax error: missing comma after `raise_exceptions=False`.
- The README describes `bge-small-en-v1.5` embeddings; [database.py](database.py) uses `bge-base-en-v1.5` (768-dim). `evaluate_rag.py` uses bge-small for the Ragas judge, so the judge and the pipeline embed with different models.
- `modules/generation.py` also exposes `generate_response_stream`, which nothing calls — the UI uses the synchronous `generate_response`.
- `.gitignore` ignores `data/` and `*.pdf`, so source policy PDFs and the benchmark CSV are untracked by design.
- `.claude/settings.json` currently contains a copy of the gitignore text, not JSON.
