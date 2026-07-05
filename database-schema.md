# Database Schema – Airtable

Base name: `Invoice Automation`
Table name: `Invoices`

| Field Name          | Airtable Field Type        | Notes / Options |
|---------------------|-----------------------------|------------------|
| Vendor              | Single line text            | |
| Vendor UID          | Single line text            | |
| Vendor IBAN         | Single line text            | |
| Invoice Number      | Single line text            | Used for duplicate detection (bonus) |
| Invoice Date        | Date                         | ISO format |
| Due Date            | Date                         | ISO format |
| Net Amount          | Currency / Number (2 decimals) | |
| VAT Amount          | Currency / Number (2 decimals) | |
| VAT %               | Number (percent)             | |
| Gross Amount        | Currency / Number (2 decimals) | |
| Currency            | Single select                | EUR, USD, GBP, INR, CHF, other |
| Cost Center         | Single line text            | |
| Line Items          | Long text (JSON string)     | Rendered as a sub-table in the review UI via a Notion/NocoDB alternative if preferred |
| Confidence          | Number (0–1, 2 decimals)    | From AI response |
| Anomalies           | Long text (JSON array string) | |
| Application Status  | Single select                | `Draft`, `Fully Approved`, `Rejected` |
| Selected Departments| Multiple select              | Finance, Procurement, Operations, IT, HR, Facilities (configurable) |
| Send for Approval   | Checkbox (boolean)          | |
| Sender              | Single line text            | Raw "From" header |
| Sender Email        | Email                        | |
| Sender Name         | Single line text            | |
| Email Subject       | Single line text            | |
| Email Received      | Checkbox                     | True once ingested |
| Received At         | Date + time                  | |
| PDF Attachment      | Attachment                   | Original PDF stored in Airtable |
| Approval Requested At | Date + time                | Set when Step 8 fires |
| Approved/Rejected By | Single line text            | Approver identity from the approval email/webhook |
| Approved/Rejected At | Date + time                | |
| Rejection Reason     | Long text                   | Optional |

## Second table: `Audit Log`

| Field Name   | Type            | Notes |
|--------------|-----------------|-------|
| Invoice      | Link to `Invoices` | |
| Action       | Single select   | Created, Reviewed, SentForApproval, Approved, Rejected, Error |
| Actor        | Single line text | "System (AI)", reviewer name, or approver name |
| Details      | Long text        | Free text, e.g. which fields were edited |
| Timestamp    | Date + time      | Auto-set by workflow |

## Third table (config, optional but recommended): `Department Approvers`

| Field Name  | Type            | Notes |
|-------------|-----------------|-------|
| Department  | Single select   | Must match `Selected Departments` options above |
| Approver Name | Single line text | |
| Approver Email | Email          | Used by the approval workflow to know who to notify |

This keeps the department → approver mapping data-driven instead of hardcoded in the workflow,
so it can be edited without touching the n8n JSON.
