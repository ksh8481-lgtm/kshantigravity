# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout (important — nested git repos)

This top-level folder (`law-assistant` root) and `backend/` are **two separate git repositories** — the root repo records `backend/`'s commit as a gitlink pointer (there is no `.gitmodules`), so treat it as a submodule-like boundary:

- **Root repo**: only really owns `.github/workflows/law_crawler.yml` (a daily cron that runs `backend/crawler.py` and auto-commits new files into `backend/data/laws/`) plus loose top-level scratch files (`test_kcsc*.py`, `.env` for the KCSC ingestion scripts).
- **`backend/` repo** (remote: `github.com/ksh8481-lgtm/law-assistant`): the actual Flask application — this is where nearly all real work happens.

Always check `git status`/`git log` from inside `backend/` when working on app code — commands run at the root will not show `backend/`'s history.

**Push order**: commit and push in `backend/` first, verify with `git ls-remote origin refs/heads/main` that the remote hash equals the local `HEAD` (a push once silently did not land), then in the root `git add backend` and commit (`backend <old> -> <new>`) and push. Do not bundle unrelated root changes (e.g. the pending `law_crawler.yml` deletion) into the pointer commit. `git pull --rebase origin main` is often needed first because the crawler bot pushes to `backend/` daily; rebasing changes your commit hashes.

The `backend/` directory is also very flat and has accumulated many one-off patch/debug scripts at its root (`patch_*.py`, `fix_*.py`, `modify_*.py`, `add_*.py`, `debug*.py`, `test_*.txt`, `old_app.py`, etc.) — these were ad-hoc migration/patch scripts, not part of the running app. The live app only needs: `app.py`, `rule_engine.py`, `kcsc_mcp.py`, `moleg_mcp.py`, `mcp_agent_sync.py`, `templates/`, `static/`, `data/`.

## Running the app

```bash
cd backend
pip install -r requirements.txt
python app.py          # dev server on http://localhost:5000 (debug=True)
```

Production (Procfile, deployed on Cloudtype): `gunicorn app:app`.

Tests are plain scripts (no pytest; run each with `python <file>.py` from `backend/`):

- `test_admrul.py` (49 tests, `unittest`) — 고시/별표 retrieval, citation verification, follow-up grounding, Gemini wrapper. Tests that hit the 법제처 API are skipped when it is unreachable or when `SKIP_ONLINE_TESTS=1`; they depend on the current text of the 고시, so a 고시 amendment can legitimately break them.
- `test_gemini_endpoints.py` (12 tests) — each endpoint's call into Gemini (model, contents format, config) against a fake client; no network or key needed.
- `test_pq_*.py`, `test_injection.py` — older offline scripts. The other `test_*.py` files (vworld, kcsc, etc.) are manual scripts that need a running server or external APIs.

A daily GitHub Action (`dependency_canary.yml`) installs the latest dependencies and runs these offline.

### Required environment variables (`backend/.env`, see `backend/.env_example`)

- `GEMINI_API_KEY` — required; without it, `/api/analyze/start` and other AI endpoints return an explicit "no key configured" error.
- `GEMINI_MAIN_MODEL` (default `gemini-2.5-pro`), `GEMINI_FAST_MODEL` (`gemini-2.5-flash`), `GEMINI_MAIN_FALLBACKS`, `CITATION_VERIFY_MODE` (`off` default | `pro` | `flash`) — optional Gemini switches, see `gemini_client.py` and `backend/docs/운영안내.md`.
- `VWORLD_API_KEY`, `LAW_API_KEY` — optional; `app.py` falls back to base64-decoded hardcoded defaults (`_V`, `_L` constants near the top of `app.py`) if these are unset, so the app functions without them being set explicitly.

The root-level `.env` (used only by `backend/kcsc_pipeline.py`, `backend/kcsc_ingestion.py`, `backend/init_vectordb.py` when run from repo root) additionally needs `KCSC_API_KEY`.

