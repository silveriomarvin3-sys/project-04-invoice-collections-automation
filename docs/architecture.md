# Architecture

## High-Level Architecture

```
External Billing System
        │
        ▼
Receive Invoice Event
        │
        ▼
Normalize Invoice
        │
        ▼
Validate Invoice
   ┌────┴─────┐
   │          │
INVALID      VALID
   │          │
   ▼          ▼
Reject      Transform Invoice
Invoice          │
   │             ▼
   ▼      Create Finance Event
Rejected         │
Invoices         ▼
         Find Existing Invoice
                  │
                  ▼
        Invoice Already Exists?
           ┌──────┴──────┐
          YES            NO
           │              │
           ▼              ▼
          Stop      Calculate Score
                           │
                           ▼
                     Assign Priority
                           │
                           ▼
                      Store Invoice
                           │
                           ▼
                Route Collection Priority
                 ┌─────────┼─────────┐
                 ▼         ▼         ▼
                LOW      MEDIUM     HIGH
                 │         │          │
                 │         ▼          ▼
                 │     Finance     Urgent
                 │      Review      Finance
                 │   Notification    Alert
                 │         │          │
                 └─────────┼──────────┘
                           ▼
                 Prepare Finance Delivery
                           │
                           ▼
                    HTTP Delivery
                           │
                           ▼
                    HTTP Succeeded?
                    ┌──────┴──────┐
                   YES            NO
                    │              │
                    ▼              ▼
             Prepare Delivery   Handle Failure
              Status Update          │
                    │                ▼
                    ▼           Should Retry?
              Update Invoice     ┌────┴────┐
                               NO         YES
                                │           │
                                ▼           ▼
                          Final Failure    Wait
                                │           │
                                ▼           ▼
                         Automation     Retry HTTP
                           Errors         Request
                                            │
                                            ▼
                                    Increment Retry
                                         Count
                                            │
                                            ▼
                                    Retry Limit Reached?
                                      ┌─────┴─────┐
                                     YES          NO
                                      │            │
                                      ▼            └──► Retry
                               Final Failure
                                      │
                                      ▼
                               Automation Errors
```

---

## Overview

The **Invoice Intake & Collections Automation** workflow is designed as an event-driven finance operations pipeline.

Instead of treating automation as a simple app-to-app sequence, the workflow separates responsibilities into clear processing stages:

```
Invoice Event
→ Normalize
→ Validate
→ Transform
→ Create Finance Event
→ Duplicate Check
→ Calculate Score
→ Assign Priority
→ Store Invoice
→ Route by Priority
→ Deliver Finance Event
→ Handle Success or Failure
```

Each stage has one clear responsibility, which makes the workflow easier to debug, maintain, test, and extend.

---

## 1. Invoice Intake

The workflow begins with an n8n Webhook node.

```
External Billing System
        ↓
POST /invoice-created
        ↓
Receive Invoice Event
```

The webhook acts as the system entry point.

The incoming payload may contain:

- nested objects
- inconsistent capitalization
- numbers stored as strings
- unnecessary whitespace
- external field names that do not match the internal finance model

Because external data should not be trusted to already match the internal structure, the workflow normalizes the payload before applying business logic.

---

## 2. Normalization Layer

The normalization stage extracts invoice data from the webhook request and converts it into a predictable working structure.

Example incoming payload:

```json
{
  "invoice_number": " INV-2026-00841 ",
  "customer": {
    "company_name": " APEX CONSTRUCTION GROUP ",
    "contact_email": " ACCOUNTS@APEXBUILD.COM "
  },
  "invoice": {
    "amount": "8750.50",
    "currency": "USD",
    "payment_terms": "net30"
  },
  "issued_date": "2026-08-13",
  "due_date": "2026-09-12",
  "account_tier": "business",
  "sales_rep": "Elena Cruz"
}
```

The workflow extracts fields such as:

- `invoiceNumber`
- `company`
- `email`
- `amount`
- `currency`
- `terms`
- `issued_date`
- `due_date`
- `account_tier`
- `sales_rep`

This creates a stable input shape for validation and transformation.

---

## 3. Validation Boundary

Validation happens before the invoice enters the main finance-processing pipeline.

The workflow checks business rules such as:

- Invoice number → required
- Customer company → required
- Billing email → must be valid
- Invoice amount → numeric, greater than 0
- Currency → supported value
- Payment terms → supported value
- Issued date → required
- Due date → required
- Account tier → recognized value

The workflow immediately branches:

```
Validate Invoice
      │
   ┌──┴──┐
   │     │
 VALID  INVALID
   │     │
   ▼     ▼
Continue Reject Invoice
           │
           ▼
     Rejected Invoices
```

Invalid invoices never enter the primary invoice store. This prevents malformed business records from contaminating downstream systems.

---

## 4. Transformation Layer

After validation, the invoice is transformed into the internal finance format.

Example:

