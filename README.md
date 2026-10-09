# Bulk Certificate Generator API

A backend API that accepts **one request containing many recipients**, generates a PDF certificate for each valid recipient from a predefined template, tracks the progress of the job, and lets the client download the results (individually or as a ZIP).

**Stack:** Python 3.10+, FastAPI, SQLAlchemy 2 (SQLite by default, any relational DB works), ReportLab (PDF), pytest.

---

## 1. Setup

```bash
# 1. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate          # Windows (PowerShell): .venv\Scripts\Activate.ps1
                                   # Windows (cmd):        .venv\Scripts\activate.bat

# 2. Install dependencies
pip install -r requirements.txt
```

No database server is needed: SQLite is used by default and the tables are created automatically on startup.

### Configuration (optional)

Environment variables, all with defaults:

| Variable          | Default                          | Meaning                                        |
|-------------------|----------------------------------|------------------------------------------------|
| `DATABASE_URL`    | `sqlite:///./certificates.db`    | Any SQLAlchemy URL, e.g. `postgresql+psycopg://user:pass@localhost/certs` |
| `CERT_DIR`        | `./generated_certificates`       | Where PDFs are stored (one sub-folder per job) |
| `MAX_RECIPIENTS`  | `1000`                           | Maximum recipients accepted in one request     |

## 2. Run the application

```bash
uvicorn app.main:app --reload
```

The API is now on <http://localhost:8000>. Interactive documentation (try every endpoint from the browser): <http://localhost:8000/docs>.

## 3. Run the tests

```bash
pytest
```

Each test uses its own temporary database and folder, so tests never touch real data. 71 tests cover job creation, input validation, PDF generation, progress, individual failures and retrieval.

## 4. Submit a certificate generation request

`POST /jobs` with all recipients in a single call:

```bash
curl -X POST http://localhost:8000/jobs \
  -H "Content-Type: application/json" \
  -d @examples/sample_request.json
```

Request body:

```json
{
  "event_name": "Python Bootcamp 2026",
  "issued_by": "Aereo Academy",
  "issue_date": "2026-10-09",
  "recipients": [
    { "name": "Priya Sharma", "email": "priya@example.com" },
    { "name": "Rahul Verma", "email": "rahul@example.com" },
    { "name": "", "email": "this-row-is-invalid@example.com" }
  ]
}
```

| Field          | Required | Rules                                                   |
|----------------|----------|---------------------------------------------------------|
| `event_name`   | yes      | 1–120 characters                                        |
| `issued_by`    | no       | 1–80 characters                                         |
| `issue_date`   | no       | `YYYY-MM-DD`, defaults to today                         |
| `recipients`   | yes      | 1 to `MAX_RECIPIENTS` entries                           |
| `recipients[].name`  | yes | 1–80 characters, Latin letters (accents are fine), no control characters |
| `recipients[].email` | no  | must look like an email address if given                |

The server answers immediately with **`202 Accepted`** and the job:

```json
{
  "id": "bf7d1fda-39b4-4753-9ac7-2e7f0a6435ad",
  "status": "pending",
  "total": 3, "generated": 0, "failed": 1, "pending": 2,
  "progress_percent": 33.3,
  "certificates_url": "/jobs/bf7d1fda-.../certificates",
  "download_url": null
}
```

(Recipients that are invalid are already counted as `failed` here, because they are checked at submission time.)

### Check progress

```bash
curl http://localhost:8000/jobs/<job_id>
```

`status` is one of:

| Status                  | Meaning                                              |
|-------------------------|------------------------------------------------------|
| `pending`               | accepted, waiting to start                           |
| `processing`            | certificates are being generated (`progress_percent` rises) |
| `completed`             | every certificate was generated                      |
| `completed_with_errors` | some generated, some failed                          |
| `failed`                | no certificate could be generated                    |

### See which recipients failed, and why

```bash
curl "http://localhost:8000/jobs/<job_id>/certificates?status=failed"
```

```json
{
  "total": 1, "limit": 100, "offset": 0,
  "items": [
    { "id": 3, "index": 2, "recipient_name": "", "status": "failed",
      "error": "name must not be blank", "download_url": null }
  ]
}
```

`index` is the 0-based position in the list you submitted, so errors map straight back to your input rows. The listing supports `status=pending|generated|failed`, `limit` (1–1000) and `offset`.

## 5. Retrieve generated certificates

| Endpoint                                                | Returns                                      |
|---------------------------------------------------------|----------------------------------------------|
| `GET /jobs/{job_id}/certificates/{certificate_id}/download` | One certificate as a PDF                     |
| `GET /jobs/{job_id}/download`                           | ZIP with all successfully generated PDFs     |

