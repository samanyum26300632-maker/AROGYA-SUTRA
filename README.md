# AROGYA-SUTRA Backend
FastAPI backend prototype aligned to `adityaraj3532/Asha` and SIH26133.

## Architecture
`app/api/v1` REST routers -> `app/services` domain logic -> `app/models` SQLAlchemy async persistence. PostgreSQL is the intended DB; SQLite is the zero-setup fallback. Redis is provisioned for future cache/queue use; this prototype deliberately uses FastAPI BackgroundTasks/on-demand recomputation instead of Celery.

## Phase 1 — Foundation
- Layered FastAPI project, async SQLAlchemy 2, Pydantic v2, `.env`, CORS, structured JSON logs.
- Docker Compose starts API + Postgres + Redis.
- `/health` and `/docs` are immediately available.

Run locally:
```bash
cp .env.example .env
python -m venv .venv
# Windows: .venv\\Scripts\\activate
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
curl http://localhost:8000/health
```
Docker: `docker compose up --build`.

## Phase 2 — Auth & RBAC
Roles: `ASHA_WORKER`, `ANM`, `DOCTOR`, `PHC_ADMIN`, `DISTRICT_ADMIN`. Accounts are admin-provisioned. ASHA list scope is restricted to their assigned village in the patient endpoint; doctor/admin-only routes enforce RBAC.

Seed and login:
```bash
python -m scripts.seed
curl -X POST http://localhost:8000/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"doctor@demo.in","password":"Demo@123"}'
```
Demo emails: `asha@demo.in`, `anm@demo.in`, `doctor@demo.in`, `phcadmin@demo.in`, `district@demo.in`; password `Demo@123`.

## Phase 3 — Offline-first Sync
`POST /api/v1/sync/batch` accepts up to 500 records. Each item carries `client_id`, `client_updated_at`, and `version`. Retrying a `client_batch_id` is idempotent. Conflicts use last-write-wins: if server `updated_at >= client_updated_at`, the client write is marked `CONFLICT` rather than silently overwriting. Clinical items become `PENDING_REVIEW`.

The frontend's legacy action names are accepted: `REGISTER_PATIENT`, `BOOK_APPOINTMENT`, `UPDATE_VITALS`, `RECORD_FOLLOWUP`. For production, change its timestamp-style IDs to `crypto.randomUUID()`.

Example:
```bash
curl -X POST http://localhost:8000/api/v1/sync/batch -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"client_batch_id":"2a4a5a2d-1e19-4c48-a2ee-fb477838568a","items":[{"entity_type":"REGISTER_PATIENT","client_id":"58d51c8d-50ef-48d6-aa29-078f1c324ca4","client_updated_at":"2026-08-30T12:00:00Z","version":1,"payload":{"name":"Rukmini Atram","age":42,"gender":"Female","phone":"9000000002","village_id":"REPLACE_WITH_SEEDED_VILLAGE_UUID"}}]}'
```
Status: `GET /api/v1/sync/batches/{client_batch_id}`.

Tradeoff: LWW is understandable and demo-friendly but can discard simultaneous field-level edits. Production should add field-level merge rules/event logs, and consider CRDTs only for genuinely concurrent collaborative records.

## Phase 4 — Clinical Triage
Models: Patient, Visit, VitalSigns, TriageAssessment. The `TriageEngine` is a swappable, deterministic rules service. It scores danger flags, oxygen saturation, BP, pulse, respiratory rate, temperature and key symptoms into `LOW/MEDIUM/HIGH/EMERGENCY` with a suggested action. It is decision support, not autonomous diagnosis.

Routes: `POST /clinical/triage`, `GET /clinical/triage/queue`, `POST /clinical/triage/{id}/review`. Doctors can confirm/override urgency and attach notes/prescription JSON.

## Phase 5 — Voice
`POST /api/v1/voice/transcribe` accepts multipart audio + `language=mr|hi|en` and optional `patient_id`.

**MOCKED:** ASR currently returns realistic Marathi/Hindi/English demo text regardless of audio. `# TODO` marks the integration point. Bhashini is the preferred government-aligned ASR integration; Whisper is a fallback. Keyword/entity extraction is also a stub. Both raw transcript and structured extraction are stored for human verification.

```bash
curl -X POST http://localhost:8000/api/v1/voice/transcribe -H "Authorization: Bearer $TOKEN" -F "audio=@sample.wav" -F "language=mr"
```

## Phase 6 — Drug Forecasting
Models: PHC, DrugInventory, DrugConsumptionLog, StockAlert. `POST /inventory/consumption` logs usage and decrements stock. `GET /inventory/at-risk` calculates an explainable recent moving-average forecast and returns the arithmetic reasoning.

**Prototype model:** mean of up to the latest 14 consumption observations, then `days_until_stockout = current_stock / average_daily_consumption`. This is intentionally transparent. Later swap the service for exponential smoothing/Prophet/ML using disease surveillance, seasonality and local events without changing the route contract.

## Phase 7 — Dashboard
`GET /api/v1/dashboard/summary` returns triage counts by urgency, sync review backlog, patient count, health-worker count and stockout risk summary as one read-optimized payload.

## Frontend integration map
The current Asha frontend exposes patient, ASHA, doctor, hospital resource and admin analytics views. Replace imports from `src/data/mockData.ts` incrementally with a small `src/services/api.ts` client. Its TypeScript models already include Patient, Facility, Appointment, Prescription, Referral, MedicineStock, DiagnosticTest, HighRiskPatient and OfflineAction, so keep those UI-facing shapes as adapters while the backend uses normalized records.

Recommended client base URL: `VITE_API_URL=http://localhost:8000/api/v1` (the repo is Vite, despite the original prompt saying Next.js).

## Testing
```bash
pytest -q
```
Critical unit coverage included for JWT/passwords, triage scoring and stockout forecast. Add API integration tests with a disposable Postgres container before production.

## Alembic
For the hackathon prototype, `AUTO_CREATE_TABLES=true` makes first run frictionless. For real deployment set it false and use migrations:
```bash
alembic revision --autogenerate -m "initial"
alembic upgrade head
```

## Mocked vs real
Real: authentication primitives, RBAC checks, async DB persistence, idempotent sync batches, conflict detection, deterministic triage, inventory logging, explainable forecast math, aggregate dashboard queries.

Mocked/stubbed: Bhashini/Whisper ASR; NLP structuring; SMS/call notifications; ABHA creation/verification; production forecasting signals. Redis is provisioned but not yet needed by request paths.

## What to extend first
1. **Connect the existing offline queue to `/sync/batch` and switch IDs to `crypto.randomUUID()`.** This makes the strongest SIH differentiator visibly work during a network-off/network-on demo.
2. **Integrate real Bhashini ASR for one Marathi clinical phrase, while retaining transcript review.** Judges can see vernacular capture is real rather than only UI theater.
3. **Make stock forecasting visually explainable.** Seed 30–60 days per PHC/drug, expose trend + depletion date + confidence/data-quality flags; keep the model simple enough to defend in an audit.

For a production healthcare deployment, add immutable audit logs, consent/ABHA rules, encryption/KMS, rate limiting, device registration, clinical governance, monitoring, backups and a formal threat/privacy review.
