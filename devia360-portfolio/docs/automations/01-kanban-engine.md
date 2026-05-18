# 01 — Kanban Production Engine

> How I replaced a physical instruction sheet and a broken Excel file with a 7-stage state machine that processes ~70 production orders per month.

## The problem

At EQUIMSA, every production order (OP) followed a sequence of stages — from raw material reception through fabrication, quality testing, and final packaging. Before DEVIA, tracking this was done with:

1. **A physical paper sheet** with instructions that literally traveled between the lab and the production floor. If the sheet was on someone's desk, no one else knew the order's status.
2. **A shared Excel file** I introduced as an improvement — but it created new problems. Rows were accidentally deleted, data was entered incorrectly (~50% of OPs needed manual cleanup), and there was no way to know who changed what or when.
3. **Phone calls.** Knowing where an order stood required calling 1–2 people across 3 departments (lab, production, post-fabrication). Each inquiry took 10–15 minutes.

**Result:** ~3 OPs per week got stuck in the pipeline without anyone noticing. Dead time accumulated silently.

## The solution

A **7-stage Kanban board** where each production order is a card that moves through a defined pipeline. Every transition is governed by rules.

### The 7 stages

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ 1. Recep │──▶│ 2. Fab   │──▶│ 3. QC    │──▶│ 4. Adjust│
│   ción   │   │  Sample  │   │  Review  │   │  (if any)│
└──────────┘   └──────────┘   └──────────┘   └────┬─────┘
                                                   │
                    ┌──────────────────────────────┘
                    ▼
┌──────────┐   ┌──────────┐   ┌──────────┐
│ 5. Final │──▶│ 6. Pack  │──▶│ 7. Close │
│  Release │   │  & Ship  │   │          │
└──────────┘   └──────────┘   └──────────┘
```

### Conditional logic

Not every order follows a straight line. The engine supports:

- **Advance** — Move to the next stage when conditions are met (e.g., QC results within limits)
- **Rollback** — Return to a previous stage when something fails (e.g., adjustment needed after QC)
- **Conditional branching** — Skip stages or require additional steps based on product type or test results

### Permission–step–role matrix

Every transition is evaluated against a centralized matrix:

```
Can User X move OP Y from Stage A to Stage B?
```

This checks three things simultaneously:
1. **Role** — Does the user's role have permission for this transition?
2. **Step** — Is this transition valid from the current stage?
3. **Data** — Are all required fields populated for the target stage?

If any check fails, the transition is blocked with a specific error message.

### Illustrative implementation pattern

```python
# This is a simplified, rewritten illustration — not production code

from enum import IntEnum
from pydantic import BaseModel

class Stage(IntEnum):
    RECEPTION = 1
    FAB_SAMPLE = 2
    QC_REVIEW = 3
    ADJUSTMENT = 4
    FINAL_RELEASE = 5
    PACKAGING = 6
    CLOSED = 7

class TransitionRequest(BaseModel):
    op_id: int
    target_stage: Stage
    user_role: str

# Centralized permission matrix
ALLOWED_TRANSITIONS: dict[str, list[tuple[Stage, Stage]]] = {
    "analyst": [
        (Stage.RECEPTION, Stage.FAB_SAMPLE),
        (Stage.QC_REVIEW, Stage.ADJUSTMENT),
        (Stage.QC_REVIEW, Stage.FINAL_RELEASE),
    ],
    "supervisor": [
        (Stage.FAB_SAMPLE, Stage.QC_REVIEW),
        (Stage.FINAL_RELEASE, Stage.PACKAGING),
        (Stage.PACKAGING, Stage.CLOSED),
    ],
    # rollback permissions
    "admin": [
        (Stage.ADJUSTMENT, Stage.FAB_SAMPLE),  # rollback
    ],
}

def can_transition(current: Stage, request: TransitionRequest) -> bool:
    allowed = ALLOWED_TRANSITIONS.get(request.user_role, [])
    return (current, request.target_stage) in allowed
```

### Real-time visibility

The Kanban board polls the backend every 30–60 seconds. Supervisors see:

- **All active OPs** organized by stage
- **Who is assigned** to each order
- **Time in current stage** — orders sitting too long are visually flagged
- **Blocked orders** — highlighted immediately when they can't advance

No phone calls. No walking to someone's desk. No "I thought Juan had it."

## Impact

| Metric | Before | After |
|--------|--------|-------|
| Status check | 10–15 min (phone calls) | Instant |
| Stuck OPs (undetected) | ~3/week (~12/month) | 0 |
| Data entry errors | ~50% of OPs | Near-zero |
| Dead time from lack of visibility | Unmeasured but pervasive | Supervisors act on delays in real time |

## Why this matters beyond manufacturing

This is a **submission pipeline with quality gates** — the same pattern used in:

- **Bug bounty triage:** Reports arrive → initial review → validation → response → resolution → close. Reports get stuck. Duplicates need routing. SLAs need enforcement.
- **Support ticket systems:** Tickets move through stages with role-based routing, escalation rules, and time-based alerts.
- **CI/CD pipelines:** Builds progress through stages with conditional gates, rollbacks on failure, and audit trails.

The specific domain is chemical manufacturing. The engineering pattern is universal.

---

← [Back to README](../../README.md) · → [Next: LIMS Quality Control](02-lims-quality-control.md)
