Seek — Caregiver/Agency Marketplace (Code Private)

Note: The code for this project is intentionally private. This README documents the product, architecture, decisions, and how to run the system locally/in prod so stakeholders can evaluate the project and contributors can collaborate without exposing source code.

Overview

Seek is a two-sided platform that helps home care agencies find and manage qualified caregivers. Agencies post jobs with specific requirements (skills, languages, schedule, POC/authorization hours), and caregivers build verified profiles and apply. A matching engine ranks caregivers for each job. Beyond recruiting, Seek includes EVV-friendly shift workflows, document/compliance tracking, and audit-ready logs designed for CCU/MCO realities.

Core users & goals

Caregivers (CG): Create profile, verify credentials, specify availability, apply to jobs, manage shifts/documents.

Agencies: Create jobs with requirements & POC hours, shortlist matches, message, hire, schedule shifts, track EVV events, maintain compliance.

Admins: Operate the platform, manage orgs/roles, handle audits/backups, tune matching weights.

Feature Snapshot
Area	MVP (Phase 1)	Next (Phase 2+)
Auth	Email+password, role-based access (CG/Agency/Admin)	2FA, SSO/OAuth for enterprises
Profiles	CG profile (skills, certifications, languages, distance radius, availability)	References, performance history, badges
Jobs	Post jobs with skills, schedule, pay range, location, POC hours	Multi-site jobs, templates, draft/publish workflow
Matching	Weighted ranking by skills/schedule/location/language	Learning-to-rank + A/B test of weights
Applications	Apply, status updates, messaging	Interview scheduling, offer letters
EVV/Shifts	Manual check-in/out + geo/time capture, deviation notes	Mobile app EVV, GPS fencing, offline tolerance
Compliance	Document upload/expiry tracking, audit logs	Automated reminders, e-sign, policy acknowledgments
Notifications	Email in MVP	SMS/push, digest preferences
Admin	Org/user mgmt, feature flags, metrics	Multi-tenant billing, data retention policies
I18n/A11y	English + simple i18n hooks	Full Spanish/Gujarati content, WCAG 2.1 AA polish
High-Level Architecture

API (Node.js, TypeScript, Express/Fastify): REST endpoints, OpenAPI spec, RBAC middleware, validation, rate limits.

DB (PostgreSQL): Relational core with strong constraints and migrations (e.g., Prisma or Knex).

Cache/Queues (Redis): Caching hot reads (match results, lookups), background jobs (emails, document OCR, re-scoring).

Object Storage (S3-compatible): Documents (certifications, IDs), signed URL upload/download, virus scan pipeline.

Observability: Structured logs, health checks, request metrics, error tracing.

CI/CD: Lint/test/build, migration gate, canary deploy.

Clients: Web app (agency dashboard, caregiver portal). Mobile app optional later for EVV convenience.

Security by default

JWT access + short-lived refresh tokens; HTTPS everywhere; secrets via env; least-privilege RBAC; audit logs on sensitive actions; immutable event store for EVV.

Data Model (Conceptual)

Key entities (selected)

user (id, email, role, status, last_login_at)

agency (id, name, address, contact)

caregiver_profile (user_id FK, skills[], certifications[], languages[], radius_km, has_car, hourly_min)

availability_block (profile_id FK, day_of_week, start/end, repeats)

job (id, agency_id FK, title, location, schedule blocks, required_skills[], languages[], pay_range, poc_hours_per_week)

job_requirement (job_id FK, requirement_type, value, weight)

application (id, job_id, caregiver_user_id, status, notes)

match_result (job_id, caregiver_user_id, score, explanation_json, created_at)

document (owner_user_id/agency_id, type, storage_key, expires_at, verified_status)

shift (id, job_id, caregiver_user_id, scheduled_start/end, approved_minutes)

evv_event (shift_id, user_id, action CHECK_IN|CHECK_OUT, timestamp, geo)

deviation_note (shift_id, text, created_by, created_at)

audit_log (actor_id, action, entity, entity_id, before/after, ts)

Matching Engine (MVP Logic & Rationale)

Goal: Rank caregivers for a job with a transparent score and explanation.

MVP scoring (0–100)

Skills & Certifications (40%): exact matches → +points, must-have gaps → disqualify or heavy penalty.

