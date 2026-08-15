# Testing

## Overview

The **Invoice Intake & Collections Automation** workflow was tested across both successful and failure scenarios.

Testing focused on more than confirming the happy path. The workflow was intentionally given:

- valid invoices
- invalid business data
- duplicate invoices
- different collection priorities
- successful HTTP responses
- retryable HTTP failures
- non-retryable HTTP failures
- repeated delivery failures

The goal was to verify that each branch of the workflow behaves predictably and that failures do not silently disappear.

---

## Test Coverage

The following scenarios are covered:

| Test | Expected Behavior | Result |
|---|---|---|
| Valid LOW priority invoice | Process normally through LOW route | PASS |
| Valid MEDIUM priority invoice | Trigger finance review route | PASS |
| Valid HIGH priority invoice | Trigger urgent finance alert | PASS |
| Invalid invoice | Reject before main processing | PASS |
| Duplicate invoice | Prevent second processing attempt | PASS |
| Successful HTTP delivery | Mark invoice as delivered | PASS |
| Retryable HTTP failure | Enter retry mechanism | PASS |
| Non-retryable HTTP failure | Skip retries and log failure | PASS |
| Retry exhaustion | Stop after retry limit and log failure | PASS |

---

# 1. Valid LOW Priority Invoice

## Purpose

Verify that a normal low-value invoice:

- passes validation
- is transformed correctly
- is stored
- receives a LOW priority
- does not generate unnecessary finance alerts
- reaches downstream delivery

## Test Payload

Use:

```text
examples/low-priority-invoice.json
```

Example characteristics:

```text
Amount:        $1,200
Account Tier:  Business
Payment Terms: Net 30
```

## Expected Score

```text
Amount < $5,000    +0
Business Account   +0
Net 30             +0
----------------------
Total                0
```

Expected priority:

```text
LOW
```

## Expected Route

```text
Receive Invoice Event
        ↓
Normalize Invoice
        ↓
Validate Invoice
        ↓
Transform Invoice
        ↓
Create Finance Event
        ↓
Find Existing Invoice
        ↓
Invoice Already Exists?
        ↓ NO
Calculate Score
        ↓
Assign Priority
        ↓
Store Invoice
        ↓
Route Collection Priority
        ↓
LOW
        ↓
Low Priority Processing
        ↓
Prepare Finance Delivery
        ↓
Deliver Finance Event
```

## Expected Result

- Invoice is inserted into the primary invoice store.
- Priority is `LOW`.
- Score is within the LOW range.
- Finance Review Notification is not triggered.
- Urgent Finance Alert is not triggered.
- Finance delivery is attempted.

**Result: PASS**

---

# 2. Valid MEDIUM Priority Invoice

## Purpose

Verify that an invoice meeting the MEDIUM scoring threshold is routed for finance review.

## Test Payload

Use:

```text
examples/medium-priority-invoice.json
```

Example characteristics:

```text
Amount:        $12,500
Account Tier:  Business
Payment Terms: Net 30
```

## Expected Score

```text
Amount >= $10,000    +3
Business Account     +0
Net 30               +0
------------------------
Total                  3
```

Expected priority:

```text
MEDIUM
```

## Expected Route

```text
Calculate Score
        ↓
Assign Priority
        ↓
MEDIUM
        ↓
Medium Priority Processing
        ↓
Finance Review Notification
        ↓
Prepare Finance Delivery
```

## Expected Result

- Invoice passes validation.
- Invoice is stored.
- Score equals `3`.
- Priority equals `MEDIUM`.
- Finance review notification is triggered.
- Workflow continues to finance delivery after notification.

**Result: PASS**

---

# 3. Valid HIGH Priority Invoice

## Purpose

Verify that a high-value enterprise invoice with extended payment terms reaches the urgent finance path.

## Test Payload

Use:

```text
examples/high-priority-invoice.json
```

Example characteristics:

```text
Amount:        $65,500
Account Tier:  Enterprise
Payment Terms: Net 60
```

## Expected Score

```text
Amount >= $10,000    +3
Enterprise Account   +2
Net 60               +1
------------------------
Total                  6
```

Expected priority:

```text
HIGH
```

## Expected Route

```text
Calculate Score
        ↓
Assign Priority
        ↓
HIGH
        ↓
High Priority Processing
        ↓
Urgent Finance Alert
        ↓
Prepare Finance Delivery
```

## Expected Result

- Invoice passes validation.
- Invoice is stored.
- Score equals `6`.
- Priority equals `HIGH`.
- Urgent Finance Alert is triggered.
- Workflow continues to downstream delivery.

