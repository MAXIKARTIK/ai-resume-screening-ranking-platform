# AI Resume Screening Platform

Backend-first platform that ingests resumes and job descriptions, extracts and
normalizes structured candidate data, and ranks applicants against a JD with a
hybrid, explainable pipeline — plus AI analysis, comparison, duplicate
detection, interview-question generation, resume review/ATS optimization,
explainable ranking rationales, and exportable reports.

Every AI capability is coordinated through a single **AI Orchestrator**; no
endpoint or worker step calls a model directly. The system follows Clean
Architecture / DDD (`api → application → domain → infrastructure`). Structured
data lives in PostgreSQL; vectors live in ChromaDB; the two are never mixed.

## Architecture

```
api/             FastAPI routers, DI wiring, auth guards, error mapping
application/     use cases: AI Orchestrator, services, ranking, normalization, pipeline
domain/          framework-free entities, enums, ports (seams), errors
infrastructure/  adapters: mock/NIM LLM+embeddings, in-memory/Chroma vectors,
                 in-memory/SQLAlchemy repos, storage, security, Celery
```

Dependencies point inward. Infrastructure implements domain **ports**
(`LLMPort`, `EmbeddingPort`, `VectorPort`, `RepoPort`, `StoragePort`), so
adapters are swappable (mock ↔ NIM, in-memory ↔ Postgres/Chroma) with no change
to inner layers.

## High-Level Design (HLD)

```mermaid
graph TD
    subgraph API["api — FastAPI + Swagger"]
        R["Routers: auth · jobs · resumes · features(rank/analyze/compare/interview/review/chat/report)"]
        DEP["deps: DI container · current_user (JWT) · RateLimiter · error mapping"]
    end

    subgraph APP["application — use cases"]
        ORCH["AI Orchestrator (sole AI gateway)"]
        PM["PromptManager (versioned)"]
        MR["ModelRouter (tiered)"]
        OV["OutputValidator (Pydantic)"]
        SVc["Services: Auth · RBAC · Job · Resume · Parsing · Embedding · Retrieval · Ranking · Analysis · Comparison · Interview · Reviewer · Report · Audit · Evaluation"]
        PIPE["IngestionPipeline (UPLOADED→PARSED→NORMALIZED→EMBEDDED)"]
    end

    subgraph DOM["domain — entities + ports"]
        ENT["Entities: User · JobDescription · JobCandidateAssociation · Resume · NormalizedProfile · Ranking · Report · AuditLog · TokenCostMetric"]
        PORTS["Ports: LLMPort · EmbeddingPort · VectorPort · RepoPort · StoragePort"]
        ENUM["Enums: Role · PipelineStatus"]
    end

    subgraph INFRA["infrastructure — adapters"]
        MOCK["MockLLM + MockEmbedding (Offline_Mode)"]
        NIM["NIM adapters (online)"]
        VEC["InMemory / ChromaDB VectorStore"]
        REPO["InMemory / SQLAlchemy repos"]
        STOR["InMemory / LocalDisk / object storage"]
        SEC["security: argon2 + JWT"]
        CEL["Celery + Redis worker"]
    end

    R --> DEP --> SVc
    SVc --> ORCH
    ORCH --> PM & MR & OV
    ORCH --> PORTS
    SVc --> PORTS
    PIPE --> SVc
    PORTS -. implemented by .-> MOCK & NIM & VEC & REPO & STOR
    DEP --> SEC
    CEL --> PIPE
    ENT --- ENUM
```

**Invariant:** arrows from `api`/`application` reach infrastructure **only through ports**. `LLMPort`/`EmbeddingPort` have paired mock + NIM implementations selected by config; Offline_Mode binds the mocks so the whole system boots key-less.

## AI request flow (every AI call goes through the orchestrator)

```mermaid
sequenceDiagram
    participant UC as Use case / Worker step
    participant ORCH as AI Orchestrator
    participant PM as PromptManager
    participant MR as ModelRouter
    participant LLM as LLMPort (Mock|NIM)
    participant OV as OutputValidator
    participant PG as Postgres (metrics + audit)

    UC->>ORCH: run(task, payload, schema)
    ORCH->>PM: render(versioned template)
    ORCH->>MR: pick_model(task)  %% small vs large tier
    ORCH->>LLM: complete(prompt, model)
    LLM-->>ORCH: raw text (NEVER persisted as-is)
    ORCH->>OV: parse_and_validate(raw, schema)
    alt valid
        OV-->>ORCH: typed schema instance
    else invalid
        ORCH->>LLM: bounded repair retry
        Note over ORCH,OV: after max attempts → SchemaValidationError (persist nothing)
    end
    ORCH->>PG: record token/cost/latency + AI_REQUEST audit
    ORCH-->>UC: validated result only
```

