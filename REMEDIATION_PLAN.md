# Remediation Plan

Companion to [DESIGN_DOC_DISCREPANCIES.md](DESIGN_DOC_DISCREPANCIES.md). That file is the audit — *what* is wrong. This file is the work plan — *how* to fix it, in what order, with the exact edits.

**Written 2026-08-20 for the next working session. No code has been changed yet.**

---

## How to use this document

- **Phases 0–4 require no decisions.** They are ordered by dependency; each one unblocks the next. Start at P0-1 and work down. Roughly 80% of the total work lives here.
- **Phases 5–7 are branch work.** Two strategic decisions are deferred (see below). Both branches are written out in full, so we can pick at the moment we get there without stopping to research.
- Every task has: the file and line, what's wrong, the concrete change, and how to verify it worked.
- Task IDs (`P0-1`, `P1-3`, …) are stable — use them to say "let's do P2-1 next."

**Legend:** 🔴 blocking · 🟠 correctness · 🟡 robustness · 🔵 docs/hygiene

---

## Deferred decisions (do NOT need answering to start)

Nothing in Phases 0–4 depends on either of these. We can decide when we reach Phase 5/6, or never.

### Decision A — Document parsing

| | Branch A1: keep PyMuPDF | Branch A2: implement unstructured.io |
|---|---|---|
| **Work** | Rewrite SDD §1.3, §4, §5.2, §8.1, §12, App A. Drop 3 deps. | Rewrite `modules/ingestion.py`; install Poppler + Tesseract; full re-ingest. |
| **Effort** | ~1 hour, docs only | 2+ sessions; `hi_res` is slow on CPU |
| **Risk** | Low | Medium-high |
| **Gain** | Honest docs, lighter install | Real table extraction; §12's "exceeds requirement" becomes true |
| **Cost** | Lose the table-handling story | Re-ingest + re-run the whole benchmark |

Everything else in this plan works identically under either branch. **Phase 5 holds this work.**

### Decision B — Token streaming / TTFT

| | Branch B1: wire it up | Branch B2: drop the claims |
|---|---|---|
| **Work** | Switch `app.py` to `generate_response_stream` via `st.write_stream`; measure real TTFT. | Delete streaming language from SDD §2.4, §4, §8.3, §9.3; delete `modules/generation.py:58-77`. |
| **Effort** | ~20 lines — the function is already written and works | Minimal |
| **Gain** | Makes four SDD sections true at once; much better UX on CPU | Removes dead code |
| **Cost** | Slightly more complex latency accounting | §2.4 states a limitation with no mitigation |

**Phase 6 holds this work.**

> A third, smaller choice appears at **P1-2** (distance metric). It has a clear low-risk default and is written up as a recommendation, not a branch — see the task.

---

# PHASE 0 — Make it run 🔴

Nothing else can be tested until these four are done. Estimated 30–45 minutes total.

## P0-1 🔴 Finish the module move

**Problem:** `auth_service.py` is still at the repo root; `app.py:44` imports `modules.auth_service` → `ModuleNotFoundError` → `st.stop()` at `app.py:49`. (Discrepancy §0)

**Change:**
```powershell
Move-Item auth_service.py modules\auth_service.py
New-Item -ItemType File modules\__init__.py
Remove-Item modules\.gitkeep
```

`modules/__init__.py` stays empty — README line 56 and SDD Appendix B both say it exists, and it stops the folder being a namespace package.

**Verify:** `python -c "from modules.auth_service import LocalAuthService; print('ok')"`

---

## P0-2 🔴 Fix the return-arity mismatch

**Problem:** `modules/database.py:147` returns a 3-tuple; `app.py:230` and `evaluate_rag.py:58` unpack 2 → `ValueError`. (Discrepancy §3.2)

**`app.py:230-231` — replace:**
```python
retrieved_docs_with_scores, ret_latency = vector_db.retrieve_context_hybrid(search_query)
best_score = retrieved_docs_with_scores[0][1] if retrieved_docs_with_scores else 9.9
```
**with:**
```python
retrieved_docs_with_scores, ret_latency, best_distance = vector_db.retrieve_context_hybrid(search_query)
```

Then at `app.py:235` and `app.py:262`, replace `best_score` with `best_distance`. This lands P1-1 at the same time — the two are the same edit.

