# Setup Guide

This guide explains how to import and configure the **Invoice Intake & Collections Automation** workflow in n8n.

The repository contains a sanitized version of the workflow. Credentials, spreadsheet IDs, email addresses, and other environment-specific values are intentionally excluded and must be configured after import.

---

## Requirements

Before importing the workflow, you will need:

- An n8n instance
- A Google account
- Google Sheets access
- Gmail access for finance notifications
- Three Google Sheets tabs or documents for operational data
- An HTTP endpoint for downstream finance-event delivery

The workflow can run on either:

- n8n Cloud
- a self-hosted n8n instance

---

## 1. Import the Workflow

The workflow file is located at:

```text
workflow/invoice-collections-automation.json
```

Open n8n and create or open a workflow.

Import the JSON file using n8n's workflow import functionality.

After importing, the workflow architecture should contain the following major stages:

```text
Webhook
→ Normalize Invoice
→ Validate Invoice
→ Transform Invoice
→ Create Finance Event
→ Duplicate Check
→ Calculate Score
→ Assign Priority
→ Store Invoice
→ Priority Routing
→ Finance Delivery
→ Success / Failure Handling
```

The imported workflow will not be fully operational until credentials and external resources are configured.

---

## 2. Configure Google Sheets Credentials

The public workflow does not contain the original Google Sheets credentials.

In n8n, open each Google Sheets node and select or create your own Google Sheets credential.

Google Sheets is used for three purposes:

```text
Invoices
→ primary invoice datastore

Rejected Invoices
→ invalid business-data log

Automation Errors
→ technical delivery-failure log
```

All Google Sheets nodes that belong to the same account can reuse the same Google Sheets credential.

---

## 3. Create the Invoices Sheet

Create a Google Sheet for valid invoice records.

Create a sheet tab named:

```text
Invoices
```

Add the following headers to Row 1:

```text
Invoice ID
Created At
Customer Name
Billing Email
Amount Due
Currency
Payment Terms Days
Issued At
Due At
Account Tier
Sales Owner
Payment Status
Event ID
Priority
Score
Delivery Status
Delivered At
```

The workflow uses this sheet for:

- duplicate detection
- invoice storage
- collection priority information
- delivery-status tracking

The `Invoice ID` column is especially important because it is used as the business identifier when checking for duplicates and updating existing records.

---

## 4. Create the Rejected Invoices Sheet

Create another sheet tab or Google Sheet for rejected invoice records.

Name the tab:

```text
Rejected Invoices
```

Recommended headers:

```text
Timestamp
Invoice ID
Customer Name
Billing Email
Reason
Source
Raw Amount
Raw Currency
```

This datastore is separate from the main invoice table.

Invalid incoming data should never be inserted into the primary `Invoices` sheet.

Instead, rejected records are written here for investigation.

---

## 5. Create the Automation Errors Sheet

Create another sheet tab or Google Sheet for technical workflow failures.

Name the tab:

```text
Automation Errors
```

Recommended headers:

```text
Timestamp
Event ID
Stage
HTTP Status
Error Message
Retry Count
Invoice ID
```

This sheet is used when downstream finance delivery ultimately fails.

It is intentionally separate from `Rejected Invoices`.

The distinction is:

```text
Rejected Invoices
→ business-data problem

Automation Errors
→ technical/integration problem
```

For example:

```text
Invalid billing email
→ Rejected Invoices

HTTP 503 after retry exhaustion
→ Automation Errors
```

---

## 6. Replace Spreadsheet Placeholders

The public workflow contains placeholder spreadsheet identifiers instead of the original private Google Sheets IDs.

Examples may include values such as:

```text
YOUR_INVOICES_SPREADSHEET_ID
YOUR_REJECTIONS_SPREADSHEET_ID
YOUR_ERROR_LOG_SPREADSHEET_ID
```

Open each Google Sheets node and select the appropriate document and sheet.

Map them according to their responsibility:

```text
Find Existing Invoice
→ Invoices

Store Invoice
→ Invoices

Mark Invoice Delivered
→ Invoices

Rejected Invoices
→ Rejected Invoices

Log Automation Error
→ Automation Errors
```

Do not publish personal spreadsheet IDs or private Google Sheets URLs in a public repository.

---

## 7. Configure Gmail Credentials

The workflow uses Gmail notifications for MEDIUM and HIGH priority invoices.

Open the notification nodes and configure your own Gmail credential.

The relevant nodes are:

```text
Finance Review Notification
Urgent Finance Alert
```

The public workflow uses a placeholder recipient such as:

```text
finance@example.com
```

Replace this with the email address that should receive finance notifications.

For testing, you can use your own email address.

Do not commit personal credentials, OAuth tokens, passwords, or API keys to the repository.

---

## 8. Configure the Downstream HTTP Endpoint

The workflow sends the processed finance event to an HTTP endpoint.

Locate:

```text
Deliver Finance Event
```

and:

```text
Retry Finance Delivery
```

Both nodes should ultimately target the same downstream service.

The repository version may use a mock or demonstration endpoint.

For a real implementation, replace the URL with your own finance API or webhook endpoint.

Example architecture:

```text
n8n
 │
 ▼
Deliver Finance Event
 │
 ▼
External Finance API
```

The delivery node sends processed workflow data rather than the original raw invoice payload.

---

## 9. Keep Retry Delivery Consistent

The initial HTTP delivery and retry HTTP delivery should send the same finance event.

Conceptually:

```text
Prepare Finance Delivery
        │
        ▼
Deliver Finance Event
        │
        ├── SUCCESS
        │
        └── FAILURE
                │
                ▼
           Retry System
                │
                ▼
        Retry Finance Delivery
```

The retry request should reuse the already prepared finance event instead of rebuilding the invoice from the failed HTTP response.

---

## 10. Configure the Webhook

The workflow begins with an n8n Webhook node.

The webhook accepts an invoice payload through an HTTP `POST` request.

During development, n8n provides a test webhook URL.

Example:

```text
https://YOUR-N8N-DOMAIN/webhook-test/invoice-created
```

When the workflow is activated, use the production webhook URL provided by n8n.

Example:

```text
https://YOUR-N8N-DOMAIN/webhook/invoice-created
```

Do not hard-code someone else's webhook URL when importing the workflow.

Use the URL generated by your own n8n instance.

---

## 11. Expected Incoming Payload

The workflow expects an invoice payload with the following general structure:

```json
{
  "invoice_number": "INV-DEMO-001",
  "customer": {
    "company_name": "Example Company",
    "contact_email": "billing@example.com"
  },
  "invoice": {
    "amount": "8750.50",
    "currency": "USD",
    "payment_terms": "net30"
  },
  "issued_date": "2026-08-15",
  "due_date": "2026-09-14",
  "account_tier": "business",
  "sales_rep": "Elena Cruz"
}
```

Additional ready-to-use payloads are available in:

```text
examples/
```

These include:

```text
low-priority-invoice.json
medium-priority-invoice.json
high-priority-invoice.json
invalid-invoice.json
duplicate-invoice.json
```

---

## 12. Test the Webhook

With the workflow listening for a test event, send a request to the n8n test webhook.

Example using JavaScript:

```javascript
fetch("https://YOUR-N8N-DOMAIN/webhook-test/invoice-created", {
    method: "POST",

    headers: {
        "Content-Type": "application/json"
    },

    body: JSON.stringify({
        invoice_number: "INV-DEMO-001",

        customer: {
            company_name: "Example Company",
            contact_email: "billing@example.com"
        },

        invoice: {
            amount: "8750.50",
            currency: "USD",
            payment_terms: "net30"
        },

        issued_date: "2026-08-15",
        due_date: "2026-09-14",
        account_tier: "business",
        sales_rep: "Elena Cruz"
    })
})
.then(response => response.text())
.then(data => console.log(data))
.catch(error => console.error(error));
```

Replace:

```text
YOUR-N8N-DOMAIN
```

with your own n8n domain.

---

## 13. Verify Normalization

After sending the request, inspect the output of:

```text
Normalize Invoice
```

The external payload should have been converted into a predictable internal working structure.

Check fields such as:

```text
invoiceNumber
company
email
amount
currency
terms
issued_date
due_date
account_tier
sales_rep
```

Make sure:

- unnecessary whitespace has been removed
- expected fields exist
- nested source fields were extracted correctly

---

## 14. Verify Validation

A valid invoice should follow:

```text
Validate Invoice
        │
       TRUE
        │
        ▼
Transform Invoice
```

An invalid invoice should follow:

```text
Validate Invoice
        │
       FALSE
        │
        ▼
Reject Invalid Invoice
        │
        ▼
Rejected Invoices
```

Invalid invoices should not continue into:

```text
Transform Invoice
Create Finance Event
Store Invoice
Priority Routing
Finance Delivery
```

---

## 15. Verify Duplicate Detection

Use:

```text
examples/duplicate-invoice.json
```

Send the payload once.

Expected behavior:

```text
Invoice ID not found
        ↓
Continue processing
        ↓
Store Invoice
```

Send the exact same payload again.

Expected behavior:

```text
Invoice ID found
        ↓
Invoice Already Exists?
        ↓
TRUE
        ↓
Stop duplicate processing
```

The second request should not create another invoice row.

---

## 16. Verify Priority Scoring

The workflow calculates collection priority using the following rules:

| Condition | Points |
|---|---:|
| Amount >= $10,000 | +3 |
| Amount >= $5,000 | +2 |
| Enterprise account | +2 |
| Net 60 payment terms | +1 |

Only one amount tier applies.

Priority thresholds:

| Score | Priority |
|---:|---|
| 0–2 | LOW |
| 3–4 | MEDIUM |
| 5+ | HIGH |

Use the payloads inside:

```text
examples/
```

to verify each route.

---

## 17. Verify LOW Priority Routing

Use:

```text
examples/low-priority-invoice.json
```

Expected route:

```text
Calculate Score
        ↓
Assign Priority
        ↓
LOW
        ↓
Low Priority Processing
        ↓
Prepare Finance Delivery
```

No finance-review or urgent-alert notification should be required.

---

## 18. Verify MEDIUM Priority Routing

Use:

