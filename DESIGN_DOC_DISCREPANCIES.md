# Design Document vs. Codebase — Discrepancy Report

**Revision 2** — full re-review after `database.py` and `ingestion.py` were moved into `modules/`.

**Artifacts reviewed:** `Software design document.docx` (cover: Version 2.0, 06 July 2026; revision history ends at 3.0, 08/07/2026), `README.md`, `requirements.txt`, `app.py`, `evaluate_rag.py`, `auth_service.py`, `modules/{ingestion,database,generation}.py`, `.streamlit/config.toml`, `.gitignore`, `config/users.json`, `data/benchmarking_data.csv`, `ragas_evaluation_results.csv`, `chroma_db/`.

**Method:** full text extracted from the .docx and diffed section-by-section against the code as committed on 2026-08-20.

> **Not reviewed:** §5.3 (System Architecture Diagram), Appendix B (Folder Structure) and the §9 screenshots are embedded images with no text layer. They may contain further discrepancies.

Findings are ordered worst-first. Items marked **[NEW in rev 2]** were not in the first pass.

---

## 0. Status of the file-structure fix — **incomplete** [NEW in rev 2]

`modules/database.py` and `modules/ingestion.py` are now in place. Two things remain, and the first is still blocking:

1. **`auth_service.py` is still at the repository root**, while `app.py:44` imports `from modules.auth_service import LocalAuthService`. This raises `ModuleNotFoundError` → the `st.error` / `st.stop` path at `app.py:49`. **The application still cannot start.** It needs to move into `modules/` alongside the other three.
2. **There is no `modules/__init__.py`** — the folder contains only `.gitkeep` plus the three module files. Python 3.3+ namespace packages make the imports work anyway, so this is not fatal, but both `README.md:56` ("`__init__.py` — Empty initialization file") and SDD Appendix B claim the file exists. Either create it or correct both documents.

After fixing (1), the next failure is the return-arity mismatch in §3 below, then the authentication lockout in §4.

---

## 1. The document's headline parser does not exist in the code

The document commits to `unstructured.io` in six places:

- §1.3 Scope — "High-fidelity document parsing using computer vision for tabular data extraction (unstructured.io)."
- §4 Risk table — "Replaced naive text readers (PyMuPDF) with unstructured.io computer vision."
- §5.2 — "unstructured.io performs bounding-box parsing to identify tables versus text."
- §8.1 — full code listing using `partition_pdf(strategy="hi_res", infer_table_structure=True, ...)`.
- §12 — "Exceeds requirement by handling complex tabular data via unstructured.io."
- Appendix A — lists unstructured.io, Poppler, Tesseract-OCR under "Computer Vision".

`modules/ingestion.py:4` imports `fitz` (**PyMuPDF**) and calls plain `page.get_text()` per page. There is no `partition_pdf`, no `hi_res` strategy, no `infer_table_structure`, no `text_as_html`, and no OCR anywhere in the tree.

Every downstream claim that depends on it fails with it:

- table-to-HTML/Markdown conversion (§5.1, §8.1),
- the §9.3 "Markdown grids (vital for displaying AI-generated tables)" rationale,
- the §12 "Exceeds requirement" verdict for FR-1.1,
- the §2.2 requirement for Poppler/Tesseract on the system PATH (the code needs neither).

### Chunking mismatch in the same component

| §8.1 states | `modules/ingestion.py:12` implements |
|---|---|
| `by_title` chunking strategy (respects headers/tables) | `RecursiveCharacterTextSplitter`, applied page-by-page |
| max 600 characters | `chunk_size=500` |
| overlap 100 | `chunk_overlap=50` |

§5.2's "chunked by logical document boundaries (headers/titles)" is therefore also inaccurate — chunking is per-page character splitting.

---

## 2. Both neural models are the wrong ones

### Embedding model

The document specifies `BAAI/bge-small-en-v1.5` at **384 dimensions**, pinned to CPU — stated in §3.3, §6.1, §7.2 (schema table), §10.2 and Appendix A. `README.md:16` and `README.md:20` repeat the same claim.

`modules/database.py:20` loads `BAAI/bge-base-en-v1.5`, which is **768-dimensional**, and passes no `model_kwargs`. Consequences:

