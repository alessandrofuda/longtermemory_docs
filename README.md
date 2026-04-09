# LongTermMemory

**[LongTermMemory](https://longtermemory.com)** is an AI-powered SaaS platform for studying and exam preparation. It automatically generates question-answer pairs from uploaded documents and web links using Retrieval-Augmented Generation (RAG), then schedules review sessions using spaced repetition to move knowledge into long-term memory.

## Architecture

```
┌─────────────────────────────────────────────────┐
│           Frontend (React SPA)                  │
│           http://localhost:5173                  │
└────────────────────┬────────────────────────────┘
                     │ REST API (Bearer token)
                     ▼
┌─────────────────────────────────────────────────┐
│        Laravel 12 API (Nginx + PHP-FPM)         │
│           http://localhost:8080                  │
│                                                 │
│  ┌──────────────┐    ┌─────────────────────┐   │
│  │  MySQL 8.0   │    │    Redis             │   │
│  │  (port 3318) │    │  (cache + queues)    │   │
│  └──────────────┘    └─────────────────────┘   │
└────────────────────┬────────────────────────────┘
                     │ HTTP + X-API-Key
                     ▼
┌─────────────────────────────────────────────────┐
│      Python RAG Service (FastAPI + Celery)      │
│           http://localhost:8000                  │
│                                                 │
│  ┌──────────────┐    ┌─────────────────────┐   │
│  │   Qdrant     │    │  OpenAI API          │   │
│  │ (port 6333)  │    │ (embeddings + LLM)  │   │
│  └──────────────┘    └─────────────────────┘   │
└─────────────────────────────────────────────────┘

        ┌─────────────────────────┐
        │  MinIO (S3-compatible)  │
        │  Object Storage         │
        │  (port 9000/9001)       │
        └─────────────────────────┘
        Used by both Laravel and RAG Service
```

### How it works

1. **User uploads** a PDF/DOCX or submits a web link via the React frontend
2. **Laravel** stores the file in MinIO and triggers the RAG service
3. **RAG service** (Celery worker) chunks and embeds the document, stores vectors in Qdrant, then generates Q&A pairs using OpenAI
4. **Laravel** receives the generated Q&A pairs via a callback and saves them to MySQL
5. **User studies** via an interactive session with spaced repetition scheduling

---

## Services

| Service | Technology | Dev Port |
|---------|-----------|----------|
| Frontend | React 19 + Vite 7 | 5173 |
| API | Laravel 12 (Nginx) | 8080 |
| RAG Service | FastAPI + Uvicorn | 8000 |
| Celery monitoring | Flower | 5555 |
| Database | MySQL 8.0 | 3318 |
| Cache / Queue broker | Redis 7 | 6379 |
| Vector database | Qdrant | 6333 |
| Object storage | MinIO | 9000 (API), 9001 (Console) |

---

## Prerequisites

- **Docker** and **Docker Compose** (for the backend stack)
- **Node.js 18+** and **npm** (for frontend and blog)
- **PHP 8.2+** and **Composer** (only if running Laravel without Docker)

---

## Quick Start

### 1. Clone and set up environment files

```bash
git clone <repo-url>
cd longtermemory

# Copy environment templates
cp backend/.env.example backend/.env
cp backend/main-app/.env.example backend/main-app/.env
cp frontend/.env.example frontend/.env
```

Open `backend/main-app/.env` and fill in the required variables listed in the [Environment Variables](#environment-variables) section below.

### 2. Start the backend stack

```bash
cd backend
docker compose up -d
```

This starts Laravel, MySQL, Redis, MinIO, Qdrant, the RAG service, Celery worker, and Flower.

### 3. Run database migrations and seed plans

```bash
docker exec -it longtermemory_app php artisan key:generate
docker exec -it longtermemory_app php artisan migrate
docker exec -it longtermemory_app php artisan db:seed --class=CommercialPlanSeeder
```

### 4. Create the MinIO bucket

Open the MinIO console at `http://localhost:9001` (dev credentials: `minioadmin` / `miniopassword`) and create a bucket named `documents`.

### 5. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

The app is now running at **http://localhost:5173**.

---

## Environment Variables

Copy the `.env.example` files — they document every variable. The following are required and have no defaults:

| Variable | Where | Purpose |
|----------|-------|---------|
| `OPENAI_API_KEY` | `backend/main-app/.env` | OpenAI API access for RAG and embeddings |
| `STRIPE_PUBLIC_KEY` | `backend/main-app/.env` | Stripe publishable key |
| `STRIPE_SECRET_KEY` | `backend/main-app/.env` | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | `backend/main-app/.env` | Stripe webhook signature verification |
| `STRIPE_PRICE_ID_BASIC` | `backend/main-app/.env` | Stripe price ID for Basic plan |
| `STRIPE_PRICE_ID_PRO` | `backend/main-app/.env` | Stripe price ID for Pro plan |
| `RESEND_KEY` | `backend/main-app/.env` | Resend API key for email delivery |
| `VITE_STRIPE_PUBLIC_KEY` | `frontend/.env` | Stripe publishable key (same as above) |

---

## Documentation

| Guide | Contents |
|-------|----------|
| [Frontend](docs/frontend.md) | React SPA — setup, routing, auth, API integration, state management |
| [Backend](docs/backend.md) | Laravel API + Python RAG service — endpoints, data models, Celery, testing |
| [Blog](docs/blog.md) | Astro content site — content management, routing, SEO, deployment |