**Result: PASS**

---

# 4. Priority Scoring Boundary Tests

## Purpose

Verify that score thresholds produce the correct priority.

The configured thresholds are:

| Score | Priority |
|---:|---|
| 0–2 | LOW |
| 3–4 | MEDIUM |
| 5+ | HIGH |

Important boundary values are therefore:

```text
2 → LOW
3 → MEDIUM

4 → MEDIUM
5 → HIGH
```

These boundaries prevent ambiguous routing between priority levels.

---

# 5. Enterprise Account Normalization

## Purpose

Verify that account-tier capitalization does not incorrectly affect scoring.

During development, the scoring logic exposed a case-sensitivity issue.

For example:

```javascript
"enterprise" === "Enterprise"
```

returns:

```text
false
```

This caused an Enterprise invoice to lose its expected `+2` score.

The comparison was made defensive by normalizing the value before comparison.

Conceptually:

```javascript
String(accountTier)
    .trim()
    .toLowerCase() === "enterprise"
```

## Test Cases

The following values should represent the same account tier after normalization:

```text
enterprise
Enterprise
ENTERPRISE
 enterprise
Enterprise 
```

## Expected Result

Each valid representation should receive:

```text
Enterprise Account → +2
```

**Result: PASS**

---

# 6. Invalid Invoice

## Purpose

Verify that malformed business data is rejected before entering the main processing pipeline.

## Test Payload

Use:

```text
examples/invalid-invoice.json
```

The test payload intentionally contains invalid values such as:

- missing invoice number
- missing company name
- invalid email
- negative invoice amount
- unsupported payment terms
- missing required fields

## Expected Route

```text
Receive Invoice Event
        ↓
Normalize Invoice
        ↓
Validate Invoice
        ↓ FALSE
Reject Invalid Invoice
        ↓
Rejected Invoices
```

## Must NOT Execute

The following stages should not execute:

```text
Transform Invoice
Create Finance Event
Find Existing Invoice
Calculate Score
Store Invoice
Priority Routing
Finance Delivery
```

## Expected Result

- Validation returns FALSE.
- Invoice is not inserted into the primary invoice table.
- Rejected record is written to the rejected-invoice log.
- Downstream delivery is never attempted.

**Result: PASS**

---

# 7. Duplicate Invoice

## Purpose

Verify that repeated delivery of the same invoice does not create duplicate business records.

## Test Payload

Use:

```text
examples/duplicate-invoice.json
```

The same payload is sent twice.

---

## First Request

Expected behavior:

```text
Receive Invoice Event
        ↓
Validate
        ↓
Create Finance Event
        ↓
Find Existing Invoice
        ↓
Invoice Already Exists?
        ↓ FALSE
Continue Processing
        ↓
Store Invoice
```

The invoice should now exist in the invoice datastore.

---

## Second Request

Send the exact same payload again.

Expected behavior:

```text
Receive Invoice Event
        ↓
Validate
        ↓
Create Finance Event
        ↓
Find Existing Invoice
        ↓
Invoice Already Exists?
        ↓ TRUE
Stop Duplicate Processing
```

## Expected Result

After both requests:

```text
Invoice rows created: 1
```

not:

```text
Invoice rows created: 2
```

The second webhook execution should not continue into scoring, storage, notifications, or finance delivery.

**Result: PASS**

---

# 8. Successful HTTP Delivery

## Purpose

Verify the complete successful processing path.

The downstream HTTP endpoint is configured to return a successful response.

## Expected Route

```text
Prepare Finance Delivery
        ↓
Deliver Finance Event
        ↓
HTTP Succeeded?
        ↓ TRUE
Prepare Delivery Status Update
        ↓
Mark Invoice Delivered
```

## Expected Invoice Update

The existing invoice row should be updated with:

```text
Delivery Status → Delivered
Delivered At     → current timestamp
```

The workflow should update the existing invoice instead of inserting another row.

## Expected Result

- HTTP request succeeds.
- Failure handling is skipped.
- Invoice is marked as delivered.
- Delivery timestamp is recorded.

**Result: PASS**

---

# 9. Retryable HTTP Failure

## Purpose

Verify that temporary downstream failures enter the retry mechanism instead of immediately becoming permanent failures.

The workflow treats the following HTTP statuses as retryable:

```text
408 Request Timeout
429 Too Many Requests
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

## Example Test

Force the downstream endpoint to return:

```text
503 Service Unavailable
```

## Expected Route

```text
Deliver Finance Event
        ↓
