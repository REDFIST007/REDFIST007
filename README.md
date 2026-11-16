# School Operations Agent Platform (SIM Senior Engineering Assignment)

A production-shaped, role-aware **School Operations Agent Platform** supporting multi-tenant school onboarding, scoped invites, chat identity linking (Telegram & WhatsApp canonical envelope + built-in simulator), document ingestion with prompt-injection defense and human-in-the-loop approval, intent routing, policy-aware reminders, live SSE dashboards, and a privacy-safe audit/trace timeline.

## Quickstart (Local Run)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --port 8000
```

Open `http://localhost:8000` in your browser.

### Run Automated Tests & Evals
```bash
python3 -m unittest discover -s tests -v
```

## Architecture Overview

```
+-----------------------+       +-----------------------------------+
|  Web App (HTML + SSE) |       | Telegram / WhatsApp / Simulator   |
+-----------+-----------+       +-----------------+-----------------+
            |                                     |
            v                                     v
+-----------+-------------------------------------+-----------------+
|               FastAPI API & Security Boundary                     |
|  - scrypt + HMAC-SHA256 Sessions (server-side revocation)         |
|  - Role & Resource-Scoped Guards (School / Class / Student)       |
|  - Idempotency Store (webhooks, registrations, submissions, SHA)  |
+-----------+-------------------------------------+-----------------+
            |                                     |
            v                                     v
+-----------+-----------+       +-----------------+-----------------+
|   Document Parser     |       |          Intent Layer             |
| - PDF/DOCX/CSV/Img/Txt|       | - 6 Intent Categories +           |
| - Injection Defense   |       |   Unknown/Unsafe Fallback         |
| - Chunking & Q&A      |       | - Strict Deterministic Policy     |
| - Review & Approve    |       |   Gate Before Any Tool Execution  |
+-----------+-----------+       +-----------------+-----------------+
            |                                     |
            +------------------+------------------+
                               |
                               v
+------------------------------+------------------------------------+
|        SQLite Persistence + Restart-Safe Reminder Engine          |
|  - Explicit State Machines (Assignment, Target, Document)         |
|  - Quiet Hours, Blocked Escalations, Submitted Skip               |
|  - Correlation-ID Audit Events & Live SSE Pub/Sub                 |
+-------------------------------------------------------------------+
```

## Domain & Access Model

- **Entities**: `schools`, `classes`, `users`, `teacher_classes` (M:N), `students`, `guardians`, `student_guardians` (M:N), `invites`, `sessions`, `chat_links`, `documents`, `document_chunks`, `school_policies`, `assignments`, `assignment_targets`, `submissions`, `feedback`, `reminders`, `audit_events`, `idempotency_keys`, `feature_flags`.
- **Access Boundaries**:
  - **Admin / Coordinator**: Full control within their own `school_id` only.
  - **Teacher**: Scoped strictly to `teacher_classes` assignments.
  - **Student**: Scoped strictly to own `student_id` assignments, submissions, and feedback.
  - **Parent / Guardian**: Scoped strictly to `student_guardians` linked children with high-level progress only.
  - **System / Agent**: Must act through explicit deterministic service functions and policy checks.

## Parser Strategy & Safety

1. **Multi-Format Extraction**: Supports PDF (`pypdf`), DOCX (`python-docx`), CSV (`csv.DictReader`), Images (`Pillow` metadata), and plain text.
2. **Prompt-Injection Defense**: Scans untrusted text for override/exfiltration patterns, quarantines matched lines, forces `injection_detected=1`, drops confidence, and blocks auto-approval.
3. **Ambiguity & Review Gate**: Missing/ambiguous fields (e.g., relative due dates, duplicate roster rows, missing class) transition the document to `needs_clarification` with explicit `follow_up_questions`. Nothing mutates `assignments`, `students`, or `school_policies` until reviewed and approved.

## Secret Rotation & Security Assumptions

- Secrets are never committed to git (`.env` is gitignored; `.env.example` documents required variables).
- Rotating `APP_SECRET_KEY` invalidates active HMAC-SHA256 session tokens immediately, forcing clean re-authentication.
- Webhook secrets (`TELEGRAM_WEBHOOK_SECRET`, `WHATSAPP_VERIFY_TOKEN`) are verified at the router boundary.

## Known Limitations & Productionization Path

- **Database**: Currently uses SQLite with WAL mode for zero-setup portability; productionizing requires swapping `app/db.py` to PostgreSQL with connection pooling (`psycopg` / `asyncpg`).
- **Scheduler**: Currently runs an in-process `asyncio` loop; multi-worker production deployments should move reminder cycles to a distributed worker (e.g., Celery/ARQ/Cloud Tasks) with DB row locking.
- **OCR / Voice**: Image uploads extract dimensions/metadata cleanly; productionizing adds Tesseract/Cloud Vision OCR and Whisper transcription behind the same `extract_text_from_bytes` boundary.
- **SSE Broker**: In-process `asyncio.Queue` pub/sub; multi-instance deployments should back `SSEBroker` with Redis Pub/Sub.

## Architecture Decision Records (ADRs)

- [ADR-001: Deterministic Rules vs. LLM Boundaries](docs/adr/001-deterministic-rules-vs-llm-boundaries.md)
- [ADR-002: Canonical Channel Envelope and Idempotency](docs/adr/002-canonical-channel-envelope-and-idempotency.md)
- [ADR-003: SQLite Repository and Restart-Safe Scheduler](docs/adr/003-sqlite-repository-and-restart-safe-scheduler.md)
- [One-Page Threat Model](docs/THREAT_MODEL.md)