```json
{
  "invoiceId": "INV-2026-00841",
  "customerName": "Apex Construction Group",
  "billingEmail": "accounts@apexbuild.com",
  "amountDue": 8750.5,
  "currency": "USD",
  "paymentTermsDays": 30,
  "issuedAt": "2026-08-13",
  "dueAt": "2026-09-12",
  "accountTier": "business",
  "salesOwner": "Elena Cruz",
  "paymentStatus": "unpaid"
}
```

This separates the external billing-system format from the internal workflow representation.

If the external system changes its field names later, only the normalization and transformation layers should need to change.

---

## 5. Finance Event Envelope

The transformed invoice is wrapped inside a standardized finance event.

Example:

```json
{
  "eventId": "evt-...",
  "event": "invoice.created",
  "timestamp": "2026-08-13T...",
  "source": "billing-system",
  "data": {
    "invoiceId": "INV-2026-00841",
    "customerName": "Apex Construction Group",
    "billingEmail": "accounts@apexbuild.com",
    "amountDue": 8750.5,
    "currency": "USD",
    "paymentTermsDays": 30,
    "issuedAt": "2026-08-13",
    "dueAt": "2026-09-12",
    "accountTier": "business",
    "salesOwner": "Elena Cruz",
    "paymentStatus": "unpaid"
  }
}
```

The `eventId` provides a correlation value that can be reused across:

- Invoice records
- Delivery attempts
- Failure logs
- Notifications
- Execution history

---

## 6. Duplicate Detection

Before a new invoice is stored, the workflow checks whether the same invoice ID already exists.

```
Create Finance Event
        ↓
Find Existing Invoice
        ↓
Invoice Already Exists?
     ┌──────┴──────┐
    YES            NO
     │              │
     ▼              ▼
Stop Duplicate   Continue
```

The invoice ID is used as the business identifier. This is different from the event ID.

For example:

- Event ID: `evt-1001` — Invoice ID: `INV-500`
- Event ID: `evt-1002` — Invoice ID: `INV-500`

These are two different events, but they refer to the same invoice.

Duplicate detection prevents the same business record from being stored and processed twice.

---

## 7. Collection Priority Scoring

Valid, non-duplicate invoices are assigned a collection score.

The scoring model is:

| Condition | Points |
|---|---|
| Amount due ≥ $10,000 | +3 |
| Amount due ≥ $5,000 | +2 |
| Enterprise account | +2 |
| Net 60 payment terms | +1 |

Only one amount tier applies.

Example — $65,500 invoice, Enterprise account, Net 60 terms:

| Rule | Points |
|---|---|
| Amount | +3 |
| Enterprise | +2 |
| Net 60 | +1 |
| **Total** | **6** |

Priority is assigned using:

| Score | Priority |
|---|---|
| 0–2 | LOW |
| 3–4 | MEDIUM |
| 5+ | HIGH |

This means collection priority is derived from visible business rules rather than supplied directly by the external billing system.

---

## 8. Persistent Invoice Storage

Invoices are stored in Google Sheets as a lightweight finance-operations datastore.

The main invoice table stores fields such as:

- Invoice ID
- Created At
- Customer Name
- Billing Email
- Amount Due
- Currency
- Payment Terms Days
- Issued At
- Due At
- Account Tier
- Sales Owner
- Payment Status
- Event ID
- Priority
- Score
- Delivery Status
- Delivered At

Only validated and non-duplicate invoices are stored in this table.

---

## 9. Priority Routing

The workflow uses a Switch node to route invoices according to their calculated priority.

```
                  ┌── LOW
                  │
Priority Router ──┼── MEDIUM
                  │
                  └── HIGH
```

**LOW**

```
LOW
↓
Standard Processing
```

Low-priority invoices continue through the standard processing path without an urgent finance notification.

**MEDIUM**

```
MEDIUM
↓
Finance Review
↓
Finance Review Notification
```

Medium-priority invoices generate a finance review notification.

**HIGH**

```
HIGH
↓
Urgent Finance Review
↓
Urgent Finance Alert
```

High-priority invoices trigger an urgent finance alert before continuing through the delivery pipeline.

---

## 10. Finance Delivery

After priority routing, the workflow prepares a processed finance event for downstream delivery.

The delivery payload includes fields such as:

- Event ID
- Invoice ID
- Customer Name
- Amount Due
- Priority
- Score
- Action
- Alert Requirement
- Timestamp

The event represents the result of workflow processing instead of the original raw invoice payload.

Example:

```json
{
  "eventId": "evt-...",
  "event": "invoice.processed",
  "timestamp": "2026-08-13T...",
  "source": "northstar-ar-workflow",
  "invoiceId": "INV-2026-00841",
  "customerName": "Apex Construction Group",
  "amountDue": 8750.5,
  "priority": "MEDIUM",
  "score": 3,
  "action": "finance-review",
  "requiresAlert": false
}
```

This event is sent to a downstream HTTP API.

---

## 11. HTTP Success Detection