```text
examples/medium-priority-invoice.json
```

Expected route:

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

Verify that the configured finance recipient receives the review notification.

---

## 19. Verify HIGH Priority Routing

Use:

```text
examples/high-priority-invoice.json
```

Expected route:

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

Verify that the urgent notification is sent.

---

## 20. Verify Successful Delivery

With a working HTTP endpoint, a successful request should follow:

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

The matching invoice row should then contain values similar to:

```text
Delivery Status → Delivered
Delivered At → current timestamp
```

---

## 21. Verify Failure Handling

To test technical failure handling, temporarily configure the downstream endpoint to return an error.

Expected flow:

```text
Deliver Finance Event
        ↓
HTTP Succeeded?
        ↓ FALSE
Handle HTTP Failure
        ↓
Should Retry?
```

The workflow should then classify the HTTP failure.

---

## 22. Verify Retryable Failures

Retryable HTTP statuses include:

```text
408
429
500
502
503
504
```

A retryable failure should enter:

```text
Should Retry?
        ↓ TRUE
Wait Before Retry
        ↓
Retry Finance Delivery
        ↓
Increment Retry Count
        ↓
Retry Limit Reached?
```

The retry loop should stop once the configured maximum retry count is reached.

---

## 23. Verify Non-Retryable Failures

Failures such as:

```text
400
401
403
404
```

should not repeatedly retry.

Expected behavior:

```text
HTTP Failure
        ↓
Should Retry?
        ↓ FALSE
Final Delivery Failure
        ↓
Log Automation Error
```

---

## 24. Verify Retry Exhaustion

To test retry exhaustion, use an endpoint that continuously returns a retryable failure such as:

```text
503 Service Unavailable
```

Expected behavior:

```text
Initial delivery fails
        ↓
Retry
        ↓
Retry
        ↓
Retry
        ↓
Retry limit reached
        ↓
Final Delivery Failure
        ↓
Automation Errors
```

Confirm that the workflow does not retry indefinitely.

---

## 25. Activate the Workflow

Once all tests pass:

1. Restore the correct downstream HTTP endpoint.
2. Confirm Google Sheets credentials.
3. Confirm Gmail credentials.
4. Confirm notification recipients.
5. Confirm spreadsheet mappings.
6. Confirm the production webhook URL.
7. Activate the workflow.

The production webhook can then receive invoice events without using n8n's manual test listener.

---

## Environment-Specific Values

The public repository intentionally does not include real environment-specific values.

You must provide your own:

```text
n8n webhook domain
Google credentials
Gmail credentials
Google spreadsheet IDs
notification recipients
downstream API endpoint
API authentication
```

This keeps the repository safe for public use.

---

## Security Notes

Never commit:

```text
API keys
OAuth tokens
Passwords
Authorization headers
Private webhook secrets
Personal email credentials
Private spreadsheet IDs
Production secrets
```

If authentication is required by an external API, configure it through n8n Credentials rather than hard-coding secrets directly into workflow expressions whenever possible.

---

## Repository Test Payloads

The repository contains example payloads under:

```text
examples/
```

Use these fixtures to test the main workflow paths:

| File | Purpose |
|---|---|
| `low-priority-invoice.json` | Tests LOW priority routing |
| `medium-priority-invoice.json` | Tests MEDIUM priority and finance review |
| `high-priority-invoice.json` | Tests HIGH priority and urgent alert |
| `invalid-invoice.json` | Tests validation rejection |
| `duplicate-invoice.json` | Tests duplicate prevention |

For the duplicate test, send the same payload twice.

---

## Troubleshooting

### Google Sheets fields are undefined

Confirm that:

- Row 1 contains the expected column headers
- the correct spreadsheet is selected
- the correct sheet tab is selected
- n8n has refreshed the sheet schema
- field names match the workflow mappings

---

### Gmail notification does not send

Confirm that:

- Gmail credentials are connected
- the recipient address has been replaced
- the workflow actually reached the MEDIUM or HIGH branch

---

### HIGH invoice routes to MEDIUM

Inspect the calculated score.

The maximum test should produce:

```text
Amount >= $10,000    +3
Enterprise           +2
Net 60               +1
------------------------
Total                  6
```

Make sure account-tier comparisons are normalized consistently.

For example:

```javascript
String(accountTier).trim().toLowerCase() === "enterprise"
```

---

### Duplicate test does not work

Confirm that the first execution successfully inserted the invoice into the `Invoices` sheet.

The duplicate check requires an existing record with the same `Invoice ID`.

---

### HTTP failure does not enter failure handling

Confirm that the HTTP Request node is configured to continue workflow execution when an HTTP error occurs.

The failure must reach the workflow's HTTP success/failure decision instead of terminating the entire execution immediately.

---

### Retry count never reaches the limit

Inspect the retry counter after each retry execution.

The counter must be incremented and passed into the next retry-limit check.

The retry loop should have a clear terminating condition.

---

## Next Steps

After setup is complete, see:

```text
docs/architecture.md
```

for a detailed explanation of the workflow architecture and design decisions.

See:

```text
docs/testing.md
```

for the complete test matrix and expected workflow behavior.