```bash
curl -OJ http://localhost:8000/jobs/<job_id>/certificates/<certificate_id>/download
curl -OJ http://localhost:8000/jobs/<job_id>/download
```

Error responses: `404` unknown job/certificate, `409` certificate failed or job still running (for the ZIP), `404` for a ZIP when nothing was generated.

### Try a big batch

```bash
python examples/make_large_request.py 500 > big.json
curl -X POST http://localhost:8000/jobs -H "Content-Type: application/json" -d @big.json
# then poll GET /jobs/<id> and watch progress_percent climb
```

## 6. Important design decisions

### Processing model: background task, not synchronous

`POST /jobs` stores the job, returns `202` right away and generates the PDFs in a background task (FastAPI `BackgroundTasks`, run in a worker thread). The client polls `GET /jobs/{id}`.

* **Why not synchronous?** Hundreds of PDFs would make one HTTP request run for many seconds, risking client/proxy timeouts, and the client would learn nothing until the end. With background processing the request time is independent of batch size and progress is visible.
* **Why not Celery/RQ?** It would need a broker (Redis/RabbitMQ) and a separate worker process. For this scope that is more moving parts than the problem needs. The processing logic lives in one function (`services/processor.py::process_job`) that only takes a job id, so moving it to a real queue later means changing only how it is called, not what it does.
* **Trade-off (known limitation):** background tasks live inside the server process. If the server is killed mid-job, that job stays `processing` and is not resumed automatically. A queue with persistent tasks would fix this.

### Failure isolation

* Every certificate is generated inside its own `try/except` and committed immediately. One failure is recorded on that row (`status=failed`, `error=...`) and the loop continues.
* Validation happens in two layers:
  1. **Request level** (no recipients, missing `event_name`, bad date, too many recipients) → whole request rejected with `422`; nothing is stored.
  2. **Recipient level** (blank name, bad email, ...) → only that recipient fails; the job still runs for the rest. Recipient fields are deliberately loosely typed in the schema so that e.g. a number instead of a name fails *one row*, not the whole request.
* The job's final status is **derived from the certificate rows**, not from in-memory counters, so it cannot drift.
* If something unexpected breaks the whole processor (e.g. the database drops), remaining certificates are marked failed with a "Job aborted" error and the job is finalised, so it never stays `processing` forever.
* PDFs are written to a temporary file and renamed on success, so a crash never leaves a corrupt certificate.

### Other choices

* **Template:** one fixed landscape-A4 design drawn directly with ReportLab's canvas (no template editor). Using the canvas instead of markup-based flowables means recipient text is never interpreted as markup. Long names/titles shrink to fit on one line.
* **Data model:** `jobs` (one per request) and `certificates` (one per recipient, with `status`, `error`, `file_path`). Job IDs are random UUIDs, so they cannot be guessed or enumerated.
* **Files on disk, metadata in the DB:** PDFs are stored under `CERT_DIR/<job_id>/<certificate_id>.pdf`; the DB stores a *relative* path, so the folder can be moved or put on shared storage.
* **Progress via SQLite WAL mode:** the processor writes while clients poll; WAL lets both happen without "database is locked" errors. With PostgreSQL this is not needed.
* **Dependency injection for settings and DB sessions** makes the tests simple: they swap in a temporary database and folder without patching globals.
* **Tables are created with `create_all` on startup.** For a real deployment with schema changes, add Alembic migrations.

### Known limitations

* Names must use Latin characters (accents like `Zoë Müller` work). The built-in PDF font cannot draw e.g. Devanagari, and rather than silently producing blank names such recipients are rejected with a clear error. To support other scripts, register a Unicode TTF font in `services/generator.py` and remove `_is_renderable` in `services/validation.py`.
* No authentication or rate limiting (out of scope for this exercise).
* The ZIP is built in memory; fine for the default limit of 1000 certificates, but a streaming response would be better for much larger batches.
* A job interrupted by a server restart is not resumed (see above).

## 7. Project structure

```
app/
  main.py              FastAPI app, startup (creates tables)
  config.py            Settings from environment variables
  database.py          Engine and session factory
  models.py            Job and Certificate tables
  schemas.py           Request/response models
  dependencies.py      DB session / session-factory dependencies
  routers/jobs.py      All HTTP endpoints
  services/
    validation.py      Per-recipient validation
    generator.py       The certificate template (PDF)
    processor.py       Background job processing + failure isolation
tests/                 pytest suite (71 tests)
examples/              Sample request + generator for large requests
```