## Architecture

**`backend/app.py`** is a single-file Flask app (~2900 lines) serving both server-rendered pages (Jinja templates in `templates/`) and a JSON API consumed by `static/*.js`. Main features, each with its own template/route pair: `law_review`, `duty_list` (법정 의무 마스터; body parsing/chunking/article-number verification live in `duty_text.py`), `design_review` (AI 설계도서 검토), `other_review` (기타 행정 검토), `report`, `pq_score` (사업수행능력평가 점수 산정, `pq_engine.py` + `pq_specs/*.json`; its step 6 is the 적격심사 종합평점·낙찰자 panel `static/award_panel.js` backed by `award_engine.py` + `award_specs/*.json` - it is a step of this page, not a separate page; `/award` just redirects here).

**Async job pattern**: any endpoint that calls the Gemini API (which can take >30s) does *not* block the request. It spawns a `threading.Thread`, stores progress in the in-memory `JOBS` dict keyed by a `uuid`, and returns the `job_id` immediately (`/api/analyze/start`, `/api/analyze/other_review`, `/api/analyze/design_review`). The frontend polls a matching `/api/analyze/*/status/<job_id>` endpoint until `status` is `completed`/`error`, at which point the job is deleted from `JOBS`. Because `JOBS` is a plain in-process dict, it does **not** survive a process restart and is not safe if the app is ever run with multiple gunicorn worker processes. `backend/gunicorn.conf.py` (auto-loaded by gunicorn from the working directory; the Procfile options are not applied on Cloudtype) runs **one worker with 8 threads (gthread)** so requests are handled concurrently - keep it at one worker for that reason.

**Two independent knowledge-retrieval subsystems, both branded "MCP" but structurally different:**
- `kcsc_mcp.py` — *not* a real MCP server. It's a plain Python class (`KCSCMCPEngine`) doing keyword scoring over `data/kcsc_standards.json` (KCSC 국가건설기준 표준시방서/설계기준 data). This is what `app.py` actually imports and calls synchronously for design-review context building.
- `moleg_mcp.py` — law/precedent/ordinance/고시 retrieval against the 법제처 API. It still declares `FastMCP` tools (used by `rag_app.py`, which spawns it as a subprocess), but the Flask app imports and calls its plain functions directly. Key functions: `search_laws_with_text` (법령 조문, `target=law`), `search_admrul_with_text` (고시/지침 조문 **and 별표**, `target=admrul`; exact-name current version only, 별표 split by 호/목 and ranked against the question), `search_ordinance`, precedent search.
- `mcp_agent_sync.py` — despite the name there is **no MCP client or agent loop anymore**. `get_mcp_context_sync()` uses Gemini flash only to extract keywords / law names / 고시 names (the three extractions run concurrently), then calls the `moleg_mcp` functions directly and passes the raw results to the main prompt. `_ADMRUL_HINTS` deterministically picks 고시 by words in the question (max 3).

**`rag_app.py`** is a separate, standalone Streamlit app (not wired into `app.py`/gunicorn at all) that combines a Chroma vector store (`chroma_db/`, built by `init_vectordb.py` from `data/kcsc_standards.json` using Gemini embeddings) with the same `moleg_mcp.py` SSE server. Treat it as an experimental/alternate UI, not part of the production request path.

**`rule_engine.py`** evaluates the 감리자 체크리스트: `law_knowledge_base.json` holds rules with a `condition` field that is a Python boolean expression string (e.g. referencing project parameters); `evaluate_knowledge_base(params)` runs `eval()` on each condition with `__builtins__` stripped and a `SafeDict` (missing vars default to `False`) so unknown/omitted params don't crash the eval — only add new rule variables by extending `law_knowledge_base.json`, not by hand-parsing conditions elsewhere.

