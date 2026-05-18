# DEVIA 360

**A full-stack internal operations platform that replaced paper, spreadsheets, and phone calls across an entire chemical manufacturing plant.**

Built solo. Deployed in production. Used daily by 15 people across 6 departments.

---

## What is this?

DEVIA 360 is a web-based platform I designed and built from scratch for [EQUIMSA](https://equimsa.com), a tribological products manufacturer in Monterrey, Mexico. It covers the full operational lifecycle: production tracking, laboratory quality control, regulatory document management, commercial order processing, and historical analytics.

This repository documents the **architecture, automation strategies, and measurable impact** of the project. The source code is proprietary. What you'll find here are architecture diagrams, process documentation, illustrative code snippets (rewritten, not production extracts), and quantified before/after metrics.

## The problem I solved

Before DEVIA, the plant ran on:

- **Physical instruction sheets** that traveled back and forth between the lab and the production floor
- **Disconnected Excel files** where ~50% of production orders needed manual data cleanup
- **Phone calls** to 1–2 people across 3 departments just to check the status of a single order
- **Email threads** as the "repository" for regulatory safety documents with conflicting records between departments
- **No proactive alerts** for re-inspections, document expirations, or stuck orders
- **Zero trend analysis** quality parameters were only reviewed after something failed

~3 production orders per week got stuck in the pipeline without anyone noticing.

## What I built

| Module | What it does |
|--------|-------------|
| **DEVIO**: Production Engine | 7-stage Kanban with conditional advance/rollback logic. Real-time visibility for supervisors. |
| **DEVIC**: LIMS | Raw material intake, in-process QC, finished product analysis. Auto-validation against product-specific limits. Managed certificate lifecycle with digital signatures. |
| **DOCS**: Regulatory Docs | Centralized SDS repository with 2-year validity tracking and 90-day expiry alerts. |
| **SINTEGRA**: Analytics | Historical view of all closed OPs. RFT (Right First Time) calculation. Parameter trend charts. |
| **Pedidos**: Orders | Automated CSV/TSV import of commercial orders (39+ fields). Assignment, tracking, currency conversion. |

→ Deep dives for each module: [`docs/automations/`](docs/automations/)

## Impact at a glance

| Metric | Before | After |
|--------|--------|-------|
| OP status check | Call 1–2 people, ~10–15 min | Instant (Kanban board) |
| Raw material QC report | ~240 seconds across 3 steps | 30 seconds, single workflow |
| Certificate of analysis | ~5 min manual assembly × 70/month | Auto-generated, seconds each |
| Data entry errors | ~50% of OPs needed cleanup | Near-zero (Pydantic validation) |
| Stuck OPs (undetected) | ~12/month | 0 |
| Re-inspection scheduling | Reactive (emergency-only) | Proactive alerts |
| SDS document tracking | Scattered in emails | Centralized, audit-ready |
| Quality trend analysis | None | Full historical visibility |

## Tech stack

```
Backend:    Python 3.x · FastAPI · Pydantic v2 · SQLAlchemy 2.0 · Alembic · OpenPyXL
Database:   PostgreSQL
Auth:       JWT (stateless) · Role-based permission matrix
Infra:      Nginx (SSL/TLS) · SMTP/Gmail notifications
Frontend:   Vanilla JS · HTML/CSS · 150+ REST endpoints · Polling (30–60s)
Data I/O:   CSV/TSV import (39+ fields) · XLSX bulk export · File system for docs/signatures
```

## Architecture

→ See [`docs/architecture/`](docs/architecture/) for system diagrams and data flow documentation.

```
┌─────────────────────────────────────────────────────────┐
│                      NGINX (SSL/TLS)                    │
└─────────────┬───────────────────────────┬───────────────┘
              │                           │
     ┌────────▼────────┐         ┌─────────▼─────────┐
     │    Frontend     │         │   FastAPI Backend │
     │  (Vanilla JS)   │───────▶│   15 Routers      │
     │  150+ REST calls│◀───────│   ~6,400 lines    │
     └─────────────────┘         └────────┬──────────┘
                                         │
                          ┌──────────────┼──────────────┐
                          │              │              │
                  ┌───────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐
                  │  PostgreSQL  │ │   SMTP    │ │   File    │
                  │  (SQLAlchemy │ │  (Gmail)  │ │  System   │
                  │   + Alembic) │ │           │ │ (SDS/sigs)│
                  └──────────────┘ └───────────┘ └───────────┘
```

## How I work

This project wasn't built in a vacuum. My process:

1. **Observe**. I work full-time as a QC Analyst on the production floor. I see the pain firsthand.
2. **Listen**. I approach the people involved, ask about their process and their problems.
3. **Define**. Weekly meetings to scope solutions, prioritize, and set expectations.
4. **Build**. I develop iteratively, shipping usable modules that get immediate feedback.
5. **Support**. When something breaks in production, I go to the floor and fix it. When it's not urgent, I ask for a detailed description and reproduce it on my machine.

## Repository structure

```
devia360-portfolio/
├── README.md                          ← You are here
├── docs/
│   ├── automations/
│   │   ├── 01-kanban-engine.md        ← Production pipeline automation
│   │   ├── 02-lims-quality-control.md ← Lab information management system
│   │   ├── 03-certificate-lifecycle.md← Certificate generation & digital signatures
│   │   ├── 04-document-management.md  ← SDS tracking & regulatory compliance
│   │   ├── 05-data-validation.md      ← Business rule engine & error prevention
│   │   └── 06-analytics-engine.md     ← RFT calculation & trend analysis
│   └── architecture/
│       └── system-overview.md         ← Architecture decisions & data flow
└── assets/                            ← Diagrams (referenced from docs)
```

## About me

**Carlos Arturo Payén Guzmán**
Physics graduate (UANL) · QC Analyst at EQUIMSA · Solo developer of DEVIA 360

I build tools that make operational chaos manageable. My background in physics gave me the analytical rigor. Building DEVIA taught me that the hardest part of engineering isn't the code, it's understanding what people actually need and shipping something they'll use every day.

📧 arturopayen.r2@gmail.com
🔗 [linkedin.com/in/arturo-payen](https://linkedin.com/in/arturo-payen)

---

*DEVIA 360 is proprietary software developed for EQUIMSA. This portfolio documents architecture and impact without disclosing source code or trade secrets.*