**`evaluate_rag.py:58` and `:64` — replace:**
```python
retrieved_docs_with_scores, _ = vector_db.retrieve_context_hybrid(question, top_k=3)
...
best_score = retrieved_docs_with_scores[0][1] if retrieved_docs_with_scores else 9.9
if best_score > 1.15:
```
**with:**
```python
retrieved_docs_with_scores, _, best_distance = vector_db.retrieve_context_hybrid(question, top_k=3)
...
if best_distance > RELEVANCE_THRESHOLD:
```

> **Keep the two call sites identical.** They drifted apart once already, which is how the benchmark ended up measuring different behaviour from the app. P1-3 makes the threshold a shared constant so this can't recur.

**Verify:** app starts and a query returns something without a `ValueError` in `admin_secure.log`.

---

## P0-3 🔴 Repair the authentication lockout

**Problem:** `config/users.json` exists but is empty (1 byte). `auth_service.py:21` only writes `{}` when the file is *missing*, so `json.load` raises on both sign-in (`:43`) and registration (`:63`). With no `.streamlit/secrets.toml` either, there is no way to authenticate at all. (Discrepancy §4)

**Change — in `modules/auth_service.py`, replace lines 18-23:**
```python
        # Ensure the config directory exists
        os.makedirs(os.path.dirname(self.filepath), exist_ok=True)
        # Initialize JSON database file if it doesn't exist yet
        if not os.path.exists(self.filepath):
            with open(self.filepath, "w") as f:
                json.dump({}, f)
```
**with:**
```python
        os.makedirs(os.path.dirname(self.filepath), exist_ok=True)
        self._ensure_store()

    def _ensure_store(self):
        """Creates the ledger if absent, and repairs it if empty or malformed."""
        try:
            with open(self.filepath, "r") as f:
                if isinstance(json.load(f), dict):
                    return
        except (FileNotFoundError, ValueError):
            pass
        with open(self.filepath, "w") as f:
            json.dump({}, f)

    def _read_users(self):
        """Reads the ledger, self-healing on a corrupt or empty file."""
        self._ensure_store()
        with open(self.filepath, "r") as f:
            return json.load(f)
```

Then replace the `with open(self.filepath, "r") as f: users = json.load(f)` blocks in `sign_in_user` (`:42-43`) and `sign_up_user` (`:63-64`) with `users = self._read_users()`.

**Also add `.streamlit/secrets.toml.example`** (and confirm `.gitignore` covers the real `secrets.toml`):
```toml
[admin]
admin_email = "admin@cdac.in"
admin_password = "change-me"
invite_code = "change-me"
corporate_domain = "@cdac.in"
```

**Verify:** delete `config/users.json`, start the app, register `test@cdac.in`, sign in. Then blank the file to 0 bytes and repeat — both must work.

---

## P0-4 🔴 Fix the evaluation script syntax error

**Problem:** `evaluate_rag.py:112-113` is missing a comma after `raise_exceptions=False` — the file does not parse. Separately, `max_workers` is not an `evaluate()` kwarg. (Discrepancy §6.2)

**Change — replace lines 107-114:**
```python
    result = evaluate(
        dataset=dataset,
        metrics=[context_recall, faithfulness],
        llm=eval_llm,
        embeddings=eval_embeddings,
        raise_exceptions=False,
        run_config=RunConfig(max_workers=1, timeout=300),
    )
```
and add near the other ragas imports:
```python
from ragas.run_config import RunConfig
```

**Verify:** `python -c "import ast; ast.parse(open('evaluate_rag.py').read()); print('parses')"`

> Confirm the `RunConfig` import path against the installed ragas version — it moved between 0.1 and 0.2. This interacts with **P4-1**; if we pin ragas first, verify once.

---

# PHASE 1 — Fix the relevance gate 🟠

This is the anti-hallucination mechanism SDD §3.2 calls the guarantee of "zero hallucination." It currently never fires. Do this phase as a unit.

## P1-1 🟠 Gate on the distance, not the reranker logit

Already covered by the P0-2 edit — `best_distance` replaces `docs[0][1]`. Recorded separately because it is the actual *semantic* fix: `docs[0][1]` is a cross-encoder logit (higher = better, roughly −11…+11), so `> 1.15` was **inverted** — refusing good chunks and admitting bad ones. (Discrepancy §3.2)