## Automatic ingestion chain (stops at EMBEDDED)

```mermaid
stateDiagram-v2
    [*] --> UPLOADED
    UPLOADED --> PARSED: parse (pdf/docx text)
    PARSED --> NORMALIZED: extract → validate → normalize (atomic; profile persisted)
    NORMALIZED --> EMBEDDED: chunk → embed → store vectors (verify)
    EMBEDDED --> [*]: automatic chain COMPLETE

    UPLOADED --> FAILED: corrupt/empty/unsupported
    PARSED --> FAILED
    NORMALIZED --> FAILED: vector upsert retries exhausted

    note right of EMBEDDED
        ANALYZED / RANKED / REPORTED are reached ONLY by
        on-demand, recruiter-triggered operations — never
        by the automatic chain.
    end note
```

## Database / ER model (PostgreSQL — structured source of truth)

Vectors are **never** stored here; they live only in ChromaDB with metadata keys
restricted to a subset of `{candidate_id, job_id, chunk_type}`.

```mermaid
erDiagram
    USERS ||--o{ JOBS : "owns (recruiter)"
    USERS ||--o{ RESUMES : "owns (candidate)"
    USERS ||--o{ REFRESH_TOKENS : has
    USERS ||--o{ PASSWORD_RESET_TOKENS : has
    JOBS ||--o{ JOB_CANDIDATE_ASSOCIATIONS : "scopes"
    USERS ||--o{ JOB_CANDIDATE_ASSOCIATIONS : "linked as candidate"
    RESUMES ||--|| NORMALIZED_PROFILES : "produces (1:1)"
    JOBS ||--o{ RANKINGS : "ranked for"
    USERS ||--o{ RANKINGS : "ranked candidate"
    RANKINGS ||--o{ REPORTS : "exported as"

    USERS {
        uuid id PK
        string email UK
        string password_hash "argon2 only"
        string role "recruiter|candidate|admin"
        datetime created_at
    }
    JOBS {
        uuid id PK
        uuid recruiter_id FK
        string title
        text raw_text "raw JD preserved"
        json normalized "derived NormalizedJD"
        string jd_vector_id "Chroma JD vector id"
        datetime created_at
    }
    JOB_CANDIDATE_ASSOCIATIONS {
        uuid id PK
        uuid job_id FK
        uuid candidate_id FK
        uuid created_by FK "job owner"
        datetime created_at
    }
    RESUMES {
        uuid id PK
        uuid candidate_id FK
        string filename
        string content_hash "dedup"
        int version ">0"
        string storage_key "binary outside DB"
        string status "PipelineStatus"
        datetime created_at
    }
    NORMALIZED_PROFILES {
        uuid id PK
        uuid resume_id FK
        uuid candidate_id FK
        string full_name
        json skills "deduped+lowercased"
        int total_experience_months ">=0"
        json education
        json projects
        datetime normalized_at
    }
    RANKINGS {
        uuid id PK
        uuid job_id FK
        uuid candidate_id FK
        float total_score "[0,1] scalar, not a vector"
        json breakdown "per-signal sub-scores"
        json weights "recorded for reproducibility"
        text rationale "validated LLM text"
        float confidence "[0,1]"
        datetime created_at
    }
    REPORTS {
        uuid id PK
        uuid ranking_id FK
        string pdf_key
        json json_payload "round-trippable export"
        datetime created_at
    }
    AUDIT_LOGS {
        uuid id PK
        uuid actor_id
        string action
        string subject_type
        uuid subject_id
        json meta
        datetime created_at
    }
    TOKEN_COST_METRICS {
        uuid id PK
        string task
        string model
        int prompt_tokens
        int completion_tokens
        float estimated_cost
        float latency_ms
        datetime created_at
    }
    REFRESH_TOKENS {
        uuid id PK
        uuid user_id FK
        string token UK
        bool revoked
        datetime created_at
    }
    PASSWORD_RESET_TOKENS {
        uuid id PK
        uuid user_id FK
        string token UK
        datetime expires_at
        bool used
        datetime created_at
    }
```

## Vector store (ChromaDB — vectors only)

