---
name: branch-onboarding-expert
description: >
  Guides AI agents and developers through the correct Branch integration path
  for worker onboarding and payouts. Use this skill whenever anyone is building,
  planning, or discussing a Branch API integration — including worker onboarding,
  roster sync, wallet setup, payment profiles, disbursements, or payout flows.
  Triggers on: Branch integration, worker onboarding, 1099, W2, gig platform,
  disburse, payout setup, Branch API, Branch Direct, Branch Wallet,
  CreateEmployee, payment profile, onboarding URL, Branch SDK, Branch sandbox.
  Also trigger when reviewing Branch MCP server tool output or planning how to
  call Branch endpoints — this skill prevents agents from using non-standard
  API paths that appear valid but violate Branch's required integration workflow.
version: 3.0.0
---

# Branch Onboarding — Standard Integration Path

This skill defines the correct way to integrate with Branch for worker onboarding
and payouts. It exists because the Branch API exposes endpoints that *look* like
valid integration paths but are not the standard workflow. Without this guidance,
AI agents will build integrations that handle sensitive data incorrectly.

## Why This Skill Exists

The Branch OpenAPI spec (available via the ReadMe.io MCP server) lists endpoints
like `POST /employees/{id}/wallets` that accept KYC fields (SSN, DOB, address).
An agent discovering the API will naturally reach for these endpoints to build an
onboarding flow. This is wrong. Branch hosts its own onboarding experience — the
partner application never collects or transmits KYC data. The `/wallets` endpoint
exists for specific non-standard use cases and must not be used as the default
integration path.

## Core Principle

**Branch hosts the onboarding.** The partner system is never responsible for
collecting KYC data (SSN, DOB, address) or calling wallet creation endpoints
directly. Workers complete identity verification and payment method setup on
Branch's hosted onboarding flow.

---

## The Standard Onboarding Path

### Prerequisites

Before starting, identify the partner's persona:
- **Platform (1099/gig)**: Workers are independent contractors
- **Employer (W2)**: Workers are traditional employees

The onboarding path is the same for both. The difference affects disbursement
types and some business logic, not the technical integration.

### Initial Roster Sync

Before launch, all existing workers must be registered with Branch. The partner
can either send Branch a roster file for bulk import, or batch `CreateEmployee`
calls for each existing worker record.

### Step 1 — Register the Worker (Roster Data Only)

Call `CreateEmployee` with roster data only. No KYC fields.

```
POST /v1/organizations/{orgId}/employees/{employeeId}
```

Required fields:
- `first_name`
- `last_name`
- `phone_number`
- `email_address`
- `group_name` (location identifier)

Also include: Org ID, Worker ID.

This is an upsert — safe to call on every sync cycle. Do not include SSN, date
of birth, home address, or any other PII beyond the fields listed above.

**The worker must have completed the partner's own onboarding and been presented
with Branch as a payment option before this call is made.** The sequence is:
worker completes partner onboarding → partner presents payment options → worker
chooses Branch → partner calls CreateEmployee → partner redirects to Branch.

### Step 2 — Redirect Worker to Branch-Hosted Onboarding

Generate the onboarding URL and present it to the worker:

```
https://onboarding.branchapp.com/?org_id={{OrgUUID}}&worker_id={{workerID}}
```

The `worker_id` parameter is critical — it allows Branch to pre-associate the
worker's record with their onboarding session. Omitting it breaks the link
between the roster record and the payment profile.

Branch walks the worker through:
- Choosing Branch Wallet (instant, free) or Branch Direct (instant to debit, 2% fee)
- Identity verification (KYC)
- Account setup

**The partner application has no involvement in this step.** No API calls, no
data collection, no intermediate screens. The worker interacts directly with
Branch's hosted flow.

### Step 3 — Wait for the Webhook

Branch fires `PAYMENT_PROFILE_ACTIVATED` to the partner's registered webhook
endpoint once the worker has completed setup and their account is fundable.

**This webhook is the gate.** Do not enable payouts, do not attempt disbursements,
do not assume the worker is ready until this event arrives. There is no polling
alternative — implement webhook handling.