**Verify:** ask something absurd ("what is the capital of Mars?"). It must return exactly `I do not know.` Today it does not.

---

## P1-2 🟠 Settle the distance metric — recommendation, low risk

**Problem:** SDD §6.1 is built on *cosine distance*, but `modules/database.py:31` never sets `hnsw:space`, so ChromaDB defaults to **squared L2**. (Discrepancy §3.3)

**Recommended: keep L2, fix the document.** The arithmetic is on our side. For unit-normalised BGE vectors, `L2² = 2·(1 − cos_sim)`, so the existing threshold means:

```
L2² = 1.15  ->  cos_sim = (2 - 1.15) / 2 = 0.425
            ->  cosine distance = 0.575
```

A cosine similarity floor of **0.425** is a sensible relevance cut-off for BGE. The 1.15 constant was never wrong *as an L2² value* — only its label was. Meanwhile, read as a literal *cosine distance*, 1.15 would mean "accept anything not actively anti-correlated," i.e. effectively no gate at all.

So: keep the code as-is, and correct SDD §6.1, §3.2 and §4 to say **squared L2 distance**, noting the cosine equivalence above. No re-ingest, no re-tuning, no risk.

**Alternative, only if §6.1 must stay literally true:** set `collection_metadata={"hnsw:space": "cosine"}` on the `Chroma(...)` constructor at `modules/database.py:31` **and** in `save_chunks_to_db`, change the threshold to **0.575**, and **delete and rebuild `chroma_db/`** — `hnsw:space` is fixed at collection creation and silently ignored on an existing collection. Then redo P1-3 and P4-3.

---

## P1-3 🟠 Make the threshold a single shared constant

**Problem:** `1.15` is hardcoded in `app.py:235`, `app.py:262` and `evaluate_rag.py:66`. The audit found the app and the benchmark had already drifted apart. (Discrepancy §3.2)

**Change:** add to `modules/database.py` (top level):
```python
# Maximum squared-L2 distance for a chunk to count as relevant.
# For unit-normalised BGE vectors: L2^2 = 2*(1 - cos_sim),
# so 1.15 corresponds to a cosine-similarity floor of 0.425.
RELEVANCE_THRESHOLD = 1.15
```
Import it in `app.py` and `evaluate_rag.py` and use it at all three sites.

**Verify:** `grep -rn "1\.15" *.py modules/` returns only the constant definition.

---

## P1-4 🟠 Stop the reranker fallback bypassing the gate

**Problem:** `modules/database.py:143` assigns a hardcoded `0.80` when the candidate pool is empty or the reranker is unavailable. Under a `> 1.15` gate, `0.80` passes — so the degraded path silently forces answers through, and the UI renders "Score: 0.8000" as though it were measured. (Discrepancy §3.5)

**Change:** the fallback must not fabricate a passing score. Return the real dense distance where one exists, and make the reranker's absence explicit:
```python
else:
    # Reranker unavailable: fall back to RRF order, but flag the scores
    # as unmeasured rather than inventing a passing value.
    for content in sorted_contents[:top_k]:
        fused_docs_with_scores.append((doc_map[content], float("nan")))
```
Then in `app.py`, render `Score: n/a` when the score is `NaN` rather than `{:.4f}`.

The gate itself is unaffected — it reads `absolute_best_distance`, which is still real.

---

## P1-5 🟡 Decide what the gate measures

**Problem:** `modules/database.py:89` captures `absolute_best_distance` from the top **dense** hit, *before* fusion and reranking. SDD §6.1 says "any chunk exceeding this distance is deemed irrelevant," implying a test on what is actually sent to the LLM. (Discrepancy §3.4)

**Recommended:** keep the current behaviour — it is a cheap sanity check on "does the corpus contain anything near this query at all" — but **say so** in §6.1. Wording: *"the gate tests the closest dense-retrieval candidate; it is a corpus-level relevance check, not a per-chunk filter."*

If we want a true per-chunk test later, it needs the distance carried alongside each doc through fusion and reranking — a bigger refactor of `retrieve_context_hybrid`'s return shape. Not worth it now.

---

