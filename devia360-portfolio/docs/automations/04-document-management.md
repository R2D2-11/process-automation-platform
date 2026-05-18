# 04 — Regulatory Document Management

> How I turned a mess of scattered emails and conflicting records into an audit-ready document repository with automated expiration tracking.

## The problem

Chemical manufacturers are required to maintain current Safety Data Sheets (SDS/HDS) for every raw material they use. These documents have a legal shelf life — typically 2 years — and auditors check that they're current, complete, and accessible.

Before DEVIA, the situation was:

- **Purchasing** had their own list of SDS documents
- **The lab** had a different list
- **The two lists didn't match**
- Documents were exchanged via email and stored... in email threads
- **No one knew which documents were expired** until someone manually checked
- **No shared repository** — if you needed an SDS, you searched your inbox and hoped

I spent my first 6 months at EQUIMSA largely focused on "plugging holes" in documentation ahead of a regulatory audit. This was the pain that directly inspired the DOCS module.

## The solution

### Centralized document repository

Every raw material in the system has a documentation profile:

```
Material: Polyalphaolefin PAO-6
├── SDS (Spanish)      ✅ Valid until 2027-03-15
├── SDS (English)      ✅ Valid until 2027-03-15
├── Technical Data Sheet  ✅ Valid until 2026-11-20
├── Supplier Certificate  ⚠️ Expires in 45 days
└── Memo / Notes          ✅ Current
```

Documents are uploaded once and linked to their material. No more email archaeology.

### Automated validity tracking

Every document carries:
- **Upload date**
- **Expiration date** (upload + 2 years by default, configurable)
- **Status**: current, expiring soon (90-day alert), expired, or missing

The system automatically flags:
- ⚠️ **Documents expiring within 90 days** — so Purchasing can request updated versions proactively
- ❌ **Expired documents** — visible at a glance, not buried in a spreadsheet
- 🔴 **Materials with incomplete documentation** — missing SDS, missing in one language, etc.

### Department-level visibility

Instead of Purchasing and the Lab maintaining separate (conflicting) records, both departments see the same source of truth. When Purchasing uploads a new SDS, the Lab sees it immediately. When the Lab notices a document is missing, Purchasing sees the gap.

```
Before:
  Purchasing: "We sent the SDS in March"
  Lab: "We never got it"
  Reality: It was in someone's inbox, expired

After:
  System: Material X — SDS (Spanish) — uploaded 2024-03-15 — 
          expires 2026-03-15 — ⚠️ 90-day alert active
  Everyone: Same information. No disputes.
```

## Impact

| Metric | Before | After |
|--------|--------|-------|
| Document location | Scattered across email inboxes | Single shared repository |
| Expiration tracking | None (discovered during audits) | Automated with 90-day advance alerts |
| Cross-department consistency | Conflicting records | Single source of truth |
| Audit preparation time | Months of manual verification | Always audit-ready |
| Missing document detection | Unknown until someone checked manually | Flagged automatically per material |

## Why this matters beyond manufacturing

Every organization that deals with compliance, certifications, or document lifecycles faces this exact problem:

- **Security certifications** (SOC 2, ISO 27001) require current policies and evidence artifacts with expiration tracking
- **Bug bounty program policies** need version control, visibility across stakeholders, and proactive renewal
- **Vendor/partner documentation** — NDAs, contracts, SLAs — all expire and need tracking

The underlying pattern: **a repository of structured documents with validity windows, proactive alerts, and cross-stakeholder visibility.** The domain doesn't matter — the chaos of untracked document expiration is universal.

---

← [Previous: Certificate Lifecycle](03-certificate-lifecycle.md) · → [Next: Data Validation Engine](05-data-validation.md)