- The "384-dimensional dense vector" in the §7.2 ChromaDB schema table is wrong.
- §6.1's "maps chunks into a 384-dimensional vector space" is wrong.
- `README.md:20`'s "Powered by `BAAI/bge-small-en-v1.5` embeddings" is wrong.
- The explicit `device="cpu"` pinning that §10.2 presents as the OOM mitigation **is not in the code** — the model will use a GPU if one is visible.

### Cross-encoder / reranker

The document specifies `cross-encoder/ms-marco-MiniLM-L-6-v2` on CPU — §6.3 explains the architecture by name, plus §8.2 and Appendix A.

`modules/database.py:24` loads `BAAI/bge-reranker-base` — a different model (~270MB vs ~80MB), again with no device argument. §6.3's entire technical explanation names a model the system does not use.

**[NEW in rev 2]** The README is worse: it **never mentions the cross-encoder at all**. `README.md:18-22` describes the retriever as Dense + BM25 + RRF only. The reranker is the SDD's flagship Phase 2 feature (§6.3, §8.2, revision history row 2.0) and it is genuinely in the code — the README simply omits it.

### Candidate pool size

§5.2 and §8.2 both fix `initial_k = 10` ("RRF to gather 10 broad candidates", "re-ranks these 10 candidates"). `modules/database.py:85` computes `candidate_count = top_k * 3`, i.e. **9** at the default `top_k=3`.

### Knock-on effect on evaluation

`evaluate_rag.py:91` uses **bge-small** for the Ragas judge while the pipeline embeds with **bge-base**. Appendix E's claim that "the evaluation script utilized local components entirely… alongside the BGE-Small embedding model" is only true of the judge, not of the pipeline under test. Three transformer models are therefore resident during evaluation: bge-base, bge-reranker-base, and bge-small.

---

## 3. The 1.15 relevance gate is broken — and the document's own listing makes it unimplementable

This is the most serious design-level finding, because §3.2 calls this mechanism the thing that "guarantee[s] zero hallucination," and §4 lists it as the primary mitigation for the High-impact "LLM Hallucination" risk.

### 3.1 The design as written cannot implement its own safety rule

§3.2 and §6.1 specify a gate on **cosine distance > 1.15**. But §8.2's own code listing returns `(docs, latency)` where the score in each pair has been deliberately overwritten — its comment reads *"Replace basic distance score with high-precision cross encoder placement score."* The distance is discarded and never returned, so no caller can perform the comparison §6.1 mandates. The document specifies a safety mechanism its own component design makes impossible.

### 3.2 The code half-fixed it and broke it differently

- `modules/database.py:147` returns a **3-tuple** `(docs, latency, absolute_best_distance)`.
- `app.py:230` unpacks **two** values → `ValueError` at runtime.
- `app.py:231` then reads `docs[0][1]` — the **cross-encoder logit** — and compares it to `1.15`.
- Same arity bug and same semantic bug at `evaluate_rag.py:58` and `evaluate_rag.py:64`.

Because reranker logits are *higher-is-better* and span roughly −11 to +11, the test `best_score > 1.15 → "I do not know."` is **inverted**:

- a strongly relevant chunk (logit ≈ +8) is **refused**;
- an irrelevant chunk (logit ≈ −10) **passes** straight to the LLM.

**Empirical confirmation:** across all 50 rows of `ragas_evaluation_results.csv`, **zero** responses are "I do not know." The anti-hallucination gate never fired once on the benchmark.

### 3.3 The metric is not cosine distance at all [NEW in rev 2]

§6.1 is titled "Dense Vector Retrieval (**Cosine Distance**)" and the entire mathematical foundation, plus the 1.15 constant, is expressed in cosine terms. §3.2 and §4 repeat "cosine distance". `README.md:32` says "similarity distance score".

`modules/database.py:31` constructs `Chroma(persist_directory=..., embedding_function=...)` with **no `collection_metadata`**, and `hnsw:space` appears nowhere in the repository. ChromaDB 0.4.x defaults to **squared L2 (Euclidean)**, not cosine. So:

- §6.1 describes a distance metric the system does not use;
- the 1.15 constant is not even on the same numeric scale. For unit-normalised BGE vectors, `L2² = 2·(1 − cos_sim)`, so a *cosine distance* of 1.15 corresponds to an `L2²` of ≈ 2.3. A threshold tuned as a cosine value is roughly half of what it should be when read as the L2² value Chroma actually returns.