```mermaid
graph LR
    subgraph Chroma["ChromaDB (vectors ONLY + minimal metadata)"]
        RC["resume-chunk vectors<br/>metadata: candidate_id, chunk_type"]
        JD["JD vectors<br/>metadata: job_id, chunk_type (NO candidate_id)"]
    end
    PG[("PostgreSQL")] -. "candidate_id / job_id references" .-> Chroma
```


### Storage boundary
- **PostgreSQL**: profiles, scores, rationales, audit, associations, metrics.
- **ChromaDB**: vectors only + minimal metadata — a subset of
  `{candidate_id, job_id, chunk_type}` (JD vectors carry `{job_id, chunk_type}`).

### Pipeline
The automatic upload chain (Celery over Redis) advances only through
`EMBEDDED`: Upload → Parse → Validate → Normalize → Chunk → Embed → Store.
**Analyze / Rank / Report are on-demand**, recruiter-triggered against a chosen
job, and operate only over that job's associated candidates.

## Web UI (React + TypeScript + Vite)

A production-quality SaaS frontend lives in `frontend/` (React 18 + TypeScript +
Vite, React Router). It's an ATS-style app with a sidebar, dashboard, jobs &
candidate management, resume upload with a live pipeline stepper
(Uploaded → Parsed → Normalized → Embedded → Analyzed → Ranked → Report Ready),
ranking table, analysis, comparison, interview questions, resume/ATS review,
report viewer, settings, and profile — with **dark mode**,
responsive layout, loading skeletons, empty/error states, toasts, confirm
dialogs, and **searchable dropdowns instead of UUID inputs**.

The app is **built to static assets and served by FastAPI at `/`** (single
service — no separate frontend host needed). In development it runs on the Vite
dev server and proxies API calls to the backend.

```bash
# Dev (hot reload): backend on :8000, frontend on :5173
uvicorn api.app:app --reload
cd frontend && npm install && npm run dev

# Production build (served by FastAPI at /)
cd frontend && npm run build      # emits frontend/dist
```

Docs: see `docs/ARCHITECTURE.md`, `docs/API.md`, `docs/DEPLOYMENT.md`,
`docs/DEVELOPER.md`, `docs/TROUBLESHOOTING.md`.

## Offline_Mode (no NVIDIA key)

Set `AI_MODE=offline` (the default) — or simply omit `NVIDIA_API_KEY` — to bind
the paired deterministic mock LLM + embedding adapters. The whole system boots
and the full test suite passes with **no key and no external services**.

```bash
pip install -e ".[dev]"        # or: pip install pydantic fastapi hypothesis pytest ...
pytest                         # full suite, Offline_Mode, coverage gate >= 80%
uvicorn api.app:app --reload   # Swagger at http://localhost:8000/docs
```

## Local Docker stack

```bash
cp .env.example .env
docker compose up               # api + worker + postgres + redis + chroma
```

## Online deployment (free tier, no GPU)

Same image, env-only difference. AI is the hosted **NVIDIA NIM** API.

- **Fly.io**: `fly deploy` (uses `fly.toml`; `alembic upgrade head` runs on release).
- **Render**: connect the repo (uses `render.yaml`; migrations via `preDeployCommand`).
- **Railway**: uses `railway.json` (Dockerfile build; migrations run in `startCommand`).
- **Datastores**: Neon/Supabase (Postgres), Upstash (Redis), Chroma Cloud (vectors).

### NVIDIA NIM key setup
1. Get a free developer key at `build.nvidia.com`.
2. Set it as a secret (never commit it):
   - Fly: `fly secrets set NVIDIA_API_KEY=... JWT_SECRET=...`
   - Render: set `NVIDIA_API_KEY` (and other secrets) in the dashboard.
3. Set `AI_MODE=online`. Verify model ids in `MODEL_SMALL` / `MODEL_LARGE` /
   `EMBEDDING_MODEL` against `build.nvidia.com`.

The only difference between local and online is environment variables (Req 31.1/31.2).

## Admin provisioning

Admins are never created via public registration (Req 1.8):

```bash
python seed.py admin admin@example.com "a-strong-password"
```

## Quality gates (CI)

GitHub Actions runs Ruff, Black, mypy, and pytest in Offline_Mode, failing the
build on any issue or if coverage is below the threshold (see
`.github/workflows/ci.yml`).

## Requirements & design traceability

Implementation follows `.kiro/specs/ai-resume-screening-platform/`
(`requirements.md`, `design.md`, `tasks.md`). Correctness properties 1–11 plus
the report JSON round-trip are covered by `hypothesis` property tests in
`tests/`.
