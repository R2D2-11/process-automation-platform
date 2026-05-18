# 03_Certificate of Analysis Lifecycle

> How I automated the generation, routing, and digital signing of ~70 certificates per month. Replacing a manual loop of Excel templates, email chains, and physical signatures.

## The problem

Every production order that ships to a customer needs a Certificate of Analysis (CoA) a document that proves the product meets its specifications. At EQUIMSA, producing one certificate looked like this:

1. **A chemist opens a blank Excel template**
2. **Manually copies test results** from the lab Excel files into the template
3. **Formats the document** (product name, lot, date, customer, parameters, limits, results)
4. **Emails it** to the department head for review
5. **The department head reviews, approves, and forwards** it to the warehouse
6. **The warehouse matches it** with the shipment

**Time per certificate:** ~5 minutes of manual assembly and routing.
**Volume:** ~70 per month (one per production order).
**Total monthly overhead:** ~350 minutes (~6 hours) of pure manual work that added zero analytical value.

And that's the happy path. Delays in the email chain, missing results, or formatting errors added more time.

## The solution

The certificate lifecycle is now a **managed state machine** inside DEVIA:

```
┌────────────┐      ┌────────────┐     ┌────────────┐      ┌────────────┐
│  Creation  │────▶│  Review &  │────▶│  Digital   │────▶│  Issued    │
│  (auto)    │      │  Assign    │     │  Signature │      │  (ready)   │
└────────────┘      └────────────┘     └────────────┘      └────────────┘
```

### Stage 1: Auto-generation

When a production order reaches the final release stage in the Kanban pipeline, the system already has all the data it needs:

- Product identity, lot number, manufacturing date
- All test results
- Specification limits per parameter

The certificate is assembled automatically. No copy-pasting from spreadsheets.

### Stage 2: Review & assignment

The system shows a queue of certificates pending signature. The department head sees:
- Which certificates are ready
- How long each has been waiting
- Who the recipient (customer) is

This replaces the email chain. Nothing gets lost in an inbox.

### Stage 3: Digital signature

The authorized signer reviews and applies their digital signature within the system. No printing, no scanning, no physical paper routing.

### Stage 4: Issuance

The signed certificate is stored and linked to its production order. Ready for the warehouse, ready for the customer, ready for any future audit.

### Illustrative pattern

```python
# Simplified illustration of the certificate lifecycle — not production code

from enum import Enum
from datetime import datetime
from pydantic import BaseModel
from typing import Optional

class CertStatus(str, Enum):
    DRAFT = "draft"
    PENDING_SIGNATURE = "pending_signature"
    SIGNED = "signed"
    ISSUED = "issued"

class Certificate(BaseModel):
    op_id: int
    product: str
    lot: str
    status: CertStatus = CertStatus.DRAFT
    created_at: datetime
    signed_by: Optional[str] = None
    signed_at: Optional[datetime] = None

def generate_certificate(op_id: int, test_results: list, product_specs: list) -> Certificate:
    """
    Auto-generate a certificate from existing validated data.
    No manual data entry required.
    """
    # All data already exists in the system from the LIMS module
    # The certificate is assembled, not manually filled
    cert = Certificate(
        op_id=op_id,
        product=get_product_name(op_id),
        lot=get_lot_number(op_id),
        created_at=datetime.utcnow(),
    )
    
    # Attach test results (already validated against specs)
    attach_results(cert, test_results, product_specs)
    
    # Move to pending signature queue
    cert.status = CertStatus.PENDING_SIGNATURE
    notify_signers(cert)  # Email alert to authorized signers
    
    return cert
```

## Impact

| Metric | Before | After |
|--------|--------|-------|
| Time per certificate | ~5 minutes manual assembly | Seconds (auto-generated) |
| Monthly total (70 certs) | ~350 min (~6 hours) | Minutes of review + signing |
| Data accuracy | Manual copy-paste from Excel | Data pulled directly from validated LIMS records |
| Routing delays | Lost in email chains | Queue with visibility and wait-time tracking |
| Audit readiness | Scattered files, no central record | Every cert linked to its OP with full history |

## Why this matters beyond manufacturing

This is a **document generation pipeline with approval gates**:

- Data flows in from upstream processes (test results / report details)
- A structured document is assembled automatically (certificate / response template)
- It routes through an approval queue with visibility into bottlenecks (pending signatures / pending reviews)
- The signed output is stored with full traceability (audit trail)


---

← [Previous: LIMS Quality Control](02-lims-quality-control.md) · → [Next: Document Management](04-document-management.md)
