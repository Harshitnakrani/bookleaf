# BookLeaf Author Support & Communication Portal

> **Full-Stack Technical Assignment 1** &mdash; BookLeaf Publishing  
> Built with NestJS (Backend), Next.js 16 (Frontend), Neon PostgreSQL, Prisma, BullMQ, Valkey TLS (Redis), and Groq LLM.

### 🌐 Live Deployment URLs
* **Live Author & Admin Web Portal**: [https://bookleaf-web-olive.vercel.app/](https://bookleaf-web-olive.vercel.app/)
* **Live NestJS REST API**: [https://bookleaf-server-kq8x.onrender.com/api](https://bookleaf-server-kq8x.onrender.com/api)
* **Interactive OpenAPI / Swagger Documentation**: [https://bookleaf-server-kq8x.onrender.com/api/docs](https://bookleaf-server-kq8x.onrender.com/api/docs)
* **Real-time Event Stream (SSE)**: [https://bookleaf-server-kq8x.onrender.com/api/realtime/stream](https://bookleaf-server-kq8x.onrender.com/api/realtime/stream)

---

## 📖 Table of Contents
1. [Executive Summary](#executive-summary)
2. [Test Login Credentials](#test-login-credentials)
3. [Architecture Overview](#architecture-overview)
4. [AI Integration & Prompt Strategy](#ai-integration--prompt-strategy)
5. [API Specification](#api-specification)
6. [Local Setup & Running](#local-setup--running)
7. [Production Trade-offs & Limitations](#production-trade-offs--limitations)

---

## 1. Executive Summary

BookLeaf Publishing handles **1,200+ books monthly** across a catalog of **22,000+ titles**. This application solves author communication at scale through:
- **Author Portal**: Isolated author dashboard showing published books (MRP, sales, accrued/paid/pending royalties), query submission with file attachments, and real-time correspondence updates.
- **Admin Operations Desk**: Incoming query queue filterable by status, priority, and category, urgency-first ordering, AI auto-classification, priority scoring, on-demand Valkey-cached AI response drafts, and internal ops notes.
- **Durable Ingestion**: Queries commit to Neon PostgreSQL before entering BullMQ to prevent message loss.
- **80/20 Transparent Royalty Engine**: Authors track live sales and progress toward the quarterly ₹1,000 disbursement threshold.

---

## 2. Test Login Credentials

All 10 authors and 18 books from `bookleaf_sample_data.json` are pre-seeded in the database.

| Account Type | Name | Email | Password | Role / Identifier | Key Characteristics |
|---|---|---|---|---|---|
| **Ops Admin** | Operations Desk | `admin@bookleaf.pub` | `Password123!` | `ADMIN` | Global triage queue, overrides, AI response studio |
| **Author (Active)** | Priya Sharma | `priya.sharma@email.com` | `Password123!` | `AUTH001` | 2 live titles, ₹3,450 pending royalty (threshold met), bank linked |
| **Author (Production)** | Sneha Kulkarni | `sneha.kulkarni@email.com` | `Password123!` | `AUTH007` | In Production (Cover Design), ₹0 royalty, bank pending link |
| **Author (High Vol)** | Rohit Kapoor | `rohit.kapoor@email.com` | `Password123!` | `AUTH002` | 2,079 copies sold, ₹4,150 pending royalty, bank linked |
| **Authors 3–10** | *Amit, Neha, Vikram, etc.* | `<firstname>.<lastname>@email.com` | `Password123!` | `AUTH003`–`AUTH010` | Full dataset from `bookleaf_sample_data.json` |

---

## 3. Architecture Overview

```
                      ┌────────────────────────────────────────┐
                      │    Next.js 16 Frontend (Port 3000)     │
                      │   Aceternity UI • Tailwind • Socket.io │
                      └───────────────────┬────────────────────┘
                                          │ REST API / WebSocket
                      ┌───────────────────▼────────────────────┐
                      │     NestJS API Server (Port 4000)      │
                      │  JWT Auth Guard • RBAC • Global Pipes  │
                      └───┬────────────────┬────────────────┬──┘
                          │                │                │
            ┌─────────────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
            │  Neon PostgreSQL   │  │   BullMQ    │  │ Aiven Valkey│
            │   Prisma ORM 6     │  │ Async Queue │  │  TLS Cache  │
            │ (Durable Ingest)   │  │ Auto-Triage │  │ (1h Drafts) │
            └────────────────────┘  └──────┬──────┘  └─────────────┘
                                           │
                                    ┌──────▼──────┐
                                    │  Groq LLM   │
                                    │ (Llama 3.1) │
                                    └─────────────┘
```

### Architectural Decisions & Rationale:
1. **Durable Database-First Ingestion**:
   - Ticket submissions always commit to PostgreSQL *before* queueing. If BullMQ or Redis temporarily restarts, zero author messages are lost.
2. **Asynchronous Out-of-Band Triage**:
   - AI categorization and priority scoring run in background BullMQ workers (`triage.worker.ts`). HTTP requests return in `<80ms` without blocking on LLM latency.
3. **On-Demand Valkey Draft Caching**:
   - Draft responses are generated only when an admin opens a ticket, cached in Valkey TLS (`rediss://`) with a 1-hour TTL.
   - When either party sends a message, the cache key (`ai:draft:${ticketId}`) is deleted immediately.
4. **Real-Time Bidirectional Updates**:
   - NestJS WebSocket Gateway (`/events`) with Server-Sent Events (`/api/realtime/stream`) fallback. When an admin replies, the author's screen updates in real time without page refresh.

---

## 4. AI Integration & Prompt Strategy

### 1. Knowledge Base Context Injection
The system implements targeted policy slicing via `knowledge-base.service.ts`:
- **Royalty & Payments**: 80/20 net profit split, quarterly calculations, 45-day window, ₹1,000 threshold, bank transfer requirements.
- **ISBN & Metadata**: BookLeaf publisher imprint vs author imprint, high-priority escalation for mismatched ISBNs.
- **Printing & Quality**: 5–7 day turnaround, in-house + Repro India, free reprints for verified defects.
- **Distribution & Availability**: Amazon, Flipkart, BookLeaf Store, 24–48h stock sync resolution.
- **Production Updates**: Pipeline stages (`Manuscript -> Editing -> Cover Design -> Typesetting -> Proofreading -> ISBN -> Print -> Live`).

### 2. Prompt Engineering & Tone Calibration
Prompts are calibrated against BookLeaf's Communication Tone Guidelines:
- Validates the author's concern upfront with empathy (*"Authors are partners, not customers to manage"*).
- Injects actual figures from the database (e.g. `₹3,450 pending royalty`, `200 copies sold`, `ISBN 978-93-91234-01-2`).
- Never deflects or uses corporate jargon; provides explicit timelines (`within 48 hours` or `5-7 business days`).
- Concludes with concrete next steps, signed as *"Team BookLeaf Publishing"*.

### 3. Token & Cost Optimization
- **Category Slicing**: Rather than injecting the entire 1,500-token Knowledge Base on every call, the system extracts only the section relevant to the ticket's category.
- **History Capping**: Conversation context is strictly capped at the 5 most recent messages.
- **Targeted Entity Injection**: Injects only the specific book referenced by the ticket instead of the author's full catalog.

### 4. API Security & Graceful Degradation
- **Security**: `GROQ_API_KEY` and `OPENAI_API_KEY` are read exclusively by the NestJS backend via `ConfigService`. Neither key is ever exposed to client bundles.
- **Three-Tier Fallback Engine**:
  1. Primary: Groq (`llama-3.1-8b-instant`) with JSON mode for sub-second responses.
  2. Secondary: OpenAI (`gpt-4o-mini`) fallback if Groq experiences rate limits.
  3. Tertiary: High-fidelity deterministic heuristic rule engine (`ai.service.ts`), ensuring 100% operational uptime even during complete LLM outages.

---

## 5. API Specification & Swagger Documentation

Interactive OpenAPI / Swagger documentation with request schemas, responses, and live sandbox execution is hosted at:
👉 **`http://localhost:4000/api/docs`**

All protected endpoints require `Authorization: Bearer <JWT_TOKEN>` (which can be authorized directly inside Swagger UI using the **Authorize** button).

### Authentication & Profiles
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | Public | Register new author account |
| `POST` | `/api/auth/login` | Public | Authenticate with email/password, returns JWT & user profile |
| `GET` | `/api/auth/me` | Authenticated | Return currently authenticated user profile |
| `GET` | `/api/auth/users` | Admin Only | List registered users with role filter |

### Books & Catalog
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/books` | Author | Retrieve all books authored by the current user |
| `GET` | `/api/books/summary` | Author | Aggregate sales, royalty earned, paid, pending, and threshold status |
| `GET` | `/api/books/:id` | Author / Admin | Fetch detailed book record with ISBN and sales data |
| `GET` | `/api/books/admin/all` | Admin Only | Full catalog overview across all authors |

### Tickets & Operations (AI-First)
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/tickets` | Author | Submit support query (auto-dispatches BullMQ triage & ReAct resolution) |
| `GET` | `/api/tickets` | Author / Admin | Author gets their tickets; Admin gets all queue tickets |
| `GET` | `/api/tickets/:id` | Author / Admin | Ticket thread (hides internal notes from author) |
| `POST` | `/api/tickets/:id/messages` | Author / Admin | Send reply or log confidential internal ops note |
| `GET` | `/api/tickets/:id/ai-draft` | Admin Only | Generate/fetch Valkey-cached AI draft response |
| `PATCH` | `/api/tickets/:id/status` | Admin Only | Update workflow status (`OPEN`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`) |
| `PATCH` | `/api/tickets/:id/override` | Admin Only | Manually override AI category or priority rating |
| `PATCH` | `/api/tickets/:id/assign` | Admin Only | Assign ticket to an admin operator |
| `POST` | `/api/tickets/:id/ai-reclassify` | Admin Only | Force AI re-triage via BullMQ |
| `POST` | `/api/tickets/:id/ai-resolve` | Authenticated | Trigger autonomous ReAct AI resolution loop & root-cause analysis |

### Real-time & Knowledge Base
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/knowledge-base` | Public | Full BookLeaf publishing policy codex |
| `GET` | `/api/knowledge-base/:category` | Public | Category-specific knowledge slice |
| `GET` | `/api/realtime/stream` | Public | Server-Sent Events (SSE) stream for live ticket updates |

---

## 6. Local Setup & Running

### Prerequisites
- Node.js `20.x` or higher
- `pnpm` `10.x`
- Neon PostgreSQL database connection string
- Aiven Valkey / Redis instance connection string

### 1. Backend (`bookleaf-server`)
```bash
cd bookleaf-server
pnpm install

# Push schema to database
npx prisma db push

# Seed 10 authors, 18 books, starter tickets & KB articles
pnpm run seed

# Start server
pnpm run start:prod
# Backend runs on http://localhost:4000/api
```

### 2. Frontend (`bookleaf-web`)
```bash
cd bookleaf-web
pnpm install

# Build optimized production bundle
pnpm run build

# Start frontend
pnpm run start
# Frontend runs on http://localhost:3000
```

---

## 7. Production Trade-offs & Limitations

1. **In-Memory WebSocket Gateway Scaling**:
   - Currently uses Socket.io with an in-process adapter. In multi-instance Kubernetes deployments, a Redis adapter (`@socket.io/redis-adapter`) should be plugged into Valkey for cross-pod pub/sub.
2. **File Storage**:
   - The assignment requirements specify attachment support as UI-only. For full production, an S3/Cloudflare R2 pre-signed URL upload pipeline would be added for author defect images.
3. **Queue Concurrency Tuning**:
   - Triage worker is configured with concurrency of 5. For extreme volume spikes, BullMQ dynamic autoscaling based on queue depth can be enabled.
