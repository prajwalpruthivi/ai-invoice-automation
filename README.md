# AI Invoice Automation Workflow

An n8n-based pipeline that watches an inbox for supplier invoice PDFs, extracts structured
data with Claude, stores it in Airtable, and runs a human-in-the-loop approval process with
a full audit trail.

## Contents

```
workflows/
  1-invoice-ingestion-workflow.json      Steps 1-6: email -> PDF -> AI extraction -> Airtable
  2-approval-request-workflow.json       Step 8: scheduled - finds Draft+SendForApproval invoices, emails approvers
  3-approval-response-workflow.json      Step 9: webhook - handles Approve/Reject click, updates status + audit log
ai-extraction-prompt.md                  Full system/user prompt used for AI extraction
database-schema.md                       Airtable schema (Invoices, Audit Log, Department Approvers)
sample_invoice_INV-2026-0417.pdf         Sample test invoice (fictional vendor/data)
README.md                                This file
```

## Architecture Overview

```
 Supplier email (PDF)
        │
        ▼
 [1] Ingestion workflow (Gmail Trigger)
        │  filter: has PDF attachment
        ▼
 Extract PDF text ──▶ Claude (system+user prompt) ──▶ Clean/validate JSON
        │                                                    │
        │                                          parse error? ──▶ Slack alert + Audit Log
        ▼
 Airtable: create Invoice record (Status=Draft, SendForApproval=false)
 + attach original PDF + Audit Log entry
        │
        ▼
 Human reviewer edits fields in Airtable, selects department(s),
 flips "Send for Approval" = true
        │
        ▼
 [2] Approval-request workflow (Schedule, every 15 min)
        │  finds: SendForApproval=true AND Status=Draft
        ▼
 Look up department approver(s) ──▶ email with Approve/Reject links (HMAC-signed)
        │
        ▼
 [3] Approval-response workflow (Webhook)
        │  validates token ──▶ updates Status (Fully Approved / Rejected) + Audit Log
        ▼
 Done - status visible in Airtable, full history in Audit Log
```

## Setup Instructions