Fix is one line — `collection_metadata={"hnsw:space": "cosine"}` on the Chroma constructor — but it must be applied at **collection-creation time**, so the existing `chroma_db/` store would need rebuilding, and the 1.15 constant re-tuned either way.

### 3.4 The gate reads the wrong chunk anyway [NEW in rev 2]

`modules/database.py:89` sets `absolute_best_distance = vector_results[0][1]` — the distance of the top **dense** hit, captured *before* RRF fusion and *before* reranking. §6.1 states "Any chunk exceeding this distance is deemed irrelevant to the query," implying a per-chunk test on what is actually sent to the LLM. As implemented, the gate asks only "did the dense index have anything nearby?", and the chunk it measures may not appear in the final top-3 at all.

### 3.5 The reranker fallback path forces answers through [NEW in rev 2]

`modules/database.py:143` — if the candidate pool is empty or the reranker is unavailable, each document is assigned a hardcoded score of `0.80`. Under the (broken) `> 1.15` gate, `0.80` passes, so the degraded path silently bypasses the safety check. The UI then renders "Score: 0.8000" as though it were a measured relevance value.

### 3.6 Wrong component named

§8.3 states the *generation* component "Evaluates distance thresholding; intercepts the prompt and returns the fallback safety string." It does not — the gate is in `app.py:235`. `modules/generation.py` contains no thresholding logic at all.

---

## 4. Authentication is unusable out of the box [NEW in rev 2]

§9.1 describes "A secure gateway requiring credentials that match the `users.json` SHA-256 hashes," and §7.3 documents the ledger as a working store.

The state of the repository:

- `config/users.json` exists but is **empty** (1 byte, no JSON content).
- `auth_service.py:21` only writes `{}` **if the file does not exist**. It exists, so the empty file is left in place.
- `auth_service.py:43` then calls `json.load(f)` on it → `JSONDecodeError`, caught at `auth_service.py:50` → sign-in returns `"Local Directory Error: …"`.
- `auth_service.py:63` hits the identical failure → registration returns `"Failed to write to database: …"`.
- There is **no `.streamlit/secrets.toml`** in the tree, so unless `AUTH_ADMIN_EMAIL` / `AUTH_ADMIN_PASSWORD` are exported in the environment, the super-admin bypass at `auth_service.py:36` is skipped too.

**Net effect: on a fresh clone there is no way to log in and no way to register.** Both paths fail on the same malformed file. The fix is to treat an empty or invalid file as `{}` (validate the parse, not just the path), and to ship a `secrets.toml.example`.

Two related gaps the documents should state:

- `auth_service.py:55` and `auth_service.py:59` **silently disable** the invite-code gate and the corporate-domain check when their secrets are unset. `README.md:38` presents both as active protections ("Self-registration is protected by an offline Corporate Invitation Token"); as shipped, neither is.
- `auth_service.py:36-38` compares the super-admin password in **plaintext**, bypassing the SHA-256 path that §9.1 says all credentials go through.

---

## 5. Streaming / TTFT is documented throughout but not wired up

The document treats token streaming as a shipped feature in four places:

- §2.4 Known Limitations — "mitigated in the UI layer through asynchronous token streaming."
- §4 Risk table — "Implemented a Streamlit asynchronous generator to stream output tokens instantly."
- §8.3 — "**Output:** An asynchronous stream of output tokens."
- §9.3 — "Latency Counter: Appends the Time-to-First-Token (TTFT)."

`generate_response_stream` exists at `modules/generation.py:58` but **nothing calls it**. `app.py:241` calls the synchronous `generate_response` behind a spinner, and `app.py:257` renders total wall-clock latency (retrieval + generation), not time-to-first-token. The stated mitigation for the document's own Medium-impact "UI Timeout (TTFT)" risk is not implemented.

---

## 6. Evaluation claims are not supported by the artifacts

### 6.1 The 0.9216 score covers 17 of 50 questions, not 50

§11.2 (DT-01) and Appendix E present Context Recall = 0.9216 as evidence that "the hybrid search effectively retrieved the exact factual chunks required to answer the query 92% of the time."