# PHASE 2 — Data integrity 🟡

## P2-1 🟠 Stop the ingestion registry losing documents silently

**Problem:** `modules/ingestion.py:83-84` commits the registry at the end of `extract_and_chunk()`, but embedding happens afterwards at `app.py:170`. If `save_chunks_to_db` throws (caught at `app.py:177`), those PDFs are already marked processed and are **permanently skipped on every future run** — their content never reaches the index. (Discrepancy §11.1)

**Change — in `modules/ingestion.py`, remove the auto-commit at lines 82-84** and stage it instead:
```python
        # Stage the registry update; the caller commits after a successful embed.
        self._pending_registry = new_registry
        return all_chunks

    def commit_registry(self):
        """Persists the staged registry. Call ONLY after chunks are embedded."""
        if getattr(self, "_pending_registry", None) is not None:
            self._save_registry(self._pending_registry)
            self._pending_registry = None
```

**In `app.py:169-171`:**
```python
chunks = doc_processor.extract_and_chunk()
count = vector_db.save_chunks_to_db(chunks)
doc_processor.commit_registry()   # only reached if the embed succeeded
```

**Verify:** temporarily raise inside `save_chunks_to_db`, trigger ingestion, confirm `data/ingestion_registry.json` is unchanged, remove the raise, re-trigger, confirm the PDF is processed.

---

## P2-2 🟡 Stop `.gitkeep` faking a populated database

**Problem:** `app.py:220` and `modules/database.py:30` both test `os.listdir(...)` for emptiness. `chroma_db/` contains a `.gitkeep`, so both report "populated" when the store is empty — the "Vector database is empty. Admin must trigger Ingestion first." warning can never fire, and every question silently returns "I do not know." (Discrepancy §11.4)

**Change:** add to `modules/database.py`:
```python
def _store_exists(db_path):
    """A Chroma store is real only if its SQLite file is present."""
    return os.path.exists(os.path.join(db_path, "chroma.sqlite3"))
```
Use it at `modules/database.py:30` and import it in `app.py` for the guard at line 220.

**Verify:** with only `.gitkeep` in `chroma_db/`, a query must show the "Admin must trigger Ingestion first" error, not "I do not know."

---

## P2-3 🟡 Stop re-ingestion duplicating chunks

**Problem:** `modules/database.py:67` appends unconditionally, so re-ingesting a *modified* PDF duplicates its chunks rather than replacing them. The registry deliberately re-processes modified files, so this triggers on the normal path. (Discrepancy §11.1)

**Change:** before adding, delete any existing chunks for those sources:
```python
sources = {m["source"] for m in metadatas}
if self.vector_db is not None:
    for src in sources:
        self.vector_db.delete(where={"source": src})
```

**Verify:** ingest a PDF, note the collection count, touch the file, re-ingest, confirm the count is unchanged rather than doubled.

---

## P2-4 🔵 Tighten the citation-suppression test

**Problem:** `app.py:245` and `:262` use `"I do not know" not in ai_response` — a substring test. A legitimate answer quoting the phrase loses its citations. (Discrepancy §11.5)

**Change:** `ai_response.strip().rstrip(".").lower() != "i do not know"`, or better, set an explicit `answered = best_distance <= RELEVANCE_THRESHOLD` flag once and branch on that.

---

# PHASE 3 — Offline & security hardening 🟡

## P3-1 🟠 Make the air-gap claim true

**Problem:** ChromaDB telemetry is on by default and disabled nowhere. SDD Appendix E's own log records the outbound attempt: *"Anonymized telemetry enabled… Failed to send telemetry event ClientStartEvent"* — printed inside the document that claims no external requests occur. (Discrepancy §8)

**Change:** at the very top of `modules/database.py`, **before** any chromadb import:
```python
import os
os.environ.setdefault("ANONYMIZED_TELEMETRY", "False")
```
Belt-and-braces: also set it in `.streamlit/config.toml`'s environment or document it in the README run instructions.

**Verify:** run ingestion and confirm no `Anonymized telemetry enabled` line appears in the logs.

---

## P3-2 🟠 Hash the super-admin password

**Problem:** `modules/auth_service.py:36-38` compares the super-admin password in **plaintext**, bypassing the SHA-256 path §9.1 says all credentials use. (Discrepancy §4)