Schedule fit (25%): overlap between caregiver availability blocks and job schedule.

Distance (15%): haversine distance vs caregiver radius_km; hard reject if over radius when must-travel is required.

Languages (10%): each required language adds weight.

Driver/Car (5%): adds score if job requires driving.

Pay alignment (5%): job pay range meets caregiver minimum.

Explainability

Store per-factor contributions in explanation_json. UI shows “+12 skills match, +8 schedule overlap, −6 distance”.

Future

Weighted configs per agency, feedback-driven re-ranking, small LTR model once we have outcomes (accept/hire/retain).

EVV, Compliance & Audits

EVV events: Check-in/out capture time and optional lat/long; if GPS unavailable (desktop), mark source and request a short deviation note. All EVV actions are immutable and timestamped.

POC hours: Weekly caps enforced; warning banners for approaching overages; require deviation notes if over/under hours.

Documents: Upload via signed URLs with client-side size/type checks; server-side virus scan; expiry reminders.

Audit logs: Every change to jobs, shifts, EVV, and documents is logged with actor, before/after snapshots.

Privacy: Seek is designed to avoid storing PHI beyond what’s strictly necessary for scheduling and compliance. Agencies are responsible for ensuring their use complies with applicable regulations.

API Design

Contract-first with OpenAPI (YAML) in /docs/openapi.yaml.

Conventions: REST, nouns plural, snake_case body fields, pagination via ?page/?page_size, filtering by query params, idempotent PUT/PATCH, 4xx/5xx discipline.

Versioning: /v1/... with backwards-compatible changes where possible.

RBAC: Role & ownership checks in a single authorization layer to centralize policy.

Representative endpoints (non-exhaustive)

POST /v1/auth/register | POST /v1/auth/login | POST /v1/auth/refresh

GET/PUT /v1/caregivers/me (profile) | POST /v1/caregivers/me/documents

POST /v1/agencies/{id}/jobs | GET /v1/jobs?agency_id=...

POST /v1/jobs/{id}/applications | GET /v1/jobs/{id}/matches

POST /v1/shifts | POST /v1/shifts/{id}/evv/check-in | POST /v1/shifts/{id}/evv/check-out

POST /v1/shifts/{id}/deviation-notes

Tech Stack

Backend: Node.js + TypeScript, Express/Fastify, Zod/Valibot for validation

DB: PostgreSQL + Prisma/Knex migrations

Cache/Jobs: Redis + BullMQ / Cloud tasks

Storage: S3-compatible (e.g., MinIO in dev, AWS S3 in prod)

Auth: JWT access/refresh, bcrypt/argon2, optional TOTP later

Docs: OpenAPI + Swagger UI (dev)

Testing: Vitest/Jest + Supertest; Pact for contract tests (later)

CI/CD: GitHub Actions; deploy to Render/Fly.io/AWS (infra agnostic)

Obs: pino/ Winston logs (JSON), Prometheus metrics, health endpoints

Repository Layout (private, representative)
seek/
├─ api/                  # Node TS service (private)
│  ├─ src/
│  ├─ tests/
│  └─ package.json
├─ docs/
│  ├─ openapi.yaml
│  └─ architecture.md
├─ infra/                # IaC, migrations, seed scripts
├─ ops/                  # Runbooks, dashboards, alerts
└─ README.md             # (this file)

Environment & Config

Required environment variables (sample)

DATABASE_URL (Postgres connection string)

REDIS_URL

JWT_ACCESS_TTL, JWT_REFRESH_TTL, JWT_SECRET

S3_ENDPOINT, S3_BUCKET, S3_ACCESS_KEY_ID, S3_SECRET_ACCESS_KEY

MAIL_FROM, SMTP_URL (or API key for provider)

APP_BASE_URL, ALLOWED_ORIGINS

NODE_ENV (development|production)

LOG_LEVEL (info in prod)

Store secrets in a secure vault (not in git). Local .env files should be .gitignored.

Running Locally

Since the code is private, these instructions describe the process without exposing source.

Prereqs

Node 20+, Docker + Docker Compose, psql CLI

Quick start (Docker Compose)

docker compose up -d (starts Postgres, Redis, MinIO; optional mailhog)

Run DB migrations: npm run db:migrate

Seed safe sample data: npm run db:seed

Start API: npm run dev