In `ragas_evaluation_results.csv`, only **17 of 50 rows** carry a `context_recall` value. The mean of those 17 is exactly 0.9216. The remaining 33 are blank — corresponding to the `TimeoutError` and `ValidationError` lines visible in the document's own Appendix E log.

The headline metric therefore describes **34%** of the benchmark, and the excluded 66% are not a random sample — they are the questions on which the local judge failed. This limitation must be stated in §11.2, not left implicit in a pasted log.

### 6.2 `max_workers=1` was not in effect

§4 ("Ragas Timeout Crashes") and Appendix E both claim sequential execution prevented CPU queue overload. The log pasted into Appendix E shows jobs completing **interleaved and out of order** — Job[3], Job[13], Job[6], Job[4], Job[9] — which is concurrent execution, not sequential.

- `evaluate_rag.py:112-113` is a **syntax error** (missing comma after `raise_exceptions=False`), so the file as committed cannot run at all. The CSV was produced by an earlier version of the script.
- In Ragas, `max_workers` belongs in `RunConfig(max_workers=1)` passed to `evaluate()`, not as a bare kwarg on `evaluate()`.

### 6.3 The CSV was produced by a different Ragas version than the script targets [NEW in rev 2]

`evaluate_rag.py:79-84` builds the dataset with the **Ragas 0.1-era** column names:

```
question, ground_truth, answer, contexts
```

`ragas_evaluation_results.csv` has the **Ragas 0.2-era** column names:

```
user_input, retrieved_contexts, response, reference
```

These are not the same schema. Combined with `requirements.txt` pinning `langchain-core==0.1.33` while leaving `ragas` **unpinned** (modern Ragas requires `langchain-core>=0.2`), this is an unresolved dependency conflict: `pip install -r requirements.txt` will either fail to resolve or silently upgrade `langchain-core` and break the pinned LangChain 0.1.x stack. Pin `ragas` to the version that actually produced the results, and reconcile it with the LangChain pins.

### 6.4 DT-04 and DT-03 are unsourced

- **DT-04 "+14% Precision" (Cross-Encoder Lift):** no A/B harness exists in the repository comparing RRF-only retrieval against RRF + reranker. There is no artifact from which this number could have been computed.
- **DT-03 "100% Effectiveness" (Regex Sanitization):** there is no test file, fixture, or seeded-PII document in the tree. The repository has no test suite of any kind.

### 6.5 §11.2 and §12 contradict each other on Faithfulness

§11.2 marks DT-02 (Faithfulness) as **"Planned"** with score "N/A (JSON parsing ongoing)". §12 marks FR-5.1 Mathematical Benchmarking as **"Meets completely."** The `faithfulness` column is empty for all 50 rows of the CSV.

### 6.6 The benchmark dataset is git-ignored [NEW in rev 2]

`.gitignore:6` ignores `data/`, so `data/benchmarking_data.csv` — the 50-pair ground-truth set that `evaluate_rag.py:35` requires and that Appendix F reproduces in full — is **untracked**. Anyone cloning the repository cannot reproduce the evaluation. Given that Appendix F presents the dataset as part of the deliverable, it should be force-added or moved outside `data/`.

---

## 7. The PII firewall actively degrades the benchmark

§4 presents the regex firewall as unqualified upside. On the actual corpus (`data/documents/amd-global-code-of-conduct.pdf`) it destroys ground-truth answers, because `modules/ingestion.py:32` redacts email addresses *before* embedding.

Concrete failures:

- **Appendix F, Q48** — "What email address should be used to contact the company's Data Protection Officer?" Ground truth: `privacy@amd.com`. The recorded pipeline answer in the CSV is *"The correct email address to contact the Company's Data Protection Officer is: **[REDACTED_EMAIL]**."* Scored blank.
- **Appendix F, Q16** — ground truth `infosec.incidents@amd.com`: same failure mode.
- Six rows in `ragas_evaluation_results.csv` carry `[REDACTED]` markers inside their retrieved contexts.

Additionally, three of the four regexes are **Indian-format** (`+91` mobile, Aadhaar, PAN) while the only ingested document is a US corporate policy. On this corpus those three patterns are no-ops, and the one pattern that does fire is the one that breaks the benchmark.

The document should either narrow the PII claim to what the corpus actually needs, or exclude published corporate contact addresses from redaction via an allowlist.