**Parcel/GIS lookups** (`/api/regions/*`, `/api/verify_parcel`, `/api/extract_parcel_from_drawing`) call the VWorld API directly via `requests`; address matching in `verify_parcel` does manual token comparison with a hardcoded province-abbreviation map (e.g. 경북 ↔ 경상북도) since VWorld's API doesn't normalize these.

**Data pipeline for KCSC standards**: `kcsc_pipeline.py` / `kcsc_ingestion.py` crawl the KCSC OpenAPI (`kcsc.re.kr/OpenApi/...`, `verify=False` because of Korean gov SSL certs) into `data/kcsc_standards.json`, which both `kcsc_mcp.py` (keyword search) and `init_vectordb.py` → `chroma_db/` (vector search, Streamlit app only) consume. `data/laws/*.md` is a separate corpus of full law/guideline text kept up to date by the root repo's daily `law_crawler.yml` GitHub Action running `crawler.py`.

**All Gemini calls go through `backend/gemini_client.py`** (the `google-genai` SDK; the old `google.generativeai` is removed and must not come back). It pins models (env-overridable), converts old chat-style contents/file objects, logs `[gemini] label model seconds tokens` per call, and holds a lock-protected shared client (without the lock concurrent first calls fail with "client has been closed"). Thinking is deliberately **kept on** for law-name extraction and citation judging and **limited (budget 512)** for grounded follow-up chat — these were measured, see `backend/docs/작업기록/`.

**Q&A path (`other_review`)**: `run_other_review` → `mcp_agent_sync.get_mcp_context_sync` (법령 + 고시·별표 원문) → main answer (pro) → `law_linkify` (links; fixes 고시 mislinked as `/법령/`; appends a 시행일/현행 footer for each 고시 the answer mentions) → `citation_check` (deterministic checks that cited 조문/별표 exist, that quoted %/terms appear in the cited text, plus an LLM topic judge; it only annotates ⚠️/💡 and never rewrites). The Google-search verification pass is **off by default** on this path. Follow-up chat (`/api/chat/other_review`) uses `qa_grounding.py`: it remembers (in memory, keyed by the hash of `initial_context`) which 법령/고시 the first answer used and re-excerpts them for each follow-up question. It runs synchronously inside one HTTP request, so keep it at ~15-20s (Cloudtype's gateway returns 504 at 60s).

**PQ / 적격심사**: scores are never produced by AI. `pq_engine.py` computes 사업수행능력평가 점수 from `pq_specs/*.json` (고시 [별표] tables); `award_engine.py` takes those PQ scores and computes the 적격심사 종합평점 per 「지방자치단체 입찰시 낙찰자 결정기준」 제2장의2 [별표 1] (rules and citations in `award_specs/jibang_jeokgyeok_tech.json`; 낙찰자 = lowest price first with 종합평점 ≥ 92/95). `pq_watch.py` (run by the daily `law_crawler.yml`) detects 개정/시행 예정 of every watched 고시·예규 and, for the 낙찰자 결정기준 (PDF-only on 법제처), `award_check.py` diffs the spec's numbers against the new PDF so the issue lists only changed values. Procedure for an amendment: `backend/docs/운영안내.md` §10.

**Deployment**: production runs on Cloudtype at `port-0-law-assistant-mpgbhts4cec2050f.sel3.cloudtype.app`; `backend/.github/workflows/keepalive.yml` pings it every 15 minutes to prevent cold-start/sleep.

## Working agreement: keep the record

After finishing work in `backend/`, update the docs in the same change set (never put secrets in them):

- `backend/docs/CHANGELOG.md` — add one line at the top (date, what changed for users, commit, deploy status). Use the commit hash that exists **after** `git pull --rebase`/push.
- `backend/docs/작업기록/<date>_<topic>.md` — for non-trivial work: problem → cause → change → **measurements behind each decision** → what was rejected → limits → how to revert.
- `backend/docs/운영안내.md` — update when a switch, log line, check procedure or failure mode changes.
- Verify what you claim: measure before choosing a model/thinking setting, run the tests, and confirm pushes by remote hash.
