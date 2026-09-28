# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout (important — nested git repos)

This top-level folder (`law-assistant` root) and `backend/` are **two separate git repositories** — `backend/` is a plain nested clone (not a registered `.gitmodules` submodule, so treat it as a submodule-like boundary anyway):

- **Root repo**: only really owns `.github/workflows/law_crawler.yml` (a daily cron that runs `backend/crawler.py` and auto-commits new files into `backend/data/laws/`) plus loose top-level scratch files (`test_kcsc*.py`, `.env` for the KCSC ingestion scripts).
- **`backend/` repo** (remote: `github.com/ksh8481-lgtm/law-assistant`): the actual Flask application — this is where nearly all real work happens.

Always check `git status`/`git log` from inside `backend/` when working on app code — commands run at the root will not show `backend/`'s history.

The `backend/` directory is also very flat and has accumulated many one-off patch/debug scripts at its root (`patch_*.py`, `fix_*.py`, `modify_*.py`, `add_*.py`, `debug*.py`, `test_*.txt`, `old_app.py`, etc.) — these were ad-hoc migration/patch scripts, not part of the running app. The live app only needs: `app.py`, `rule_engine.py`, `kcsc_mcp.py`, `moleg_mcp.py`, `mcp_agent_sync.py`, `templates/`, `static/`, `data/`.

## Running the app

```bash
cd backend
pip install -r requirements.txt
python app.py          # dev server on http://localhost:5000 (debug=True)
```

Production (Procfile, deployed on Cloudtype): `gunicorn app:app`.

There is no test runner / test suite configured — the root-level `test_kcsc*.py` and `backend/test_*.py` files are standalone manual scripts (run individually with `python <file>.py`), not pytest tests.

### Required environment variables (`backend/.env`, see `backend/.env_example`)

- `GEMINI_API_KEY` — required; without it, `/api/analyze/start` and other AI endpoints return an explicit "no key configured" error.
- `VWORLD_API_KEY`, `LAW_API_KEY` — optional; `app.py` falls back to base64-decoded hardcoded defaults (`_V`, `_L` constants near the top of `app.py`) if these are unset, so the app functions without them being set explicitly.

The root-level `.env` (used only by `backend/kcsc_pipeline.py`, `backend/kcsc_ingestion.py`, `backend/init_vectordb.py` when run from repo root) additionally needs `KCSC_API_KEY`.

## Architecture

**`backend/app.py`** is a single-file Flask app (~1500 lines) serving both server-rendered pages (Jinja templates in `templates/`) and a JSON API consumed by `static/*.js`. There are five main features, each with its own template/route pair: `law_review`, `duty_list` (감리자 실무 체크리스트), `design_review` (AI 설계도서 검토), `other_review` (기타 행정 검토), `report`.

**Async job pattern**: any endpoint that calls the Gemini API (which can take >30s) does *not* block the request. It spawns a `threading.Thread`, stores progress in the in-memory `JOBS` dict keyed by a `uuid`, and returns the `job_id` immediately (`/api/analyze/start`, `/api/analyze/other_review`, `/api/analyze/design_review`). The frontend polls a matching `/api/analyze/*/status/<job_id>` endpoint until `status` is `completed`/`error`, at which point the job is deleted from `JOBS`. Because `JOBS` is a plain in-process dict, it does **not** survive a process restart and is not safe if the app is ever run with multiple gunicorn worker processes (`Procfile` currently runs a single default worker).

**Two independent knowledge-retrieval subsystems, both branded "MCP" but structurally different:**
- `kcsc_mcp.py` — *not* a real MCP server. It's a plain Python class (`KCSCMCPEngine`) doing keyword scoring over `data/kcsc_standards.json` (KCSC 국가건설기준 표준시방서/설계기준 data). This is what `app.py` actually imports and calls synchronously for design-review context building.
- `moleg_mcp.py` — a real `FastMCP` server exposing law/precedent search tools (법제처 API) over SSE on port 8000. `mcp_agent_sync.py` spawns it as a subprocess on demand, connects with an MCP `ClientSession`, wraps its tools with `langchain_core.tools`, and drives them through a `langgraph` `create_react_agent`. Since Flask routes are sync but the MCP client is async, `get_mcp_context_sync()` bridges it with `asyncio.run(...)` inside the worker thread (with a Windows `ProactorEventLoopPolicy` workaround).

**`rag_app.py`** is a separate, standalone Streamlit app (not wired into `app.py`/gunicorn at all) that combines a Chroma vector store (`chroma_db/`, built by `init_vectordb.py` from `data/kcsc_standards.json` using Gemini embeddings) with the same `moleg_mcp.py` SSE server. Treat it as an experimental/alternate UI, not part of the production request path.

**`rule_engine.py`** evaluates the 감리자 체크리스트: `law_knowledge_base.json` holds rules with a `condition` field that is a Python boolean expression string (e.g. referencing project parameters); `evaluate_knowledge_base(params)` runs `eval()` on each condition with `__builtins__` stripped and a `SafeDict` (missing vars default to `False`) so unknown/omitted params don't crash the eval — only add new rule variables by extending `law_knowledge_base.json`, not by hand-parsing conditions elsewhere.

**Parcel/GIS lookups** (`/api/regions/*`, `/api/verify_parcel`, `/api/extract_parcel_from_drawing`) call the VWorld API directly via `requests`; address matching in `verify_parcel` does manual token comparison with a hardcoded province-abbreviation map (e.g. 경북 ↔ 경상북도) since VWorld's API doesn't normalize these.

**Data pipeline for KCSC standards**: `kcsc_pipeline.py` / `kcsc_ingestion.py` crawl the KCSC OpenAPI (`kcsc.re.kr/OpenApi/...`, `verify=False` because of Korean gov SSL certs) into `data/kcsc_standards.json`, which both `kcsc_mcp.py` (keyword search) and `init_vectordb.py` → `chroma_db/` (vector search, Streamlit app only) consume. `data/laws/*.md` is a separate corpus of full law/guideline text kept up to date by the root repo's daily `law_crawler.yml` GitHub Action running `crawler.py`.

**Deployment**: production runs on Cloudtype at `port-0-law-assistant-mpgbhts4cec2050f.sel3.cloudtype.app`; `backend/.github/workflows/keepalive.yml` pings it every 15 minutes to prevent cold-start/sleep.