Upon receiving the webhook, the partner enables Branch payouts for that worker
in their own system and updates the worker's status to active.

### Step 4 — Disburse Funds (Always V2)

```
POST /v2/organizations/{orgId}/workers/{workerId}/disbursements
```

Required fields:
- `amount` — in cents ($12.50 = 1250)
- `external_id` — the partner's unique identifier, used as an idempotency key
- `type` — disbursement category (see below)
- `worker_group` — location identifier
- `description` — human-readable description

Disbursement types vary by use case: `TIP`, `DELIVERY`, `MILEAGE`, `TRIP`,
`BONUS`, `PAYCHECK`, and others depending on the partner's business model.

**Always use the V2 disbursement endpoint.** V1 exists in the API but is not the
current standard.

For retry behavior and failure handling, read `references/disbursement-details.md`.

### Step 5 — Handle Lifecycle Events

| Webhook | Action |
|---|---|
| `PAYMENT_PROFILE_DEACTIVATED` | Stop disbursements for that worker immediately |
| `PAYMENT_PROFILE_ACTIVATED` | Re-enable disbursements |

The deactivation/reactivation cycle also applies when workers change payment
methods. If a worker wants to switch from Wallet to Direct (or vice versa),
redirect them back to the org-specific onboarding URL. Branch handles the
transition; the partner listens for the webhook events.

### Roster Maintenance

Use `Update Employee` and `Delete Employee` endpoints to keep roster data
accurate when worker information changes or workers leave the organization.

---

## What NOT to Do

These are the most common mistakes agents and developers make when building
Branch integrations without this guidance:

### Never Call `/wallets` Directly
`POST /v1/organizations/{orgId}/employees/{employeeId}/wallets` exists in the
API spec. It accepts SSN, DOB, and address. **Do not use it.** This endpoint is
not part of the standard onboarding path. Using it means the partner application
is collecting and transmitting KYC data that should be handled exclusively by
Branch's hosted flow.

### Never Collect KYC in the Partner App
No SSN fields, no date-of-birth inputs, no address forms for Branch onboarding.
The partner app's only responsibility is roster data (name, phone, email, group).

### Never Skip the Webhook Gate
Do not attempt disbursements before receiving `PAYMENT_PROFILE_ACTIVATED`. Do
not poll the payment profile endpoint as a substitute. Do not assume that
completing the CreateEmployee call means the worker is ready for payouts.

### Never Use V1 Disbursements
The V1 disbursement endpoint may be visible in the API. Always use V2:
`POST /v2/organizations/{orgId}/workers/{workerId}/disbursements`

### Never Omit worker_id from the Onboarding URL
The URL must include both `org_id` and `worker_id`. Without `worker_id`, Branch
cannot associate the onboarding session with the roster record.

---

## Using the Branch MCP Server

The Branch MCP server (provided by ReadMe.io) exposes read-only documentation
tools. When using it:

1. Call `get-server-variables` first — this is required before calling
   `get-endpoint`, or `execute-request`.
2. Use `search-endpoints` or `list-endpoints` to discover API paths.
3. Use `get-endpoint` for detailed endpoint documentation.
4. Use `search` and `fetch` to find and read guide pages.
5. `execute-request` is the only tool that hits the live API and requires valid
   Branch API credentials. All other tools are read-only documentation lookups.

**Even when reading endpoint documentation from the MCP server, follow the
standard path defined in this skill.** The API spec describes what endpoints
*exist*, not which ones to *use* or in what *order*.

---

## Environment Configuration

| Setting | Sandbox | Production |
|---|---|---|
| Base URL (employee APIs) | `https://sandbox.branchapp.com/v1/` | `https://api.branchapp.com/v1/` |
| Base URL (disbursement APIs) | `https://sandbox.branchapp.com/v2/` | `https://api.branchapp.com/v2/` |
| API Keys | Sandbox keys only | Production keys only |
| KYC/Fraud | Fixed responses | Live verification |

Onboarding URLs and Org IDs are org-specific and provided during implementation.

---

## Reference Files

For detailed disbursement failure handling (HTTP status codes, reason codes,
retry logic, reversals) and sandbox testing procedures, read:
`references/disbursement-details.md`