---

## 8. The "100% air-gapped" claim has two documented exceptions [NEW in rev 2]

§2.3 ("Strict Offline Execution"), §10.3 ("enforces strict zero-data-transmission policies… At no point does the application make HTTP requests to external commercial APIs") and `README.md:16` all state the system is fully air-gapped.

1. **ChromaDB telemetry is enabled by default and never disabled.** `ANONYMIZED_TELEMETRY` is set nowhere in the repository — not in the code, not in `.streamlit/config.toml`, not in any env file. The SDD's **own Appendix E log** contains the proof:

   > `Anonymized telemetry enabled. See https://docs.trychroma.com/telemetry` … `Failed to send telemetry event ClientStartEvent`

   That is an outbound network attempt originating from the pipeline, recorded in the document that claims none occur. Set `ANONYMIZED_TELEMETRY=False` (or `chromadb.config.Settings(anonymized_telemetry=False)`) and the claim becomes true.

2. **First-run model downloads.** `modules/database.py:20` and `:24` fetch `bge-base-en-v1.5` and `bge-reranker-base` from HuggingFace on first use (~700MB combined); `evaluate_rag.py:90` fetches bge-small as well. The system is offline only *after* this bootstrap. Neither document states the caveat — §2.3 and README §1 read as though the system never touches the network. A one-line "first run requires connectivity to populate the local model cache" resolves it.

---

## 9. Interface Design describes three features that are not built

| §9 claim | Reality |
|---|---|
| §9.2 "**File Uploader:** Allows Admins to drag-and-drop new PDF policies into the local directory." | No `st.file_uploader` anywhere in `app.py`. Admins must copy PDFs into `data/documents/` manually. |
| §9.2 "**Telemetry Pane:** Displays live backend metrics, including database chunk counts, embedding dimensions, and previous query latency." | The sidebar (`app.py:155-185`) contains only a user-info box, the ingestion button, and sign-out. No telemetry pane exists. |
| §9.3 "**Citation Expanders:** a Streamlit `st.expander` widget displays the exact source document name, page number, and raw text chunk used to generate the answer, providing full auditability." | `app.py:246-252` renders a flat HTML `citation-card` div showing source, page and score. No expander, and **the raw chunk text is never displayed** — which undercuts the "full auditability" claim. |

**[NEW in rev 2]** `README.md:45` describes `.streamlit/config.toml` as "Forces light theme settings globally." The file sets `primaryColor`, `backgroundColor`, `secondaryBackgroundColor`, `textColor` and `font`, but **not `base = "light"`**. Streamlit still derives its base theme from the viewer's system preference, so the theme is not actually forced.

---

## 10. Data Design tables are inaccurate

### §7.2 ChromaDB Vector Vault Schema

- `embedding` described as "384-dimensional" — actually 768 (see §2).
- `metadata` described as containing `page_number` — the code writes the key `page` (`modules/ingestion.py:70`).
- The distance metric implied throughout is cosine — it is L2 (see §3.3).

### §7.3 Local Authentication Ledger

| §7.3 states | `auth_service.py` implements |
|---|---|
| Primary key: `Username (String)` | Keyed by **email** address (`auth_service.py:70`) |
| Field name: `password_hash` | Field name is `password` |

§7.3 and §9.1 also omit three implemented mechanisms entirely: the corporate-domain restriction (`auth_service.py:59`), the invite-code gate (`auth_service.py:55`), and the `st.secrets` / `AUTH_*` super-admin account — see §4 above for why all three matter.

---

## 11. Implemented behaviour the documents never mention

### 11.1 Incremental ingestion (a real feature, absent from both documents)

`modules/ingestion.py:14-28` maintains `data/ingestion_registry.json` keyed on each PDF's modification time, so re-runs skip unchanged files. §8.1's process flow says only "Iterates through the /data/documents/ folder," and the README does not mention it at all.

Two caveats belong in the document with it:

- `modules/database.py:67` **appends** to the store, so re-ingesting a *modified* PDF **duplicates** its chunks rather than replacing them.
- **[NEW in rev 2] The registry is committed before the chunks are embedded.** `modules/ingestion.py:83-84` writes the registry at the end of `extract_and_chunk()`, but embedding happens afterwards in a separate call at `app.py:170`. If `save_chunks_to_db` throws — caught at `app.py:177` — the registry already records those PDFs as processed, so they are **permanently skipped on every subsequent run** and their content never reaches the index. Silent, unrecoverable-without-manual-edit data loss. The registry write should move to after a successful embed.