1. **n8n**: use n8n Cloud or self-hosted n8n ≥ 1.60. Import the three JSON files from
   `workflows/` via *Workflows → Import from File* (import all three separately; they are
   linked only through Airtable, not through n8n's internal workflow-calling).
2. **Airtable**: create a base called `Invoice Automation` with the three tables described in
   `database-schema.md` (`Invoices`, `Audit Log`, `Department Approvers`). Populate
   `Department Approvers` with at least one row per department you plan to test.
3. **Gmail**: connect a Gmail account (ideally a shared inbox alias, e.g.
   `invoices@yourcompany.com`) via OAuth2 credentials in n8n for both the trigger (workflow 1)
   and the approval-email sender (workflow 2).
4. **Anthropic API key**: create a header-auth credential in n8n named to match
   `ANTHROPIC_API_CREDENTIAL_ID` with header `x-api-key: <your key>`.
5. **Webhook URL**: workflow 3 must be *activated* so n8n assigns it a public webhook URL.
   Copy that URL into the `N8N_PUBLIC_WEBHOOK_URL` environment variable used by workflow 2's
   link-building Code node.
6. Replace every placeholder credential ID (`AIRTABLE_CREDENTIAL_ID`, `AIRTABLE_BASE_ID`,
   `GMAIL_CREDENTIAL_ID`, `ANTHROPIC_API_CREDENTIAL_ID`, `SLACK_CREDENTIAL_ID`) inside the
   JSON, or re-select credentials in the n8n UI after import — n8n will prompt you for each
   node that has a missing credential.
7. Send yourself the sample PDF (`sample_invoice_INV-2026-0417.pdf`) as an email attachment to
   the monitored inbox to trigger an end-to-end test run.

## Required Environment Variables

| Variable | Used by | Purpose |
|----------|---------|---------|
| `ANTHROPIC_API_KEY` | Ingestion workflow (as n8n credential, not raw env var) | Claude API access |
| `APPROVAL_LINK_SECRET` | Workflows 2 & 3 | HMAC secret to sign/verify approve-reject links so they can't be forged |
| `N8N_PUBLIC_WEBHOOK_URL` | Workflow 2 | Base URL n8n exposes publicly, used to build the approve/reject email links |
| Airtable Personal Access Token | All 3 workflows (as n8n credential) | Read/write to the base |
| Gmail OAuth2 credentials | Workflows 1 & 2 (as n8n credential) | Read inbox / send approval emails |
| Slack Bot Token (optional) | Ingestion workflow | Failure alerts to `#invoice-processing-alerts` |

Set `APPROVAL_LINK_SECRET` and `N8N_PUBLIC_WEBHOOK_URL` in n8n's environment (Settings →
Environment Variables, or your hosting platform's env config) — never hardcode secrets in the
workflow JSON itself.

## Workflow Explanation

- **Workflow 1 (Ingestion)** polls Gmail every minute, discards non-PDF emails, captures sender
  metadata, extracts PDF text, sends it to Claude with a strict JSON-only system prompt, cleans
  and cross-validates the response (e.g. checks Net + VAT = Gross), then creates the Airtable
  record with `Application Status = Draft` and `Send for Approval = false` by default. Parsing
  failures branch to a Slack alert + Audit Log entry instead of silently failing.
- **Workflow 2 (Approval Request)** runs every 15 minutes, finds invoices a human has reviewed
  and flagged (`Send for Approval = true`, still `Draft`), looks up the approver(s) for the
  selected department(s), and emails each an HMAC-signed Approve/Reject link — no separate
  login system needed for a take-home assignment scope.
- **Workflow 3 (Approval Response)** is a webhook that verifies the HMAC token (so a leaked
  invoice ID alone can't be used to approve something), updates the invoice's status to
  `Fully Approved` or `Rejected`, and writes an Audit Log row with a timestamp.

## Assumptions

- A shared/monitored inbox already exists and can be connected to n8n via OAuth2.
- One invoice = one PDF attachment per email (the "Extract From File" step takes the first
  PDF found; multiple PDFs in one email would need a Split-in-Batches loop, noted as an
  extension point).
- Department approvers are managed in a separate `Department Approvers` Airtable table rather
  than hardcoded, so the mapping is configurable without editing the workflow.
- Approval identity verification is simplified to a signed link (sufficient to prevent link
  forgery) rather than a full SSO/login flow, which is out of scope for this assignment size.
- Currency and VAT rules are the general EU-style net/VAT/gross structure; the AI prompt is
  general enough to handle other tax models but hasn't been tuned per-country.
- "Confidence-based review routing" (bonus) is partially covered: the `confidence_score` and
  `anomalies` fields are captured and stored so a reviewer or a follow-up automation rule can
  route low-confidence invoices differently, but no automatic Slack escalation ties to a
  specific threshold in the current build — flagged here as a natural next step.

## Error Handling

- Non-PDF emails are routed to a `NoOp` node and left untouched.
- Malformed/unparseable AI responses are caught in the Code node (`try/catch` around
  `JSON.parse`), routed to a Slack alert, and logged to the Audit Log with the raw response
  for debugging — instead of crashing the workflow or silently creating a bad record.
- The approval webhook rejects any request with an invalid or missing HMAC token with an
  HTTP 403 before touching Airtable.
- Math cross-checks (Net + VAT vs. Gross) are re-verified in code even if the AI's own
  `anomalies` array misses it.

## Demo Recording Script (5–10 min) — for you to record

Since I can't record video for you, here's a tight script that hits every required beat:

1. **(0:00–0:30)** Show the three imported workflows in n8n and the Airtable base schema.
2. **(0:30–1:30)** Send the sample invoice PDF to the monitored inbox from another email
   account. Show the Gmail Trigger firing in n8n (Executions tab).
3. **(1:30–3:00)** Step through the ingestion workflow execution: PDF text extraction node
   output → Claude API call/response → cleaned JSON in the Code node → new Airtable record
   with all fields populated, PDF attached, Status=Draft.
4. **(3:00–4:00)** Open the Airtable record, manually edit one field (simulate reviewer
   correction), select a Department, flip `Send for Approval` to true.
5. **(4:00–5:30)** Trigger workflow 2 manually (or wait for the 15-min schedule), show the
   approval email arriving with Approve/Reject links, and show the Audit Log row for
   "SentForApproval".
6. **(5:30–6:30)** Click Approve, show workflow 3's webhook execution, and show the Airtable
   record flipping to `Fully Approved` with a new Audit Log entry.
7. **(6:30–7:30)** Error handling demo: send a non-PDF email (ignored) and/or a corrupted/
   scanned PDF that produces low-confidence extraction, showing the Slack alert or the
   `anomalies` field populated.
8. **(7:30–8:00)** Wrap-up: recap the architecture diagram from this README.