Visit Swagger UI: http://localhost:PORT/docs

Without Docker (power users)

Start local Postgres & Redis.

Create DB: createdb seek_dev

Set env vars; run npm install, npm run db:migrate, npm run dev.

Testing & Quality

Unit tests: business logic, scoring, validators

API tests: endpoint contracts & RBAC

E2E (optional): Wire up against ephemeral DB with seeded data

Static analysis: ESLint, TypeScript strict, Prettier

Security: npm audit, dependency pinning, Snyk (optional)

Performance smoke: k6/Gatling for hot endpoints (matching, list queries)

CI runs lint → typecheck → tests → build → migrations dry-run, then gates deployments.

Observability & Operations

Health endpoints: /health/live, /health/ready

Metrics: request latency, error rates, job queues, DB pool usage

Logs: JSON, correlation IDs, redaction of PII/secret fields

Backups: Nightly Postgres snapshots, S3 bucket versioning, restore drills quarterly

Runbooks: Incident checklist, EVV failure fallback (collect manual notes), document virus-scan retry policy

Product & Engineering Decisions (What we took and why)

Node.js + TypeScript for API

Why: Rapid iteration, rich ecosystem, TS for safety; team skill alignment.

Alternatives: Python/FastAPI (great DX), Go (performance). Trade-off: TS adds compile step; runtime perf still fine for CRUD/matching.

PostgreSQL for relational core

Why: Strong consistency for jobs/applications/EVV; robust JSONB for flexible attributes.

Alternatives: MySQL (ok), NoSQL (not ideal for join-heavy access).

OpenAPI contract-first

Why: Clear API boundary, easier front-end/mobile workstreams, auto-docs/tests.

Trade-off: Discipline needed to keep spec as the source of truth.

Weighted rules-based matching (MVP)

Why: Transparent and auditable for agencies; fast to ship.

Future: LTR/ML once we collect “hire/retain” outcomes to learn weights.

Signed URL doc uploads + virus scan

Why: Keeps API stateless for large files; security best practice.

Trade-off: Requires background jobs and scan pipeline; worth it for safety.

EVV as immutable events

Why: Audit requirements; reconstruct timelines; prevent tampering.

Trade-off: Stronger operational needs around correction flows (use deviation notes).

RBAC centralized policy layer

Why: Fewer security foot-guns; easier audits.

Trade-off: Slight complexity up-front; pays off at scale.

Minimal PHI footprint

Why: Reduce compliance scope; store only what’s necessary.

Trade-off: Some convenience trade-offs; safer posture for early stages.

Roadmap (Indicative)

Phase 1 (MVP): Auth, profiles, job posting, applications, matching v1, documents, EVV basic, email notifications, audit logs.

Phase 2: Admin console, i18n rollout (Spanish/Gujarati), 2FA, SMS notifications, compliance automations, shift scheduling UX.

Phase 3: Mobile EVV, advanced matching (feedback-aware), analytics dashboards, multi-tenant billing.

Accessibility & Internationalization

A11y: Keyboard navigation, semantic HTML, high-contrast options; ongoing WCAG improvements.

i18n: String catalogs; right now English, with content and UX designed for Spanish/Gujarati expansion.

Contributing

This is a private repository with restricted access.

Propose changes via PR with linked issue, include test coverage.

Do not commit secrets or sample real client data.

Follow commit convention (feat:, fix:, docs:) and keep PRs small.

License

Proprietary © Seek. All rights reserved.
No redistribution, copying, or sublicensing without written permission.

Glossary

EVV: Electronic Visit Verification (check-in/out for shifts)

POC: Plan of Care (authorized hours)

CCU / MCO: Community Care Unit / Managed Care Organization

CG: Caregiver

RBAC: Role-Based Access Control

Contact

Product & Operations: Agency/Admin stakeholders

Engineering: Repo maintainers

Security/Legal: Designated contact on request

Appendix: Safety & Audit Notes

Deviation notes are required when POC hours differ materially or EVV data is incomplete; the platform enforces this through UI and cannot be bypassed without a privileged override (which itself is audited).

Geo data for EVV is collected only where permitted and disclosed. If location is denied/unavailable, EVV still records time source and prompts a deviation explanation.

Data export features are scoped to the actor’s role and organization, with rate limits and watermarking to deter misuse.