### 11.2 BM25 caching — here the code is better than the design

§8.2's listing rebuilds the Chroma connection *and* the BM25 index from `vector_db.get()` **inside `retrieve_context_hybrid`**, i.e. on every single query. This directly contradicts §10.2's claim that `@st.cache_resource` loads the vector database and models into memory "only once upon application startup."

The shipped `modules/database.py:28` (`_load_db`) caches both correctly. The document should be updated to match the code, not the reverse.

### 11.3 Further defects in the §8.2 listing, if it is kept as the plan

1. `save_chunks_to_db` assigns `Chroma.from_documents(...)` to a **local** variable never stored on `self`, and uses `from_documents` on every call rather than appending to an existing store.
2. `scores_map` is populated (including a hardcoded `0.80` placeholder) but **never read** — dead code.
3. It returns `None, 0.0` on an empty database, while callers index the first element as a list; the shipped code correctly returns `[], 0.0, 9.9`.

### 11.4 `.gitkeep` sentinels defeat the "is it empty?" checks [NEW in rev 2]

`app.py:220` guards the query path with `not os.listdir("./chroma_db")`, and `modules/database.py:30` gates `_load_db` the same way. `chroma_db/` currently contains a `.gitkeep`, so **both checks report a populated database when the store is in fact empty**. The user-facing "Vector database is empty. Admin must trigger Ingestion first." warning will never fire. Behaviour degrades to a silent "I do not know." for every question instead of the intended instruction. Check for `chroma.sqlite3` (or a non-zero collection count) rather than directory non-emptiness.

### 11.5 Smaller code/doc divergences [NEW in rev 2]

- **Error messages.** §8.3's listing raises the friendly `"Ollama background service is offline. Please make sure Ollama is open in your taskbar tray."` for both failure paths. `modules/generation.py:56` and `:94` raise raw `f"Query Rewrite Failed: {e}"` / `f"Generation Failed: {e}"` instead. The documented UX is not implemented.
- **Which query reaches the LLM.** `app.py:226` computes `search_query` via rewriting, uses it for retrieval at `app.py:230`, then passes the **original** `user_query` to `generate_response` at `app.py:241`. This is defensible, but §5.2, §8.3 and `README.md:35` never say which query is used for generation. State it explicitly.
- **"Last 3 chat turns."** §5.2 and `README.md:35` say the rewriter sees 3 turns. `modules/generation.py:51` slices `chat_history[-3:]` — 3 **messages**, i.e. 1.5 turns.
- **Citation suppression is a substring test.** `app.py:245` and `app.py:262` use `"I do not know" not in ai_response`. A legitimate answer that quotes the phrase loses its citations.
- **BGE query instruction missing.** BGE retrieval models are trained with a query-side instruction prefix ("Represent this sentence for searching relevant passages: "). `modules/database.py:20` uses plain `HuggingFaceEmbeddings`, which does not add it — `HuggingFaceBgeEmbeddings` does. Retrieval is running below the model's designed performance. Not a doc/code mismatch, but it undercuts the §6.1 accuracy rationale.

---

## 12. README-specific problems [NEW in rev 2]

Beyond the model-name and reranker-omission issues already covered:

- **The code fence is never closed.** `README.md:41` opens a ```` ```text ```` block for the directory tree; there is exactly **one** fence marker in the file. Everything from line 42 to EOF renders as a single code block on GitHub.
- **There are no setup or execution instructions.** Line 65 describes the README as the "Setup and execution guide," but the file ends at the directory tree. There is no `pip install`, no `ollama pull llama3.2`, no `streamlit run app.py`, and no mention that Ollama must be running first. This is the single most useful thing to add.
- **"Qwen" appears in a project that has no Qwen.** `README.md:59` — "generation.py — Strict **Qwen**/Llama inference". Leftover from an earlier iteration; the model is Llama 3.2 everywhere in code and in the SDD.
- **The directory tree omits four real files:** `evaluate_rag.py`, `auth_service.py`, `data/benchmarking_data.csv` and `ragas_evaluation_results.csv`. `auth_service.py` is a core module and the README's own §6 describes its behaviour.
- **The README never mentions Ragas or the evaluation framework**, which is the entire subject of SDD §11 and Appendix E and the stated reason for revision 3.0.
- **`README.md:16` cites "Firebase"** as an example of an avoided cloud API — a leftover reference to a service this project never used.

### Requirement-ID schemes are mutually incompatible

Three different identifier systems are in play with **no mapping between them anywhere**:

| Artifact | Scheme in use |
|---|---|
| SDD §12 cross-reference table | `FR-1.1`, `FR-1.2`, `FR-2.1`, `FR-3.1`, `FR-4.1`, `FR-5.1` |
| README | `Req-01`, `REQ-03`, `Req-04`, `REQ-05`, `REQ-07`, `Req-08`, `Req-10` + `M3`, `M5`, `M7`, `M8`, `M10` |
| Code comments | `REQ-02`, `REQ-07`, `REQ-08`, `REQ-10` |

Within that, IDs are reused for unrelated things:

- **`REQ-07`** labels four different features — data sovereignty (`README.md:15`), query rewriting (`README.md:34`), the auth gateway (`README.md:37`), role segregation (`app.py:159`), SHA-256 hashing (`auth_service.py:26`) and domain segregation (`auth_service.py:58`).
- **`REQ-02`** labels both the chunk size (`modules/ingestion.py:11`) and the embedding model (`modules/database.py:19`).
- README capitalisation alternates between `Req-` and `REQ-` in the same document, and `README.md:18` contains an apparent typo, `M25`, that matches no other identifier.

§12 is the natural place to fix this: extend the table so every `FR-x.x` lists its corresponding `REQ-`/`M` label, then make the code comments and README agree.

---

## 13. `requirements.txt` problems

- **Missing but imported:** `pandas` (`evaluate_rag.py:3`) and `langchain-text-splitters` (`modules/ingestion.py:5`). The latter may arrive transitively depending on how `langchain==0.1.12` resolves — pin it explicitly rather than relying on that.
- **[NEW in rev 2] Pinned but unused:** `requests==2.31.0` is never imported by any file in the tree. (SDD §8.3's *listing* imports it; the real `modules/generation.py` does not.)
- **Pinned but unused, and expensive:** `unstructured[pdf]`, `pdf2image` and `unstructured-inference` are dead weight given §1 — they pull a large vision stack, plus the Poppler/Tesseract system prerequisites in §2.2, for code that never runs.
- **[NEW in rev 2] Unpinned and conflicting:** `ragas` and `datasets` carry no version. Modern Ragas requires `langchain-core>=0.2`, which contradicts the pinned `langchain-core==0.1.33` — see §6.3.
- SDD Appendix C omits `langchain-community==0.0.28`, which *is* in the file, and omits `pandas`. It lists `PyMuPDF==1.23.26` as a live dependency while §4 claims PyMuPDF was replaced — an internal contradiction inside the document.

---

## 14. Internal contradictions within the documents

- **Version numbers disagree across three artifacts.** SDD cover reads "Version <2.0>, 06 July 2026"; the SDD revision history's final row is "3.0, 08/07/2026 — Third Release: Final Software Design Document"; `README.md:1` reads "**v4.0**" and line 13 says "SRS v4.0 Compliance". Pick one scheme. The cover should at minimum read 3.0. The 2.0 row is also dated 15/06/2026, inconsistent with the cover's 06 July.
- **§11.2 is mislabelled** "Test Results (Phase 1 Baseline Metrics)" but contains the final-phase Ragas run described in Appendix E.
- **Appendix E's terminal log** shows the path `D:\DBDA\RAG-Based-HR-policy-Q-A-main` and the conda env `C:/Users/dbda56/.conda/envs/project` — a different project name and a different machine from the one on the cover page.
- **Appendix F calls the pairs "synthetic,"** but the ground-truth answers are verbatim extractions from AMD's published Global Code of Conduct. Neither document names the source corpus anywhere; a reviewer will ask.
- **§10.1** states "The Generative LLM generation time maintains an upper bound of O(1)" — generation time is bounded by output length, not constant. The HNSW O(log N) retrieval claim is fine.
- **§12** names the ingestion function `extract_and_chunk_pdf()`; it is `extract_and_chunk()`.

---

## Summary — recommended next steps

### Blocking, in the order you will hit them

1. **Move `auth_service.py` into `modules/`** — `app.py:44` still imports `modules.auth_service`. The app cannot start. (§0)
2. **Fix the return-arity mismatch** — `app.py:230` and `evaluate_rag.py:58` unpack 2 values from a 3-tuple. (§3.2)
3. **Repair the auth lockout** — treat an empty/invalid `config/users.json` as `{}`, and ship a `secrets.toml.example`. Right now nobody can log in or register. (§4)
4. **Fix `evaluate_rag.py:112-113`** — missing comma after `raise_exceptions=False`; the file will not parse. (§6.2)

### Correctness fixes that are wrong under any interpretation

5. **The relevance gate** — use `absolute_best_distance`, not the reranker logit; correct the comparison direction; set `hnsw:space` to cosine if §6.1 is to stay as written, and re-tune 1.15 for whichever metric you keep. Decide whether the gate should measure the top dense hit or the finally-selected chunk. (§3.2–3.5)
6. **Move the ingestion registry write** to *after* a successful embed, or a failed ingestion permanently loses those PDFs. (§11.1)
7. **Stop checking directory non-emptiness** for "is the vector DB populated" — `.gitkeep` defeats it in two places. (§11.4)
8. **Set `ANONYMIZED_TELEMETRY=False`**, which makes the air-gap claim in §2.3/§10.3 actually true. (§8)
9. **Restate 0.9216** in §11.2 and Appendix E as the mean over 17 of 50 questions, with the 33 judge failures disclosed. (§6.1)

### Two strategic choices only the author can make

10. **Parsing:** rewrite §1.3/§4/§5.2/§8.1/§12 down to PyMuPDF page-text extraction, **or** implement `unstructured.io` as designed. The document's design is stronger here (tables matter in a policy corpus), but it is substantial work — and if you drop it, also drop `unstructured[pdf]`, `pdf2image`, `unstructured-inference` and the §2.2 Poppler/Tesseract prerequisites.
11. **Streaming:** delete the TTFT/streaming claims from §2.4/§4/§8.3/§9.3, **or** switch `app.py` to the already-written `generate_response_stream` via `st.write_stream`. This one is cheap — the function exists and works.

Note the reconciliation runs in **both directions**: the document is better on parsing and streaming; the code is better on index caching (§10.2) and incremental ingestion (§8.1).

### Mechanical documentation corrections

- bge-small → **bge-base**, 384 → **768** dims (SDD §3.3, §6.1, §7.2, §10.2, App A; README lines 16, 20).
- ms-marco-MiniLM-L-6-v2 → **bge-reranker-base** (SDD §6.3, §8.2, App A); **add the reranker to the README**, which omits it entirely.
- Cosine → **L2**, or change the code (SDD §3.2, §4, §6.1).
- 600/100 chunking → **500/50**; `by_title` → `RecursiveCharacterTextSplitter` (SDD §8.1).
- `initial_k = 10` → **`top_k * 3` = 9** (SDD §5.2, §8.2).
- `page_number` → `page`; `password_hash` → `password`; username key → **email** (SDD §7.2, §7.3).
- `extract_and_chunk_pdf()` → `extract_and_chunk()` (SDD §12).
- Create `modules/__init__.py` or correct README line 56 and Appendix B.
- Remove or build the file uploader, telemetry pane, and citation expander (SDD §9.2, §9.3); add `base = "light"` to `config.toml` or correct README line 45.
- Document the ingestion registry, the append-duplication caveat, the invite code, the domain restriction, the plaintext super-admin path, and the first-run model download.
- **README:** close the code fence, add setup/run instructions, remove "Qwen" and "Firebase", add the four missing files to the tree, and mention the Ragas evaluation.
- **Requirement IDs:** unify `FR-x.x` / `REQ-nn` / `Mn` into one scheme via §12; `REQ-07` currently labels six unrelated things and `REQ-02` labels two.
- **Versions:** SDD cover → 3.0; reconcile with README's "v4.0"; retitle §11.2; reconcile Appendix C with `requirements.txt`; name the AMD corpus.
- Un-ignore `data/benchmarking_data.csv` so the evaluation is reproducible.
