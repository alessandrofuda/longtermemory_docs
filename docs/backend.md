# Backend

The backend powers [longtermemory.com](https://longtermemory.com) and consists of two services:

- **Laravel main-app** — REST API, authentication, business logic, payments
- **Python RAG service** — document processing, embeddings, AI-powered Q&A generation

Both run in Docker Compose alongside MySQL, Redis, MinIO, and Qdrant.

---

## Docker Setup

All backend services are defined in `backend/docker-compose.yml`.

```bash
cd backend

docker compose up -d              # Start all services
docker compose down               # Stop all services
docker compose logs -f app        # PHP-FPM logs
docker compose logs -f python-rag # RAG service logs
docker compose logs -f celery-worker

# Production compose file
docker compose -f docker-compose.prod.yml up -d
```

### Service ports (development)

| Service | Host Port | Notes |
|---------|-----------|-------|
| Laravel API (Nginx) | 8080 | Main API |
| RAG Service (FastAPI) | 8000 | Called by Laravel |
| Flower (Celery UI) | 5555 | Task monitoring |
| MySQL | 3318 | Database |
| Redis | 6379 | Cache + queue broker |
| Qdrant | 6333 | Vector DB |
| MinIO API | 9000 | Object storage API |
| MinIO Console | 9001 | Web UI (dev only) |

In production, only ports **8080** and **5555** (with basic auth) are exposed externally. Access all other services via SSH tunnel.

---

## Laravel Main App

### Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Laravel 12 |
| PHP | 8.2 |
| Web server | Nginx 1.25 |
| Database | MySQL 8.0 |
| Auth | Laravel Sanctum v4 |
| Payments | Laravel Cashier v16 (Stripe) |
| Email | Resend PHP SDK |
| Code formatter | Laravel Pint v1 |
| Test runner | PHPUnit v11 |

### Development Commands

```bash
# From backend/ directory (Docker recommended)
docker exec -it longtermemory_app php artisan migrate
docker exec -it longtermemory_app php artisan test
docker exec -it longtermemory_app php artisan test --filter=test_method_name
docker exec -it longtermemory_app php artisan test tests/Feature/Auth/AuthTest.php
docker exec -it longtermemory_app vendor/bin/pint          # Format all PHP code
docker exec -it longtermemory_app vendor/bin/pint --dirty  # Format changed files only

# Local (without Docker), from backend/main-app/
composer run dev    # Server + queue + logs
composer run test   # PHPUnit
```

The `main-app/` directory is mounted as `/app` inside the container.

---

### API Endpoints

#### Authentication (public)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/magic-link` | Send magic link email to user |
| POST | `/api/auth/exchange` | Exchange one-time code for JWT token |

#### Protected (require `Authorization: Bearer {token}`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/logout` | Invalidate current token |
| GET | `/api/user` | Get authenticated user and plan |
| POST | `/api/user/update-timezone` | Update user's IANA timezone |
| POST | `/api/documents` | Upload documents (PDF, DOC, DOCX) |
| POST | `/api/weblinks` | Submit web links |
| POST | `/api/check-url` | Fetch title/description from a URL |
| POST | `/api/projects/notes` | Save notes for a project |
| POST | `/api/generate-study-plan` | Start async Q&A generation |
| GET | `/api/study-plan/status/{job_id}` | Poll generation job status |
| GET | `/api/get-study-plan/pr/{projectId}` | Fetch existing study plan |
| POST | `/api/create-new-study-session` | Create a study session |
| POST | `/api/get-qa-item` | Fetch next Q&A item for practice |
| POST | `/api/qa-item-evaluation` | Save spaced repetition evaluation |
| GET | `/api/get-study-sessions/pr/{projectId}` | List all sessions for a project |
| POST | `/api/subscribe-checkout` | Generate Stripe checkout session URL |
| POST | `/api/swap-plan` | Swap subscription plan |

#### Public

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/get-commercial-plans` | List all available plans |
| POST | `/api/stripe/webhook` | Stripe event handler |
| POST | `/api/resend/webhook` | Resend inbound email handler |
| GET | `/api/notifications/unsubscribe/{user_id}` | Opt-out via signed URL |
| POST | `/api/import-from-google-drive` | Import a Google Doc as PDF |

---

### Authentication Flow

1. User submits email → `POST /api/auth/magic-link` → email with one-time link is sent via Resend
2. User clicks link (contains `?code=...`)
3. Frontend sends code → `POST /api/auth/exchange` → returns a Sanctum token
4. Token stored in `localStorage` as `auth_token`
5. All subsequent requests: `Authorization: Bearer {token}`
6. On every successful login, `notifications_enabled` is reset to `true` for the user

---

### Commercial Plans & Trial System

Three plans are seeded by `CommercialPlanSeeder`:

| Plan | ID | Notes |
|------|----|-------|
| Free | 1 | Default for new users; 14-day trial |
| Basic | 2 | Entry-level paid |
| Pro | 3 | Full-featured paid |

Each plan enforces limits: `max_projects`, `max_document_size`, `max_documents_number`, `max_weblinks_size`, `max_weblinks_number`.

**Trial flow**: When a Free plan user creates their first project, a 14-day trial starts automatically (`trial_starts_at`, `trial_ends_at`, `trial_used`). After expiry, protected endpoints return `403` with `requires_upgrade: true`. Controlled by the `TrailPeriodExpired` middleware.

---

### Stripe Integration

- Uses [Laravel Cashier v16](https://laravel.com/docs/cashier) with the `Billable` trait on the `User` model
- `POST /api/subscribe-checkout` → creates a Stripe Checkout session, returns `checkout_url`
- `POST /api/swap-plan` → swaps active subscription (handles 3D Secure)
- `POST /api/stripe/webhook` → processes `customer.subscription.created` and `customer.subscription.updated`, maps Stripe `price_id` → internal `plan_id`

Required environment variables: `STRIPE_PUBLIC_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID_BASIC`, `STRIPE_PRICE_ID_PRO`.

---

### Email System (Resend)

- **Magic link emails**: sent during authentication
- **Inbound email forwarding**: `POST /api/resend/webhook` receives emails sent to `support@longtermemory.com`, verifies the Svix signature, fetches full email content from Resend API, and re-queues it as a `SupportEmail` mailable to `RESEND_FORWARD_TO`
- **Study review reminders**: daily digest emails at 8 AM (user's local time), sent via queued `StudyReviewReminder` notifications

Required environment variables: `RESEND_KEY`, `RESEND_WEBHOOK_SECRET`, `RESEND_FORWARD_TO`.

---

### Google Drive Import

`POST /api/import-from-google-drive` — public endpoint (no Sanctum token required; identity is verified via Google `idToken`).

**Flow**:
1. Verify `idToken` via Google Client → resolve user email
2. Resolve or create user account
3. Export Google Doc as PDF via Drive API using `accessToken`
4. Enforce plan limits (file size, document count)
5. Store PDF in MinIO, persist `StoredDocument` record
6. Return a 5-minute temporary signed URL that auto-logs in the user and redirects to the dashboard

Required environment variable: `GOOGLE_CLIENT_ID`.

---

### Notification System

**Daily study reminders** (`custom:send-study-review-notifications`):
- Runs hourly; self-limits to users whose local time is 8 AM
- Sends a consolidated email listing projects with due or new study items
- Deduplicates via a `notification_logs` table
- Respects `users.notifications_enabled` flag
- Dispatched with 1-second incremental delay per user to respect Resend rate limits

**Auto-disable inactive users** (`custom:disable-inactive-user-notifications --days=30`):
- Runs weekly; finds users with no API activity in the last 30 days
- Sets `notifications_enabled = false` and sends a courtesy email
- `notifications_enabled` is re-enabled automatically on next login

**Unsubscribe**: Every reminder email contains a signed unsubscribe link (`GET /api/notifications/unsubscribe/{user_id}`).

---

### Error Handling

`UserFacingException` (`app/Exceptions/UserFacingException.php`) is a custom exception whose message is safe to return to the frontend as-is. It bypasses Laravel's production error masking:

```php
throw new UserFacingException('You have reached the maximum number of projects.');
// → HTTP 422 { "message": "..." }

throw new UserFacingException('Payment required.', 402);
// → HTTP 402 { "message": "..." }
```

Use it for business-rule violations (plan limits, resource not found). Use plain `Exception` for unexpected internal errors (they resolve to a generic 500).

---

### Database Schema

```
User
├── id, email, password_hash, plan_id (FK → CommercialPlan)
├── trial_starts_at, trial_ends_at, trial_used
├── stripe_id, stripe_status
├── notifications_enabled, timezone
└── timestamps

CommercialPlan
├── id, name, price_monthly, price_yearly
└── max_projects, max_document_size, max_documents_number,
    max_weblinks_size, max_weblinks_number

Project
└── id, user_id (FK), name, description, timestamps

StoredDocument
└── id, project_id (FK), filename, file_path (MinIO key),
    file_size, mime_type, timestamps

StoredWebLink
└── id, project_id (FK), link, title, timestamps

QAPair
└── id, project_id (FK), user_id (FK), question, answer,
    source_chunk_id, key_concepts (JSON), difficulty_level, timestamps

StudySession
└── id, user_id (FK), project_id (FK), qa_pair_id (FK),
    next_review_date, review_count, performance_score (1–5),
    interval (days), last_reviewed_at, timestamps
```

Key relationships: User → Projects (1:many) → Documents/WebLinks (1:many) → QAPairs (1:many) → StudySessions (1:many).

---

### Testing

Tests run inside Docker against a dedicated `longtermemory_test` MySQL database.

```bash
docker exec -it longtermemory_app php artisan test                          # All 51 tests
docker exec -it longtermemory_app php artisan test --testsuite=Feature
docker exec -it longtermemory_app php artisan test --filter=test_method_name
docker exec -it longtermemory_app php artisan test tests/Feature/Auth/AuthTest.php
```

**Key test patterns**:
- `actingAsUser(CommercialPlan::FREE)` — creates a user with a real Sanctum token
- `Storage::fake('s3')` — mocks MinIO
- `Http::fake([url => response])` — mocks RAG service HTTP calls
- `Notification::fake()` / `Mail::fake()` — prevents real emails
- `Carbon::setTestNow()` — freezes time for timezone-sensitive tests (reset in `tearDown()`)
- Business limit errors throw `Exception` → HTTP 500 (not 422); error logs during tests are expected

---

## Python RAG Service

### Tech Stack

| Component | Technology |
|-----------|-----------|
| API framework | FastAPI 0.109.0 |
| ASGI server | Uvicorn (2 workers) |
| Task queue | Celery 5.3.4 |
| Task monitoring | Flower 2.0.1 |
| Message broker | Redis 7 |
| RAG orchestration | LlamaIndex 0.10.0 |
| Vector database | Qdrant (client 1.7.3) |
| Embeddings | OpenAI `text-embedding-3-small` |
| LLM | OpenAI `gpt-3.5-turbo` (configurable via `LLM_MODEL`) |
| Document parsing | PyMuPDF, python-docx, openpyxl, BeautifulSoup4 |
| Object storage client | boto3 1.34.0 |
| Test framework | pytest + pytest-asyncio + pytest-timeout |

### Service Structure

```
rag-service/
├── main.py                     # FastAPI entry point
├── celery_app.py               # Celery configuration
├── config/
│   ├── settings.py             # All configuration (Qdrant, Redis, OpenAI, etc.)
│   ├── constants.py            # LLM prompts, model config, chunking params
│   └── logging_config.py
├── models/schemas.py           # Pydantic request/response models
├── services/
│   ├── celery_tasks.py         # Background tasks (Q&A generation)
│   ├── document_processor.py   # PDF/DOCX/XLSX parsing + semantic chunking
│   ├── embeddings.py           # OpenAI embeddings
│   ├── qdrant_manager.py       # Vector DB operations
│   ├── llm_provider.py         # OpenAI LLM wrapper + prompt assembly
│   └── rag_pipeline.py         # RAG orchestration
├── routers/
│   ├── documents.py
│   ├── qa.py                   # Q&A generation + job status
│   └── health.py
├── utils/
│   ├── auth.py                 # API key authentication (X-API-Key header)
│   └── job_storage.py          # Redis job tracking
└── tests/                      # 202 tests
```

### Development Commands

```bash
# Start/restart RAG services
docker compose up -d python-rag celery-worker flower
docker compose restart celery-worker        # After Python code changes
docker compose logs -f celery-worker
docker compose up -d --build python-rag     # After dependency changes

# Access container shell
docker exec -it longtermemory_python_rag bash

# Run tests (202 tests, ~7s)
docker compose exec -T python-rag python -m pytest tests/ -v
docker compose exec -T python-rag python -m pytest tests/test_embeddings.py -v
docker compose exec -T python-rag python -m pytest tests/ -v -m unit
docker compose exec -T python-rag python -m pytest tests/ -v --cov=.
```

### ⚠️ Critical: Celery Does Not Auto-Reload

Unlike the FastAPI service (which uses `--reload`), the Celery worker loads code once at startup. **You must restart it after every Python code change**:

```bash
docker compose restart celery-worker
```

This applies to changes in: `celery_tasks.py`, `rag_pipeline.py`, `llm_provider.py`, `document_processor.py`, `embeddings.py`, `qdrant_manager.py`, `schemas.py`, `constants.py`, `celery_app.py`.

---

### Q&A Generation Flow

```
Laravel POST /api/generate-qa
        │
        ▼
Active job check (Redis: project_job:{project_id})
    → 409 Conflict if already queued/processing
        │
        ▼
Celery task queued (rag_processing queue)
        │
        ▼
For each source (document or weblink):
  1. Download file from MinIO (documents) or scrape URL (weblinks)
  2. Parse text (PyMuPDF / python-docx / BeautifulSoup4)
  3. Semantic chunking (2-stage LlamaIndex, dynamic sizing)
  4. Section title enrichment (structural heading or LLM-generated)
  5. Generate embeddings (OpenAI text-embedding-3-small)
  6. Store vectors + metadata in Qdrant (project_{project_id} collection)
        │
        ▼
For each chunk:
  1. Retrieve 3 semantically similar chunks from Qdrant (score ≥ 0.7)
  2. Build prompt: user notes + document context + chunk + related context
  3. Call OpenAI LLM → JSON with question, answer, key_concepts, difficulty_level
        │
        ▼
Notify Laravel via POST /api/job-finished (push model — no polling from Laravel)
```

**Concurrent job prevention**: A Redis key `project_job:{project_id}` tracks the active job per project. The RAG service returns HTTP 409 if a job is already running; cleared on completion, failure, or cancellation.

**Job states**: `queued` → `processing` → `completed` | `failed` | `cancelled` (tracked in Redis, 24-hour expiry)

---

### Semantic Chunking

Two-stage LlamaIndex approach with dynamic sizing based on content length:

| Content | Token threshold | Chunk size | Use case |
|---------|----------------|------------|---------|
| Short (≤ ~15 pages) | < 10,000 tokens | 1024 | Articles, chapters |
| Long (> ~15 pages) | ≥ 10,000 tokens | 2048 | Full books, long reports |

**Stage 1**: `SentenceSplitter` on paragraph breaks, 200-token overlap  
**Stage 2**: `SemanticSplitterNodeParser` with embedding similarity to avoid mid-concept splits  
**Fallback**: Stage 1 only if no OpenAI API key is configured

---

### RAG Service API Endpoints

All protected endpoints require `X-API-Key` header (set via `RAG_SERVICE_API_KEY` in Laravel's config).

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| POST | `/api/generate-qa` | Start Q&A generation job |
| GET | `/api/generate-qa/{job_id}` | Get job status and results |
| POST | `/api/generate-qa/{job_id}/cancel` | Cancel a running job |
| POST | `/api/query` | Semantic search over project chunks |

**POST `/api/generate-qa` request body**:

```json
{
  "project_id": 123,
  "user_id": 456,
  "batch_size": 25,
  "documents": [
    {
      "document_id": 789,
      "document_path": "users/456/abc.pdf",
      "original_filename": "notes.pdf",
      "document_myme_type": "application/pdf",
      "size": 245678
    }
  ],
  "weblinks": [
    {
      "weblink_id": 101,
      "weblink_url": "https://example.com/article",
      "weblink_title": "Article Title",
      "weblink_description": "Optional description"
    }
  ],
  "user_notes": "Focus on key concepts for the exam",
  "enable_rag_context": true,
  "retrieval_top_k": 3,
  "retrieval_min_score": 0.7
}
```

---

### Environment Variables (RAG Service)

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | — | Required. OpenAI API key |
| `LLM_MODEL` | `gpt-3.5-turbo` | OpenAI model for Q&A generation |
| `QDRANT_HOST` | `qdrant` | Vector DB hostname |
| `QDRANT_PORT` | `6333` | Vector DB port |
| `REDIS_HOST` | `redis` | Redis hostname |
| `REDIS_PORT` | `6379` | Redis port |
| `REDIS_PASSWORD` | — | Required in production |
| `REDIS_DB` | `0` | Redis database index |
| `ENVIRONMENT` | — | `development` or `production` |
| `LOG_LEVEL` | `INFO` | Python logging level |

---

### Testing Infrastructure

- **202 tests**, ~7 second runtime
- `pytest.ini`: 30-second global timeout prevents infinite loops
- `conftest.py`: proper async event loop management
- Markers: `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.slow`

```bash
docker compose exec -T python-rag python -m pytest tests/ -v -m unit
```

---

### Object Storage (MinIO)

Both Laravel and the RAG service use MinIO as S3-compatible object storage.

- **Bucket**: `documents`
- **File key format**: `users/{user_id}/{uuid}.{extension}`
- **Dev console**: `http://localhost:9001` (default dev credentials: `minioadmin` / `miniopassword` — change in production)
- **Laravel env**: `AWS_ENDPOINT=http://minio:9000`, `AWS_USE_PATH_STYLE_ENDPOINT=true`
- **RAG service env**: `AWS_ENDPOINT_URL=http://minio:9000`

> The MinIO dev credentials (`minioadmin`/`miniopassword`) are Docker Compose defaults for local development only. Set strong credentials via environment variables before deploying to production.

---

### Flower — Celery Monitoring

Access the Flower UI at `http://localhost:5555` to monitor:
- Active and completed Celery tasks
- Worker health and concurrency
- Task history and execution times
- Redis broker connection status

In production, Flower requires basic auth (`FLOWER_USER` / `FLOWER_PASSWORD` env vars).
