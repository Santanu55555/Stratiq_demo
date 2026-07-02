# Stratiq — Project Showcase

**Live site:** [stratiq.webyor.com](https://stratiq.webyor.com)

**Product name:** Stratiq 

> A multi-tenant, AI-powered marketing analytics platform — built end-to-end (frontend, backend, database, background workers, deployment). This README is a **portfolio/interview reference**: it documents the architecture, engineering decisions, and problems solved, without exposing the actual source code, which is kept private.

---

##  What This Project Is

Stratiq lets marketing teams drop in a messy CSV/Excel export (ad spend, leads, revenue, campaign data) and turns it into normalized KPIs, dashboards, and AI-generated insights — including a conversational "chat with your data" analyst. It's built as a real multi-tenant SaaS product: workspaces, roles, billing, and premium tiers, not a single-user demo.

- **Type:** Multi-tenant B2B SaaS analytics platform
- **Role:** Full-stack — architecture, frontend, backend, database schema, background job infrastructure, deployment
- **Status:** Deployed (split frontend/backend production environment)

---

##  Tech Stack

| Layer | Technology |
|---|---|
| Frontend Framework | Next.js (App Router), React 19 |
| Styling | Tailwind CSS — glassmorphism-heavy, premium SaaS aesthetic |
| Animation | Framer Motion |
| Data Viz | Recharts |
| Data Grid | Custom high-performance table component |
| Icons / Notifications | Lucide React, Sonner |
| Markdown Rendering | react-markdown (for AI chat responses) |
| Auth | NextAuth.js — Google/GitHub OAuth + custom passwordless OTP |
| Email | Resend (production) with a Nodemailer/Ethereal fallback for local dev |
| Backend API | Express.js (Node.js) |
| Background Jobs | BullMQ + Redis |
| ORM / Database | Prisma ORM + PostgreSQL (Supabase, pooled via PgBouncer) |
| File Parsing | Multer + xlsx (CSV/Excel ingestion) |
| AI | Google Gemini 2.5 Flash (`@google/genai`) |
| Billing | Razorpay (subscriptions + webhooks) |
| Hosting | Vercel (frontend) + Render (backend, for long-running workers) |
| CI/CD | GitHub Actions, Docker, Docker Compose |

---

##  Architecture Highlights

### Multi-Tenancy & RBAC
A two-level hierarchy — **Workspace** (the tenant: billing, roles, audit logs) containing **Projects** (grouped datasets) — with three roles (`OWNER`, `ANALYST`, `VIEWER`) enforced on every data-access route. New signups are auto-provisioned a default workspace and project so they can start uploading data with zero setup screens.

### Passwordless Auth (OTP + OAuth)
NextAuth-based auth supporting Google/GitHub OAuth and a custom 6-digit email OTP flow. OTPs are generated server-side, stored with a 10-minute expiry, and delivered via Resend in production (with an auto-generated Ethereal inbox fallback for local development, so nothing needs configuring to test auth locally). Workspace invitations use the same OTP mechanism, embedding a one-click magic link that auto-fills the code.

### Raw-First Ingestion & Hybrid Column Mapping
Uploaded CSV/Excel files are persisted as raw JSON immediately — before any mapping or validation — so data is never lost if a later step fails. Column mapping (e.g., matching a user's `Revneu` header to the canonical `revenue` field) runs through a **client-side hybrid matching pipeline**: normalization → synonym dictionary lookup → Levenshtein fuzzy matching (auto-mapped above a 70% similarity threshold). This gives "AI-like" mapping accuracy with zero latency, zero API cost, and fully deterministic behavior — no LLM calls involved.

### Async AI Pipeline (BullMQ + Redis)
AI insight generation (via Gemini 2.5 Flash) takes 5–15 seconds — too slow to hold open an HTTP request. Requests are hashed (SHA-256 of the normalized metrics payload) for caching, then enqueued to a Redis-backed BullMQ queue; the API responds in under 200ms with a job ID while a background worker processes the job with retry/exponential-backoff handling. Identical inputs hit the cache and skip the LLM call entirely.

### High-Performance Data Grid
A custom data grid built to handle large datasets smoothly: static (non-animated) row rendering to avoid layout-observer overhead at scale, memoized in-memory search, adaptive ellipsis pagination, and a hybrid client-side/server-side pagination strategy that switches over automatically above 1,000 rows.

### Security
Layered API hardening: `helmet` for HTTP headers, payload size caps, `express-rate-limit` (with stricter limits on upload/AI/OTP routes to prevent brute-forcing and quota abuse), and RBAC checks on every data-access endpoint to enforce cross-tenant data isolation. Async job ownership is also verified, so users can't poll or read another user's AI job results.

### Serverless-Safe Database Connections
Resolved Postgres connection exhaustion under serverless scaling by routing through PgBouncer with Prisma's `@prisma/adapter-pg` driver (bypassing Prisma's default Rust engine, which doesn't play well with transaction-mode poolers).

---

##  Product Features

- Passwordless sign-in (OTP or OAuth) with workspace invitations
- Drag-and-drop CSV/Excel ingestion with automatic column mapping
- Dataset versioning (track and roll back mapping/data revisions)
- KPI dashboards, trend charts, and a data-quality confidence score
- Background AI insight generation with caching
- Conversational AI analyst — ask questions about your data in natural language, with confidence scores and source tracking on every answer
- Free / Premium tiers with Razorpay billing integration
- Full workspace/project lifecycle management (rename, delete, leave, cascading cleanup)

---

##  Deployment

A split production setup: Next.js frontend on **Vercel**, Express API + BullMQ workers on **Render**, and PostgreSQL on Supabase (chosen specifically because Vercel's serverless functions can't run the persistent background workers this architecture needs). Both are containerized with multi-stage Dockerfiles, orchestrated locally via Docker Compose (client, server, PostgreSQL, Redis), and validated through a GitHub Actions CI pipeline that lints, generates the Prisma client, and runs a full production build on every push to `main`.

Notable cross-platform issues solved along the way: dynamic CORS handling for Vercel's per-deployment preview URLs, synchronizing `NEXTAUTH_SECRET` byte-for-byte across both platforms so Express could validate NextAuth's JWTs, and fixing a silent production bug where the frontend was calling the wrong API host.

---

##  A Note on Source Code

This document exists so the project can be discussed and demonstrated — architecture, trade-offs, and specific engineering decisions — without publishing the underlying codebase. Happy to go deep on any part of this (the queueing design, the RBAC model, the fuzzy-matching pipeline, the deployment split) in conversation.