**Change:** store `admin_password_hash` in secrets and compare `self._hash_password(password) == self.admin_password_hash`. Update `secrets.toml.example` from P0-3 and add a one-liner to the README for generating the hash:
```powershell
python -c "import hashlib,sys; print(hashlib.sha256(sys.argv[1].encode()).hexdigest())" "your-password"
```

> While here, note in the docs that SHA-256 without a salt or key-stretching is not a password-hashing function. For a graded offline prototype it is acceptable and matches §7.3 — but §10.3 should say so rather than implying it is best practice. `bcrypt`/`argon2` is the real answer if you want to go further.

---

## P3-3 🟡 Make the optional security checks fail loudly

**Problem:** `modules/auth_service.py:55` and `:59` silently disable the invite-code gate and the domain restriction when their secrets are unset. `README.md:38` presents both as active protections. (Discrepancy §4)

**Change:** log a warning at startup naming each disabled check, and surface a banner in the admin sidebar when any is off. Keep the fail-open default (it keeps first-run usable) but make it visible.

---

## P3-4 🟡 Add the BGE query instruction

**Problem:** BGE retrieval models are trained with a query-side instruction prefix. `modules/database.py:20` uses plain `HuggingFaceEmbeddings`, which does not add it. Retrieval is running below the model's designed performance. (Discrepancy §11.5)

**Change:**
```python
from langchain_community.embeddings import HuggingFaceBgeEmbeddings

self.embedding_model = HuggingFaceBgeEmbeddings(
    model_name="BAAI/bge-base-en-v1.5",
    model_kwargs={"device": "cpu"},          # also lands the SDD §10.2 CPU-pinning claim
    encode_kwargs={"normalize_embeddings": True},
    query_instruction="Represent this sentence for searching relevant passages: ",
)
```

**No re-ingest needed** — the instruction applies to `embed_query` only, not `embed_documents`. But it *does* change retrieval behaviour, so this must land **before** the P4-3 benchmark re-run, and the threshold from P1-2 should be sanity-checked afterwards.

---

# PHASE 4 — Dependencies & reproducibility 🔵

## P4-1 🟠 Resolve the Ragas / LangChain version conflict

**Problem:** `evaluate_rag.py:79-84` builds Ragas 0.1-era columns (`question`, `ground_truth`, `answer`, `contexts`), but `ragas_evaluation_results.csv` has 0.2-era columns (`user_input`, `retrieved_contexts`, `response`, `reference`). `ragas` and `datasets` are unpinned, while `langchain-core==0.1.33` is pinned — modern ragas needs `langchain-core>=0.2`. (Discrepancy §6.3)

**Change:** in a **scratch virtualenv**, not the working one:
1. `pip install ragas==0.2.* ` and record what it forces `langchain-core` to.
2. If it cascades into `langchain` / `langchain-community` bumps, decide whether to bump the whole stack or pin `ragas==0.1.*` to match the existing pins.
3. Pin the winner explicitly in `requirements.txt`, then align `evaluate_rag.py`'s dataset construction to that version's schema.

Do this **before** P4-3 — the re-run is meaningless on an unpinned stack.

---

## P4-2 🔵 Clean up `requirements.txt`

- **Add:** `pandas` (used at `evaluate_rag.py:3`), `langchain-text-splitters` (used at `modules/ingestion.py:5` — pin it rather than relying on transitive resolution from `langchain==0.1.12`).
- **Remove:** `requests==2.31.0` — imported nowhere in the tree.
- **Pin:** `ragas`, `datasets` (from P4-1).
- **Gated on Decision A:** `unstructured[pdf]`, `pdf2image`, `unstructured-inference` — remove under branch A1, keep under A2.

---

## P4-3 🔵 Re-run the benchmark and report it honestly

**Depends on:** P0-4, P1-1…P1-4, P3-4, P4-1. Everything above changes retrieval or evaluation behaviour, so this is the last step.

