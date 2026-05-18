# System Architecture Overview

> Technical decisions, trade-offs, and the reasoning behind DEVIA 360's design.

## Design principles

DEVIA was built under specific constraints that shaped every architectural decision.

1. **Solo developer**. The architecture had to be maintainable by one person. No microservices, no Kubernetes, no over-engineering.
2. **Non-technical users**. Operators and chemists are the primary users. The UI had to be immediately understandable without training.
3. **Reliability over novelty**. This is production software running a real plant. Boring technology that works beats cutting-edge technology that breaks.
4. **Incremental delivery**. Modules were shipped one at a time, each solving an immediate pain point. No "big bang" launch.

## Architecture: Modular monolith

```
┌─────────────────────────────────────────────────────┐
│                      NGINX                          │
│                SSL/TLS Termination                  │
│                  Reverse Proxy                      │
└────────────────────────┬────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
┌───────────────────┐          ┌───────────────────────────────┐
│    Static Files   │          │         FastAPI               │
│    (Frontend)     │          │     Application Server        │
│                   │          │                               │
│  Vanilla JS       │          │  ┌────────────────────────┐   │
│  HTML/CSS         │  REST    │  │      15 Routers        │   │
│  150+ API calls   │ ◀─────▶ │  │                        │   │
│  Polling 30-60s   │          │  │  kanban    lims_mp     │   │
│  Manual cache     │          │  │  pedidos   lims_fab    │   │
│  URL state        │          │  │  densidad  lims_pt     │   │
│                   │          │  │  certs     docs_mp     │   │
└───────────────────┘          │  │  calibr    sam         │   │
                               │  │  inv_lab   historial   │   │
                               │  │  ident     tendencias  │   │
                               │  │  notif     arquitect   │   │
                               │  └─────────┬──────────────┘   │
                               │            │                  │
                               │  ┌─────────▼──────────────┐   │
                               │  │   Shared Services      │   │
                               │  │                        │   │
                               │  │  • Auth (JWT)          │   │
                               │  │  • Permission Matrix   │   │
                               │  │  • Audit Logger        │   │
                               │  │  • Email (SMTP/Gmail)  │   │
                               │  │  • File Manager        │   │
                               │  └─────────┬──────────────┘   │
                               └────────────┼──────────────────┘
                                            │
                         ┌──────────────────┼──────────────────┐
                         │                  │                  │
                         ▼                  ▼                  ▼
               ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
               │  PostgreSQL  │   │    SMTP      │   │  File System │
               │              │   │   (Gmail)    │   │              │
               │  SQLAlchemy  │   │              │   │  SDS docs    │
               │  2.0 + ORM   │   │  Automated   │   │  Signatures  │
               │              │   │  alerts &    │   │  Templates   │
               │  Alembic     │   │  notifs      │   │  Exports     │
               │  migrations  │   │              │   │              │
               └──────────────┘   └──────────────┘   └──────────────┘
```

### Why a modular monolith?

- **One process to deploy, monitor, and debug.** With a solo developer, operational complexity is the enemy.
- **Shared database access** across modules without network calls. The Kanban module can check LIMS results. The certificate module can pull data from both.
- **15 routers provide logical separation** without the overhead of 15 services. Each router owns its domain, its schemas, and its endpoints.

If DEVIA ever needed to scale horizontally (it doesn't — 15 users is well within a single server's capacity), individual routers could be extracted into services. But premature decomposition would have slowed development dramatically.

## Key technical decisions

### Authentication: JWT (stateless)

**Why:** No session store needed. The server verifies tokens without database lookups on every request. Simple, proven, and sufficient for an internal tool with 15 users.

**Implementation:** Tokens carry the user's ID, role, and permissions. The permission–step–role matrix is evaluated server-side on every state transition.

### Database: PostgreSQL + SQLAlchemy 2.0

**Why:** Relational data (products → lots → OPs → test results → certificates) with strong consistency requirements. SQLAlchemy 2.0's async-compatible ORM with type hints matches FastAPI's patterns.

**Migrations:** Alembic handles schema evolution. Every change is versioned and reversible.

### Frontend: Vanilla JS (no framework)

**Why:** This was a deliberate trade-off. A React/Vue app would add build tooling, bundling, and a learning curve for potential future maintainers. Vanilla JS with direct REST calls is:
- Zero build step
- Debuggable by anyone who knows basic JavaScript
- Fast enough for 15 users with polling-based updates

**Trade-offs accepted:**
- More verbose DOM manipulation code
- Manual cache management (explicit invalidation on state changes)
- No component reusability patterns (some code duplication across modules)

For this team size and use case, these trade-offs are correct.

### Real-time updates: Polling (not WebSockets)

**Why:** WebSockets add connection management complexity, reconnection logic, and a different programming model. Polling every 30–60 seconds provides "good enough" real-time for a manufacturing floor where decisions happen on the scale of minutes, not milliseconds.

**If I were building this for 1,000 users or sub-second requirements,** WebSockets or SSE would be necessary. For 15 users checking order status, polling is simpler and sufficient.

### File processing: Server-side with OpenPyXL

**Why:** Commercial order imports (TSV/CSV, 39+ fields) and bulk XLSX exports can involve files up to 50 MB. Processing these client-side would be slow and unreliable. Server-side processing with OpenPyXL handles the heavy lifting and returns structured results.

## Data flow example: Production order lifecycle

```
User creates OP via form
        │
        ▼
┌─────────────────┐
│ Pydantic v2     │──▶ Type validation, format checks, business rules
│ Schema          │    (OP number not duplicate, operator active, etc.)
└────────┬────────┘
         │ Valid
         ▼
┌─────────────────┐
│ Permission      │──▶ Can this user create OPs?
│ Matrix Check    │
└────────┬────────┘
         │ Authorized
         ▼
┌─────────────────┐
│ Database Insert │──▶ PostgreSQL via SQLAlchemy
│ + Audit Log     │    Activity logged with user, timestamp, action
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Kanban Board    │──▶ All users see the new OP on next poll (30-60s)
│ (via polling)   │
└────────┬────────┘
         │
         ▼
   OP moves through 7 stages
   (each transition re-checks permissions + validates required data)
         │
         ▼
┌─────────────────┐
│ LIMS Results    │──▶ Auto-validated against product specs
│ Attached        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Certificate     │──▶ Auto-generated from validated data
│ Generated       │    Routed for digital signature
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ OP Closed       │──▶ Archived in Historial Maestro
│                 │    RFT calculated, cycle times recorded
└─────────────────┘
```

## Audit trail

Every significant action in the system is logged:

- **Who** performed the action (user ID, role)
- **What** they did (created, modified, transitioned, approved, rejected)
- **When** they did it (timestamp)
- **What changed** (before/after values where applicable)

This is a regulatory requirement in chemical manufacturing, but it's also just good engineering. When something goes wrong, you need to know what happened, not guess.

## Lessons learned

**Start with the pain, not the architecture.** The first thing I built was the Kanban board because "where is my order?" was the most frequent question on the plant floor. The database schema, the permission system, the LIMS module, all came later.

**Validation is the feature, not a feature.** In a system where bad data has real-world consequences (wrong lot on a certificate, expired material used in production), validation isn't defensive programming, it's the core value proposition.

**Boring technology is a gift.** PostgreSQL, FastAPI, vanilla JS none of these are exciting. All of them are well-documented, well-understood, and reliable. For a solo developer supporting a production system, that matters more than any technical novelty.

---

← [Back to README](../README.md)