The HTTP Request node is configured to continue execution when delivery fails.

The workflow then checks whether the returned result contains an error.

```
Deliver Finance Event
        ↓
HTTP Succeeded?
    ┌─────┴─────┐
   YES          NO
    │            │
    ▼            ▼
Mark Invoice   Handle Failure
Delivered
```

Successful delivery updates the existing invoice record.

---

## 12. Error Normalization

Technical delivery failures are transformed into a predictable internal error structure.

The normalized failure includes information such as:

- `success`
- `stage`
- `eventId`
- `timestamp`
- `status`
- `message`

Example:

```json
{
  "success": false,
  "stage": "finance-delivery",
  "eventId": "evt-...",
  "timestamp": "2026-08-13T...",
  "status": 503,
  "message": "Service unavailable"
}
```

This allows the rest of the workflow to reason about failures consistently.

---

## 13. Retry Classification

Not every HTTP failure should be retried.

The workflow classifies temporary failures as retryable.

**Retryable statuses include:**

- 408 Request Timeout
- 429 Too Many Requests
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout

**Examples of failures that should not be blindly retried include:**

- 400 Bad Request
- 401 Unauthorized
- 403 Forbidden
- 404 Not Found

This prevents unnecessary repeated requests when the problem is unlikely to resolve automatically.

---

## 14. Bounded Retry Strategy

Retryable failures do not retry indefinitely.

The retry system uses:

```
Should Retry?
↓
Wait
↓
Retry HTTP Request
↓
Increment Retry Count
↓
Retry Limit Reached?
```

The maximum retry count is **3**.

Conceptually:

```
Initial Attempt
↓ fails

Retry 1
↓ fails

Retry 2
↓ fails

Retry 3
↓ fails

Final Delivery Failure
```

The delay prevents the workflow from repeatedly hitting an unavailable or rate-limited external service immediately.

The retry limit prevents infinite loops.

---

## 15. Final Failure Logging

If downstream delivery ultimately fails, the workflow records the failure in a separate automation error log.

The system therefore maintains separate operational data categories:

- **Invoices** → valid business records
- **Rejected Invoices** → invalid incoming business data
- **Automation Errors** → technical workflow or delivery failures

This separation makes failures easier to investigate without polluting the main invoice dataset.

---

## 16. Delivery Status Tracking

When downstream delivery succeeds, the workflow prepares a status-update payload.

Example:

```json
{
  "Invoice ID": "INV-2026-00841",
  "Delivery Status": "Delivered",
  "Delivered At": "2026-08-13T..."
}
```

The workflow matches the existing invoice using its invoice ID and updates that record rather than creating another row.

---

## Architectural Principles

### Separation of Concerns

Each major section has one clear responsibility.

| Stage | Responsibility |
|---|---|
| Webhook | receive data |
| Normalize | create predictable input |
| Validate | enforce business rules |
| Transform | create internal representation |
| Create Event | create standardized event envelope |
| Duplicate Check | prevent repeated business records |
| Scoring | calculate collection priority |
| Routing | select business behavior |
| Storage | persist operational data |
| HTTP Delivery | communicate with external system |
| Retry System | recover from temporary failures |
| Logging | preserve rejected data and technical failures |

This reduces coupling and makes the workflow easier to maintain.

### Fail Fast

Invalid invoices are rejected before expensive downstream processing occurs.

**Preferred:**

```
Bad Data
↓
Validate
↓
Reject Immediately
```

**Instead of:**

```
Bad Data
↓
Store
↓
Notify
↓
Deliver
↓
Discover Problem Later
```

### Idempotency

Invoice IDs are checked before new records are stored.

Repeated webhook deliveries therefore do not automatically create duplicate invoices.

### Explicit Business Rules

Collection priority is calculated from visible and documented rules.

The workflow does not rely on hidden or arbitrary routing decisions.

### Controlled Failure Recovery

- Temporary failures may retry.
- Permanent failures stop.
- Retries are delayed and bounded.

This makes the workflow more reliable without risking uncontrolled execution loops.

### Observability

Rejected invoices and technical failures are persisted instead of silently disappearing.

This makes it possible to answer:

- What failed?
- Why did it fail?
- Which invoice was affected?
- Was the error caused by bad business data or an external system?
- Did the workflow retry?
- Did delivery eventually succeed?

---

## Summary

The workflow is designed as more than a simple invoice-forwarding automation.

It acts as a small finance operations system that:

- receives external invoice events
- normalizes inconsistent payloads
- validates business data
- transforms records into an internal finance model
- creates traceable finance events
- prevents duplicate processing
- calculates collection priority
- stores valid invoice records
- routes invoices according to business rules
- sends finance notifications
- delivers processed events to external APIs
- classifies HTTP failures
- retries temporary failures safely
- logs permanent failures
- records rejected invoices separately
- updates delivery status after successful processing

The architecture emphasizes maintainability, explicit business rules, failure handling, and traceability rather than simply connecting applications together.