1. Un-ignore the ground truth so the run is reproducible: `git add -f data/benchmarking_data.csv`, or move it out of `data/` (`.gitignore:6` ignores the whole directory). (Discrepancy §6.6)
2. Re-run `python evaluate_rag.py`.
3. Report **coverage alongside the score.** The current 0.9216 is the mean over **17 of 50** questions; the other 33 are judge failures, not passes. Whatever the new number is, state it as `mean over N of 50, M judge failures excluded`. (Discrepancy §6.1)
4. Expect the gate to actually fire this time — some questions *should* now return "I do not know." That is the fix working, not a regression.
5. Note the two known-unfixable rows unless P-extra below is taken: **Q48** (`privacy@amd.com`) and **Q16** (`infosec.incidents@amd.com`) are destroyed by the PII redactor before embedding. (Discrepancy §7)

---

## P4-4 🟡 Decide what the PII firewall should redact

**Problem:** the email regex at `modules/ingestion.py:32` redacts the published corporate contact addresses that two benchmark questions depend on, and the other three patterns (`+91` mobile, Aadhaar, PAN) are **no-ops on a US policy corpus**. (Discrepancy §7)

**Recommended:** keep all four patterns (they demonstrate REQ-10 and cost nothing), but add an allowlist for published organisational contact addresses:
```python
PII_EMAIL_ALLOWLIST = {"privacy@amd.com", "infosec.incidents@amd.com"}
```
…and skip redaction for exact matches. Then document in SDD §4 and README that the redactor targets *personal* identifiers, not published corporate contact points — which is the actual policy intent, and makes Q16/Q48 answerable.

---

# PHASE 5 — Decision A work (parsing)

**Not needed to reach a working, benchmarked system.** Everything above is complete without it.

### Branch A1 — keep PyMuPDF (docs only)
- Rewrite SDD §1.3, §4 (risk row), §5.2, §8.1 (process flow + code listing), §12 (FR-1.1 row), Appendix A.
- Correct §8.1's chunking table: `by_title`/600/100 → `RecursiveCharacterTextSplitter`/500/50.
- Remove Poppler + Tesseract from §2.2 prerequisites.
- Drop the three unstructured deps (P4-2).

### Branch A2 — implement unstructured.io
- Rewrite `modules/ingestion.py` around `partition_pdf(strategy="hi_res", infer_table_structure=True, chunking_strategy="by_title", max_characters=600, ...)`.
- Install Poppler + Tesseract; add a PATH check with a clear error.
- Preserve the incremental registry (P2-1) and PII redaction — the SDD's §8.1 listing has neither.
- Delete `chroma_db/`, full re-ingest, re-run P4-3.
- Expect a large latency increase on CPU; measure it and update §2.4.

---

# PHASE 6 — Decision B work (streaming)

### Branch B1 — wire up streaming
- `app.py:240-254`: replace the `generate_response` call with `st.write_stream(...)` over `generate_response_stream` (`modules/generation.py:58`, already written and working).
- Capture `t_first_token` on the first yielded chunk; render **TTFT** *and* total, matching §9.3.
- Render citations after the stream completes.
- Note `generate_response` is still used by `evaluate_rag.py` — **keep both.**

### Branch B2 — drop the claims
- Delete `modules/generation.py:58-77`.
- Remove the streaming/TTFT language from SDD §2.4, §4 (risk row), §8.3 (Output), §9.3 (Latency Counter).
- Relabel the UI metric as "Processing Latency" consistently.

---

# PHASE 7 — Documentation reconciliation 🔵

**Do this last**, once code behaviour is settled — otherwise it gets written twice. Full itemised list is in the discrepancy report's *"Mechanical documentation corrections"*. Grouped here:

**SDD — factual corrections**
- bge-small → **bge-base**; 384 → **768** dims (§3.3, §6.1, §7.2, §10.2, App A).
- ms-marco-MiniLM-L-6-v2 → **bge-reranker-base** (§6.3, §8.2, App A).
- Cosine → **squared L2**, with the 0.425 cosine-similarity equivalence (§3.2, §4, §6.1) — from P1-2.
- `initial_k = 10` → `top_k * 3` = **9** (§5.2, §8.2).
- `page_number` → `page`; `password_hash` → `password`; username key → **email** (§7.2, §7.3).
- `extract_and_chunk_pdf()` → `extract_and_chunk()` (§12).
- §8.2's listing: replace with the shipped `_load_db` caching version — the code is better than the design here (§10.2 contradiction).