HTTP Succeeded?
        ↓ FALSE
Handle HTTP Failure
        ↓
Should Retry?
        ↓ TRUE
Wait Before Retry
        ↓
Retry Finance Delivery
```

## Expected Result

- HTTP failure is detected.
- Status is recognized as retryable.
- Workflow does not immediately log a final failure.
- Retry mechanism begins.

**Result: PASS**

---

# 10. Retry Counter

## Purpose

Verify that every failed retry increments the retry counter.

Expected progression:

```text
Initial delivery
        ↓ FAIL

Retry Count: 1
        ↓ FAIL

Retry Count: 2
        ↓ FAIL

Retry Count: 3
```

The counter is required so the workflow can determine when retry attempts must stop.

## Expected Result

Retry count increases after each failed retry.

**Result: PASS**

---

# 11. Retry Delay

## Purpose

Verify that retries are delayed rather than executed immediately.

Expected route:

```text
Failure
   ↓
Should Retry?
   ↓
Wait Before Retry
   ↓
Retry Finance Delivery
```

The Wait node prevents the workflow from continuously hitting an unavailable or rate-limited service without delay.

## Expected Result

A delay occurs between retry attempts.

**Result: PASS**

---

# 12. Retry Exhaustion

## Purpose

Verify that the workflow eventually stops retrying when a temporary failure does not recover.

The configured maximum retry count is:

```text
3
```

## Test

Configure the downstream endpoint to continuously return a retryable failure.

Example:

```text
503 Service Unavailable
```

## Expected Execution

```text
Initial Request
      ↓
    FAIL
      ↓
Retry 1
      ↓
    FAIL
      ↓
Retry 2
      ↓
    FAIL
      ↓
Retry 3
      ↓
    FAIL
      ↓
Retry Limit Reached
      ↓
Final Delivery Failure
      ↓
Automation Errors
```

## Expected Result

- Retry mechanism executes.
- Retry count increases.
- Workflow does not loop forever.
- Final failure path executes after the retry limit.
- Failure is written to the Automation Errors datastore.

**Result: PASS**

---

# 13. Non-Retryable HTTP Failure

## Purpose

Verify that permanent or request-related errors do not waste resources by entering the retry loop.

Examples include:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
```

## Example Test

Force the downstream endpoint to return:

```text
404 Not Found
```

## Expected Route

```text
Deliver Finance Event
        ↓
HTTP Succeeded?
        ↓ FALSE
Handle HTTP Failure
        ↓
Should Retry?
        ↓ FALSE
Final Delivery Failure
        ↓
Automation Errors
```

## Must NOT Execute

```text
Wait Before Retry
Retry Finance Delivery
Increment Retry Count
```

## Expected Result

- Error is detected.
- Error is classified as non-retryable.
- Retry loop is skipped.
- Failure is logged.

**Result: PASS**

---

# 14. Failure Logging

## Purpose

Verify that technical delivery failures remain separate from rejected business data.

A final delivery failure should create a record containing diagnostic information such as:

```text
Timestamp
Event ID
Stage
HTTP Status
Error Message
Retry Count
Invoice ID
```

## Expected Storage

```text
Bad invoice data
        ↓
Rejected Invoices


Technical delivery failure
        ↓
Automation Errors
```

These two failure categories should never be treated as the same type of problem.

## Expected Result

Final technical failures are recorded in the Automation Errors datastore.

**Result: PASS**

---

# 15. Invalid Data vs Technical Failure

## Purpose

Verify that the workflow distinguishes business validation failures from infrastructure or integration failures.

### Business Validation Failure

Example:

```text
Invalid email
Negative amount
Missing invoice ID
```

Expected destination:

```text
Rejected Invoices
```

### Technical Failure

Example:

```text
HTTP 503
Timeout
Downstream service unavailable
```

Expected destination after failure handling:

```text
Automation Errors
```

## Expected Result

The two categories follow separate paths and are stored separately.

**Result: PASS**

---

# 16. Notification Routing

## Purpose

Verify that notifications only occur for the appropriate collection priorities.

Expected behavior:

| Priority | Finance Review | Urgent Alert |
|---|---|---|
| LOW | No | No |
| MEDIUM | Yes | No |
| HIGH | No | Yes |

This prevents low-priority invoices from generating unnecessary alerts.

## Expected Result

Each priority triggers only its intended notification behavior.

**Result: PASS**

---

# 17. Delivery Payload Integrity

## Purpose

Verify that downstream delivery uses processed finance data rather than the original raw webhook payload.

