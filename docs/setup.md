# Testing

## Valid Low Priority Invoice

Expected:
- passes validation
- stored in Invoices
- LOW route selected
- no urgent notification
- delivery attempted

Result:
PASS

## Valid Medium Priority Invoice

Expected:
- score between 3–4
- MEDIUM route
- finance review notification

Result:
PASS

## High Priority Invoice

Expected:
- score >= 5
- HIGH route
- urgent finance alert

Result:
PASS

## Invalid Email

Expected:
- validation fails
- invoice is not stored
- record written to Rejected Invoices

Result:
PASS

## Duplicate Invoice

Expected:
- first request stored
- second request detected
- duplicate is not inserted again

Result:
PASS

## Retry Exhaustion

Expected:
- retryable HTTP error
- maximum three retries
- final failure written to Automation Errors

Result:
PASS
