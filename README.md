# EduContinuum — Learn • Grow • Belong

## Frontend-first development

The React/Vite frontend prototype is in `frontend/`. It includes the visual three-stage learning journey, competency snapshot, evidence timeline, pathway discussion prompts, and teacher-reviewed report preview. It currently uses fictional in-browser demo state and is not connected to the Python backend. See `frontend/README.md` for setup and limitations.

Run the frontend from that directory with `npm install` followed by `npm run dev` (Node.js 18+). Backend integration comes after the UI is reviewed.


**A hackathon prototype for longitudinal learner evidence, family-friendly reporting and teacher-led CBC/CBE pathway conversations. Use fictional data only.** It is not an official Ministry of Education product and does not assign a learner to a pathway.

## What it does

- Tracks learner evidence across Lower Primary (Grades 1–6), Junior Secondary (Grades 7–9), and Senior Secondary (Grades 10–12).
- Displays a visual journey map and the three Senior Secondary pathway areas: STEM, Social Sciences, and Arts & Sports Science.
- Stores teacher-entered evidence and retains evidence IDs for traceability.
- Flags a category/subject group for review only when it has at least three evidence records across at least two year/term periods.
- Uses a LangGraph pipeline with an evidence-analysis node and a summary-drafting node.
- Uses a locally hosted open-weights model through Ollama (`qwen2.5:7b`) by default; `AI_MODE=mock` is available for deterministic smoke tests and is clearly marked as mock.
- Provides a custom MCP server with five tools: `ingest_evidence`, `update_longitudinal_profile`, `get_holistic_profile`, `draft_plain_summary`, and `request_human_approval`.
- Includes an adapter/configuration for the community filesystem MCP server, sandboxed to `data/demo_workspace`.
- Records MCP action inputs (excluding full text where practical), outputs, status, actor and timestamps in the prototype audit table.
- Draft reports remain PENDING until a teacher approves or rejects them in the app. No automatic guardian delivery exists.


## HTTP API service

The Streamlit interface remains available, and the first HTTP API layer now lives in `backend/api.py`. It reuses the existing database and evidence-analysis pipeline rather than creating a second data store.

Start the API in a separate PowerShell terminal from the project root, with the virtual environment active:

```powershell
python -m pip install -r .\requirements.txt
python -m uvicorn backend.api:app --host 127.0.0.1 --port 8000
```

Open `http://127.0.0.1:8000/docs` for the interactive API documentation. The API is bound to localhost by default and uses HTTP Basic authentication with the project's demo accounts. Change the demo passwords in `.env` before testing. The health check at `/health` is public; other `/v1` routes require authentication.

Initial endpoints:
- `GET /v1/me` — current authenticated account and role.
- `GET /v1/learners/{learner_id}/evidence` — read evidence.
- `POST /v1/evidence` — add evidence (teacher/admin).
- `GET /v1/learners/{learner_id}/patterns` — run the existing analysis pipeline and return review prompts.
- `POST /v1/reports/drafts` — create a pending draft (teacher/admin).
- `GET /v1/learners/{learner_id}/reports` — list drafts/reports.
- `POST /v1/reports/{report_id}/decision?decision=APPROVED` (or `REJECTED`) — record teacher review. This does not send the report.

Security boundaries: this is a development integration, not a production-secure API. Demo authentication is not an identity provider, the request limiter is in-memory and per-process, and learner-level/school-level authorization, TLS/reverse-proxy configuration, centralized rate limiting, token/session management, hardened audit storage, deployment threat modelling, and independent security testing remain required before any real learner data or network exposure. Do not bind to `0.0.0.0` or expose this service publicly at this stage. Continue to use fictional data only.

Run tests with:

```powershell
python -m pytest -q
```

## Important boundaries

The system **must not** rank learners, label them, predict a career, or automatically recommend/assign a school pathway. The visual pathway graphic is explanatory, not a suitability score. Official placement and curriculum decisions remain governed by current Ministry of Education/KICD guidance and learner/family conversations. Every claim about a learner should cite original evidence; model-generated citations are not yet automatically validated, so teachers must check them.

