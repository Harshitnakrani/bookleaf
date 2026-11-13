# BookLeaf Technical Assignment 1: Engineering Approach & Architectural Write-Up

### Candidate Submission &bull; Author Support & Communication Portal

---

### 1. Approach & Priorities

When designing a customer and operations portal for a publishing press processing **1,200+ monthly manuscripts** across a **22,000+ title catalog**, the core engineering challenge is not generating LLM text—it is **state consistency, operational throughput, and deterministic reliability under load**.

I prioritized:
1. **Durable Ingestion Over Synchronous AI Processing**:
   Author inquiries must never be lost if an external AI provider experiences an outage, rate limit, or network spike. I decoupled ticket creation from triage: the API commits the query to PostgreSQL immediately, returning a `<80ms` response to the author. Triage is dispatched asynchronously to a BullMQ worker queue.
2. **On-Demand Caching with Strict Invalidation**:
   Rather than eagerly generating AI draft responses for thousands of incoming tickets (which wastes tokens and API costs on queries that an operator might resolve immediately), drafts are generated **on-demand** when an operator opens a ticket. They are cached in Aiven Valkey TLS with a 1-hour TTL and purged the moment either the author or admin dispatches a reply.
3. **Context Optimization & Grounded Prompting**:
   LLMs hallucinate when flooded with irrelevant data. I built a dynamic Knowledge Base slicer (`knowledge-base.service.ts`) that extracts only the policy slice pertinent to the ticket's category (e.g. 80/20 royalty calculation rules, ISBN imprint differences, or 5–7 day print reprint turnaround). The prompt injects actual database values (author name, bank verification state, specific book sales, and pending royalties) so responses cite facts, not generic fluff.
4. **Resilient Three-Tier Fallback Strategy**:
   Primary execution runs on Groq (`llama-3.1-8b-instant`) for ultra-low latency (<400ms). If Groq rate-limits, it falls back to OpenAI (`gpt-4o-mini`). If both fail, an offline deterministic heuristic rule engine classifies queries and scores priority, guaranteeing 100% platform availability.

---

### 2. Trade-offs Made

- **UI-Only Attachments**: As permitted by the assignment specifications, file attachment handling is implemented on the frontend with simulated metadata rather than integrating AWS S3 or Cloudflare R2 bucket credentials.
- **In-Memory WebSocket Gateway**: Built with NestJS Socket.io and SSE fallback. This provides seamless real-time message streaming without extra infrastructure overhead, though multi-pod horizontal scaling would require connecting Socket.io to Valkey pub/sub.
- **Single-Database Schema Design**: Neon PostgreSQL was chosen to back both relational data (authors, books, tickets, messages, audit logs) and policy documents via Prisma. While full-text search could be offloaded to Elasticsearch or Meilisearch, PostgreSQL indexes easily handle the current volume.

---

### 3. Production Evolution Roadmap

If deploying this system into a multi-region production environment at BookLeaf, I would implement:
1. **Semantic RAG with pgvector**:
   Embed author queries and historical resolved tickets into vector space (`pgvector`). This would allow the system to cite past resolutions by senior operators for tricky corner cases.
2. **Autonomous "Auto-Resolve" for Low-Risk Queries**:
   For simple queries with high classification confidence (>95%)—such as "When is the next royalty payout date?" or "How do I update my author bio?"—the system could automatically dispatch the AI draft after a 5-minute operator review window, dramatically reducing backlog.
3. **Pre-Signed Cloudflare R2 Storage**:
   Direct-to-storage presigned upload URLs for high-resolution photographs of misprints and packaging damage, paired with computer vision inspection.
4. **Distributed Telemetry & Dead-Letter Queues (DLQ)**:
   Integration with OpenTelemetry and a dedicated BullMQ Dead-Letter Queue to alert on failed jobs or recurring classification drift.