The final delivery should contain normalized and calculated information such as:

```text
Event ID
Invoice ID
Customer Name
Amount Due
Priority
Score
Action
Alert Requirement
```

The downstream service should not need to repeat the workflow's normalization or scoring logic.

## Expected Result

Finance delivery receives the processed event structure.

**Result: PASS**

---

# 18. Invoice Delivery Status Update

## Purpose

Verify that successful delivery updates the correct invoice.

The workflow prepares data similar to:

```json
{
  "Invoice ID": "INV-DEMO-001",
  "Delivery Status": "Delivered",
  "Delivered At": "2026-08-15T..."
}
```

The Google Sheets update operation matches the row using:

```text
Invoice ID
```

## Expected Result

- Existing invoice row is found.
- Delivery Status becomes `Delivered`.
- Delivered At receives a timestamp.
- No duplicate invoice row is created.

**Result: PASS**

---

# 19. End-to-End Happy Path

## Purpose

Verify the entire workflow as one integrated system.

## Test

Send a valid invoice that does not already exist.

## Expected Execution

```text
Webhook
   ↓
Normalize
   ↓
Validate
   ↓
Transform
   ↓
Create Finance Event
   ↓
Duplicate Check
   ↓
Calculate Score
   ↓
Assign Priority
   ↓
Store Invoice
   ↓
Priority Routing
   ↓
Notification if required
   ↓
Prepare Finance Delivery
   ↓
HTTP Delivery
   ↓
Success Detection
   ↓
Update Delivery Status
```

## Expected Result

The invoice moves from raw external input to a validated, scored, stored, routed, delivered, and tracked business record.

**Result: PASS**

---

# 20. End-to-End Failure Path

## Purpose

Verify the complete technical failure lifecycle.

## Test

Send a valid invoice while the downstream service continuously returns a retryable failure.

## Expected Execution

```text
Valid Invoice
     ↓
Normal Processing
     ↓
Store Invoice
     ↓
Priority Routing
     ↓
Prepare Finance Delivery
     ↓
HTTP Delivery
     ↓ FAIL
Handle HTTP Failure
     ↓
Retry Classification
     ↓
Wait
     ↓
Retry
     ↓
Increment Retry Count
     ↓
Retry Limit Check
     ↓
Final Delivery Failure
     ↓
Automation Errors
```

## Expected Result

The valid business record remains stored, while the technical delivery failure is separately recorded for investigation.

**Result: PASS**

---

# Test Fixtures

Reusable test payloads are stored under:

```text
examples/
```

| Fixture | Scenario |
|---|---|
| `low-priority-invoice.json` | LOW priority processing |
| `medium-priority-invoice.json` | MEDIUM priority finance review |
| `high-priority-invoice.json` | HIGH priority urgent alert |
| `invalid-invoice.json` | Validation rejection |
| `duplicate-invoice.json` | Duplicate prevention |

The duplicate fixture should be submitted twice to verify idempotent behavior.

---

# Test Summary

The workflow was tested across the major business and technical paths:

```text
VALIDATION
✓ Valid input accepted
✓ Invalid input rejected
✓ Invalid records logged separately

DUPLICATE PROTECTION
✓ First invoice processed
✓ Repeated invoice detected
✓ Duplicate storage prevented

PRIORITY SCORING
✓ LOW routing
✓ MEDIUM routing
✓ HIGH routing
✓ Scoring boundaries verified

NOTIFICATIONS
✓ MEDIUM finance review
✓ HIGH urgent alert
✓ LOW avoids unnecessary alerts

DELIVERY
✓ Successful HTTP delivery
✓ Delivery status update

FAILURE HANDLING
✓ HTTP failure detection
✓ Retryable error classification
✓ Non-retryable error classification
✓ Retry delay
✓ Retry counter
✓ Retry limit
✓ Final failure logging

OBSERVABILITY
✓ Rejected invoices preserved
✓ Technical failures preserved
✓ Event IDs available for correlation
```

---

# Conclusion

Testing focused on proving that the workflow behaves correctly when things go wrong, not only when everything succeeds.

The workflow successfully demonstrates:

- input validation
- fail-fast processing
- duplicate prevention
- deterministic priority scoring
- conditional business routing
- notification handling
- persistent operational records
- downstream API delivery
- HTTP error classification
- controlled retries
- retry exhaustion
- final failure logging
- delivery status tracking

The result is a workflow that can process successful invoice events while also providing predictable behavior for invalid data, repeated events, and external-system failures.