## Quick start

Requirements: Python 3.11 recommended, Node.js/npm (only for the borrowed filesystem MCP server), and Ollama for local model inference.

### 1. Install

```bash
python -m venv .venv
# Windows: .venv\\Scripts\\activate
# macOS/Linux:
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env
```

Set `DB_ENCRYPTION_KEY` to a long random secret and change demo passwords in `.env`. Install and start Ollama, then:

```bash
ollama pull qwen2.5:7b
streamlit run app.py
```

The default database mode is SQLCipher and intentionally fails closed if the compatible driver/key is missing. SQLCipher driver installation varies by platform. For a **fictional-data-only UI smoke test**, set `DB_ENCRYPTION=sqlite` in `.env`; that mode is unencrypted and must never be used for real learner records.

### 2. Run tests

```bash
pytest -q
```

See `EVALS.md` for the test task catalogue and known failure. Report the actual test output in the hackathon submission; do not claim unrun tests passed.

### 3. Run the custom MCP server

In a separate terminal, from the repository root with the virtual environment active:

```bash
python -m backend.mcp_server
```

The server exposes five tools. The approval tool only records a pending request; actual approval happens in the teacher UI. Never run multiple MCP servers against the same SQLite/SQLCipher database unless you have checked concurrency and deployment behaviour.

### 4. Try the borrowed MCP integration

The project uses the community `@modelcontextprotocol/server-filesystem` as its borrowed server because it provides standard file operations without us writing another filesystem tool. It is restricted to `data/demo_workspace`, which must contain fictional demo artifacts only.

```bash
# Requires Node.js/npm and network access for npx the first time
python -m backend.borrowed_mcp_client
```

`mcp-servers.json` records the custom and borrowed server commands for compatible MCP clients. The borrowed server adapter is a separate, callable integration example; it is not silently invoked by every Streamlit page.

## Demo credentials

Defaults are `teacher / TeacherDemo!2026` and `admin / AdminDemo!2026`. Override them with `DEMO_TEACHER_PASSWORD` and `DEMO_ADMIN_PASSWORD` in `.env` before demos. These are hackathon-only credentials, not production authentication.

## Suggested fictional evidence

Use learner ID `DEMO-AISHA` and fictional learner name `Aisha (fictional demo learner)`:

- 2023 Term 1, Academic, Mathematics: Solved 4 of 10 multiplication questions independently.
- 2023 Term 2, Academic, Mathematics: Solved 5 of 10 multiplication questions independently.
- 2024 Term 1, Academic, Mathematics: Solved 6 of 10 multiplication questions with fewer prompts.
- 2024 Term 2, Attendance, General: Present for 42 of 45 school days.
- 2025 Term 1, Participation, Reading: Volunteered to read aloud twice.

## Repository map

- `app.py` — Streamlit UI, visual learning-stage graphic, evidence entry, timeline, report approval and audit view.
- `backend/agent.py` — LangGraph evidence analysis and local summary draft.
- `backend/mcp_server.py` — custom MCP server with write/read/profile/report/approval-request tools.
- `backend/borrowed_mcp_client.py` — client adapter for community filesystem MCP server.
- `backend/db.py` — prototype data access and audit persistence.
- `backend/auth.py` — demo-only credentials.
- `backend/security.py` — SQLCipher key handling.
- `ARCHITECTURE.md` — agent flow and MCP choices.
- `EVALS.md` and `tests/` — evaluation tasks and executable tests.

## Before any real-school pilot

This is a starter MVP, not a production-secure system. Authentication is demo-grade; audit rows are not tamper-proof; database encryption must be verified in the actual runtime; key backup/recovery, access control, retention, consent, transfer authorization, secure guardian delivery, data-protection impact assessment, threat modelling and independent testing are not complete. Obtain explicit consent for real learner data and comply with applicable Kenyan data-protection requirements. Do not use actual learner records in the hackathon demo.