**SDD — honesty corrections**
- §11.2: restate 0.9216 (or its replacement) with coverage and excluded failures.
- §11.2 vs §12: reconcile Faithfulness "Planned" against "Meets completely."
- §11.2 DT-04 (+14%) and DT-03 (100%): either build the harness that produces them or remove the rows.
- §2.3/§10.3: add the first-run HuggingFace model download caveat (~700MB).
- §9.2/§9.3: remove the file uploader, telemetry pane and citation expander, or build them.
- Document the ingestion registry, invite code, domain restriction and super-admin path.
- Cover page → **Version 3.0**; retitle §11.2 (it is not "Phase 1 Baseline"); reconcile Appendix C against `requirements.txt`; name the AMD corpus in Appendix F.

**README — it needs the most work**
- **Close the code fence at line 41** — there is exactly one fence marker, so everything after renders as a code block.
- **Add setup and execution instructions.** Line 65 calls it a "Setup and execution guide" and it contains none: `pip install -r requirements.txt`, `ollama pull llama3.2`, `ollama serve`, `streamlit run app.py`, plus the secrets.toml step from P0-3.
- Remove **"Qwen"** (line 59) and **"Firebase"** (line 16) — neither exists in this project.
- **Add the cross-encoder** to the retrieval description (lines 18-22) — it omits the flagship Phase 2 feature entirely.
- Add the four missing files to the tree: `evaluate_rag.py`, `auth_service.py`, `data/benchmarking_data.csv`, `ragas_evaluation_results.csv`.
- Add a section on the Ragas evaluation — the README never mentions it.
- Fix bge-small → bge-base (lines 16, 20); add `base = "light"` to `config.toml` or correct line 45.
- Reconcile "v4.0" against the SDD's 3.0.

**Requirement IDs — unify the three schemes**
SDD §12 uses `FR-x.x`; README uses `Req-nn`/`Mn`; code comments use `REQ-nn`; there is no mapping. `REQ-07` currently labels six unrelated features and `REQ-02` labels two. Extend the §12 table with a column mapping each `FR-x.x` to its `REQ-`/`M` label, then make the code comments and README agree.

---

# Verification checklist

Run top to bottom after Phase 3. All must pass before Phase 4's re-run.

- [ ] `python -c "from modules.auth_service import LocalAuthService"` succeeds
- [ ] `streamlit run app.py` reaches the login screen with no `st.error` banner
- [ ] Register + sign in works from a **deleted** `config/users.json`
- [ ] Register + sign in works from a **0-byte** `config/users.json`
- [ ] Query with `chroma_db/` holding only `.gitkeep` → "Admin must trigger Ingestion first"
- [ ] Ingestion indexes the AMD PDF; sidebar reports a non-zero count
- [ ] Re-triggering ingestion with no file changes indexes **0** new chunks
- [ ] Touching the PDF and re-ingesting leaves the collection count **unchanged** (not doubled)
- [ ] A real policy question returns a grounded answer **with** citations
- [ ] "What is the capital of Mars?" returns exactly `I do not know.` ← the gate, currently broken
- [ ] No `Anonymized telemetry enabled` line in the logs
- [ ] `grep -rn "1\.15" *.py modules/` shows only the constant definition
- [ ] `python evaluate_rag.py` runs to completion and writes the CSV

---

# Suggested session split

| Session | Tasks | Outcome |
|---|---|---|
| **1** | P0-1 → P0-4, P1-1 → P1-4 | App starts, logs in, and the relevance gate actually works |
| **2** | P2-1 → P2-4, P3-1 → P3-4 | Ingestion is safe to re-run; air-gap claim is true |
| **3** | P4-1 → P4-4 | Pinned deps, honest benchmark numbers |
| **4** | Decision A + B, then Phase 5/6 | Whichever branches we pick |
| **5** | Phase 7 | SDD + README reconciled against final behaviour |

Sessions 1–3 need **no decisions** and leave the project in a genuinely working, defensible state. If time is short before submission, stopping after session 3 is a reasonable place to land — the remaining items are scope choices, not defects.

---

## Quick start for next session

> Open `REMEDIATION_PLAN.md` and start at **P0-1**. Do not decide anything yet — Phases 0–4 need no decisions. The first goal is the app starting and "what is the capital of Mars?" returning `I do not know.`
