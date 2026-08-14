# Invoice Intake & Collections Automation

An event-driven finance operations workflow built with n8n.

The system receives invoice events through a webhook, validates
and normalizes incoming data, prevents duplicate processing,
calculates collection priority, routes invoices according to
business rules, and handles downstream delivery failures through
controlled retries and operational logging.

## Architecture

[workflow diagram/screenshot]

## Problem

Finance systems may receive malformed, duplicate, high-value,
or temporarily undeliverable invoice events.

This workflow was designed to process those events reliably
instead of treating automation as a simple A → B integration.

## Workflow

Webhook
→ Normalize
→ Validate
→ Transform
→ Create Finance Event
→ Duplicate Check
→ Calculate Score
→ Assign Priority
→ Store Invoice
→ Priority Routing
→ Delivery
→ Failure Recovery

## Priority Model

| Condition | Score |
|---|---:|
| Amount >= $10,000 | +3 |
| Amount >= $5,000 | +2 |
| Enterprise account | +2 |
| Net 60 terms | +1 |

Priority:
- LOW: 0–2
- MEDIUM: 3–4
- HIGH: 5+

## Failure Handling

Retryable delivery failures enter a bounded retry loop.

Non-retryable failures and exhausted retries are routed to
the final failure path and recorded for investigation.

## Test Scenarios

- Valid low-priority invoice
- Valid medium-priority invoice
- Valid high-priority invoice
- Invalid invoice
- Duplicate invoice
- Successful API delivery
- Retryable delivery failure
- Retry exhaustion

## Tech

- n8n
- Webhooks
- Google Sheets
- HTTP APIs
- Email notifications

## Running the Workflow

1. Download the workflow JSON.
2. Import it into n8n.
3. Configure your own Google Sheets credentials.
4. Configure notification credentials.
5. Replace the example downstream API endpoint.
6. Activate the webhook.
