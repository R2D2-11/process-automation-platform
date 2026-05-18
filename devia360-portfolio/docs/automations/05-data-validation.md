# 05_Data Validation & Business Rule Engine

> How I reduced data entry errors from ~50% of production orders to near-zero using a centralized validation architecture.

## The problem

When 15 people across 6 departments enter data into a shared system, errors are not a possibility they are a certainty. Before DEVIA, I personally cleaned the data in a shared Excel-Kanban file and found that roughly **1 in 2 production orders** had some form of data error: wrong lot numbers, duplicate OP numbers, missing fields, values outside valid ranges.

These errors cascaded:
- A wrong lot number in the production log meant the certificate of analysis referenced the wrong batch
- A duplicate OP number caused confusion about which order was being tracked
- Missing fields meant someone had to chase down the information later, adding delays

## The solution

### Validation at every boundary

DEVIA uses **Pydantic v2** to enforce strict type checking and business rules on every piece of data that enters the system, at three levels:

```
Level 1: Type & Format        Level 2: Business Rules         Level 3: Cross-Entity
─────────────────────          ─────────────────────           ─────────────────────
"Is this a number?"            "Is this OP number             "Does this material
"Is this date valid?"           already taken?"                have a valid SDS?"
"Is this field empty?"         "Is this value within          "Is this user allowed
                                the spec limits?"              to modify this stage?"
```

### Illustrative pattern: layered validation

```python
# Simplified illustration — not production code

from pydantic import BaseModel, Field, field_validator
from typing import Optional
from datetime import date

class ProductionOrderCreate(BaseModel):
    """
    Every field is typed. Every constraint is explicit.
    If the data doesn't match, the request is rejected 
    before it ever touches the database.
    """
    op_number: str = Field(..., min_length=1, max_length=20)
    product_id: int
    lot_number: str = Field(..., min_length=1)
    operator_id: int
    load_weight_kg: float = Field(..., gt=0)
    order_type: str = Field(..., pattern="^(sale|inventory)$")
    
    @field_validator("op_number")
    @classmethod
    def op_number_must_be_alphanumeric(cls, v: str) -> str:
        if not v.replace("-", "").isalnum():
            raise ValueError("OP number must be alphanumeric")
        return v.upper()

# At the router level: check for duplicates before insert
async def create_op(data: ProductionOrderCreate, db: Session):
    # Business rule: no duplicate OP numbers
    existing = db.query(OP).filter(OP.op_number == data.op_number).first()
    if existing:
        raise HTTPException(409, f"OP {data.op_number} already exists")
    
    # Business rule: operator must be active
    operator = db.query(User).filter(User.id == data.operator_id).first()
    if not operator or not operator.is_active:
        raise HTTPException(400, "Operator not found or inactive")
    
    # All checks passed — safe to create
    new_op = OP(**data.model_dump())
    db.add(new_op)
    db.commit()
    return new_op
```

### The permission–step–role matrix

The most complex validation in the system isn't about data types it's about **who can do what, and when.** Every state transition in the Kanban pipeline is checked against a centralized matrix:

```
┌──────────────┬───────────────────┬──────────────────┐
│     Role     │   Allowed From    │   Allowed To     │
├──────────────┼───────────────────┼──────────────────┤
│ Analyst      │ Reception         │ Fab Sample       │
│ Analyst      │ QC Review         │ Adjustment       │
│ Analyst      │ QC Review         │ Final Release    │
│ Supervisor   │ Fab Sample        │ QC Review        │
│ Supervisor   │ Final Release     │ Packaging        │
│ Admin        │ Adjustment        │ Fab Sample (←)   │
│ Admin        │ Any               │ Any              │
└──────────────┴───────────────────┴──────────────────┘
```

This matrix is evaluated on **every single transition.** It's not middleware that can be bypassed, it's in the core transition function. If the check fails, the API returns a clear error explaining why the transition was denied.

### Approval workflow for exceptions

Sometimes a legitimate need exists to do something the rules don't allow like correcting an OP number after it's been assigned. Instead of giving users blanket edit access (which is how errors happened in Excel), DEVIA uses an **approval request system:**

1. User requests the change through the Notifications module
2. The request goes to an administrator's approval queue
3. Admin approves or rejects, with a record of who decided and when
4. If approved, the system makes the change with an audit entry

This keeps the data clean while acknowledging that rules need exceptions, handled through process, not workarounds.

## Impact

| Metric | Before | After |
|--------|--------|-------|
| Data entry errors | ~50% of OPs needed cleanup | Near-zero |
| Duplicate OP numbers | Discovered manually, after confusion | Blocked at entry |
| Unauthorized modifications | Anyone could edit any cell in Excel | Permission matrix per action |
| Error response time | Discovered hours/days later | Instant rejection with explanation |
| Exception handling | Informal ("just fix it") | Formal approval workflow with audit trail |

## Why this matters beyond manufacturing

Data validation and access control are the backbone of any platform that processes external submissions at scale.
The cost of bad data is the same: wasted time, lost trust, and cascading errors downstream.

---

← [Previous: Document Management](04-document-management.md) · → [Next: Analytics Engine](06-analytics-engine.md)
