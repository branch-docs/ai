---
name: branch-onboarding-expert
description: >
  Guides AI agents and developers through the correct Branch integration path
  for worker onboarding and payouts. Use this skill whenever anyone is building,
  planning, or discussing a Branch API integration — including worker onboarding,
  roster sync, wallet setup, payment profiles, disbursements, payout flows,
  earned wage access (EWA), paycards, or Stripe Connect payouts to Branch.
  Triggers on: Branch integration, worker onboarding, 1099, W2, gig platform,
  disburse, payout setup, Branch API, Branch Direct, Branch Wallet,
  CreateEmployee, payment profile, onboarding URL, Branch SDK, Branch sandbox.
  Also trigger when reviewing Branch MCP server tool output or planning how to
  call Branch endpoints — this skill prevents agents from using non-standard
  API paths that appear valid but violate Branch's required integration workflow.
version: 4.2.0
---

# Branch Onboarding — Standard Integration Paths

This skill defines the correct way to integrate with Branch for worker onboarding
and payouts. It exists because the Branch API exposes endpoints that *look* like
valid integration paths but are not the standard workflow. Without this guidance,
AI agents will build integrations that handle sensitive data incorrectly.

## Why This Skill Exists

The Branch OpenAPI spec (available via the ReadMe.io MCP server) lists endpoints
like `POST /employees/{id}/wallets` that accept KYC fields (SSN, DOB, address).
An agent discovering the API will naturally reach for these endpoints to build an
onboarding flow. That isn't the standard path. Branch hosts its own onboarding
experience — the partner application doesn't collect or transmit KYC data. The
`/wallets` endpoint is a legacy path. Some existing customers use it by agreement
with Branch, but it isn't used for new standard integrations.

## Core Principle

**Branch hosts the onboarding.** In the standard path, the partner system is
never responsible for collecting KYC data (SSN, DOB, address) or calling wallet
creation endpoints directly. Workers complete identity verification and payment
method setup on Branch's hosted onboarding flow.

---

## Before You Start

Do these before giving integration steps or changing any code:

1. **Ask about Branch documentation.** Ask whether Branch has given the user
   solution documentation for their integration, such as a solution overview or
   implementation plan. If so, ask them to share it, and follow it. It takes
   precedence over this skill, including where it specifies a non-standard path.
2. **Ask what they're building.** If they have no documentation, ask the user
   to describe it in their own words: who gets paid, for what, and how. Don't
   show them a menu of Branch solutions. Route them using their answer (see
   Step 0). Code clues can help, but don't decide from the code alone.
3. **Leave existing code alone unless asked.** If integration code already
   exists, don't rewrite, replace, or rebuild it unless the user asks. Review
   it, explain how it differs from the standard path, and ask whether those
   differences were agreed with Branch. A non-standard path can be valid for a
   specific customer.

---

## Step 0 — Identify the Solution

Branch integrations come in several standard solutions, and they do not all work
the same way. The webhook that means "this worker is ready" and the way money
moves both depend on the solution. **Confirm the solution with the user before
giving any integration steps.**

If the user has already named it (for example, a prompt that says "I'm building a
Branch Disbursements integration"), use that. Otherwise, route them from their
own description, not from a list:

- **If they describe paying their workers** (wages, tips, gig earnings)
  **instantly to a Branch Wallet or the worker's own debit card**, that's
  Disbursements. This includes Disbursements with Tax Filing (W-9 collection),
  a Co-Branded Card, or Whitelabel (a Branch app branded for them). Confirm it
  with the user in a sentence, then read `references/disbursements.md`.
- **Otherwise, or if you can't tell,** tell them their Branch team provides the
  integration documentation for their solution, and work from that
  documentation.

For your reference only, not to present as a menu, Branch's other solutions
include Flex EWA (workers take advances on wages they've already earned),
Paycard (regular payroll into a Branch Wallet), and Stripe Connect (a platform on
Stripe Connect pays out its workers to Branch Wallets). Code clues: Branch
disbursement calls point to Disbursements, Stripe payout or app install calls
point to Stripe Connect, and advance or eligibility calls point to EWA.

**For these and any other solution besides Disbursements:**

- Work from the documentation the user's Branch team provided. If they don't
  have it yet, tell them their Branch implementation team provides it. Keep
  this simple and confident: don't say a solution lacks a standard path or
  documentation, and don't explain why its guide isn't public.
- Don't design the integration flow yourself, and don't apply the Disbursements
  steps. Don't assume `PAYMENT_PROFILE_ACTIVATED` is the gate, and don't assume
  the partner calls the disbursement endpoint.
- Don't build the flow from the public guides, and don't ask the user to choose
  setup options (such as how earned wages are calculated) from them. Branch
  decides those with the customer.
- Once the flow is known from their Branch documentation, use the Branch MCP
  server to look up specific endpoints, fields, and webhooks.
- The Core Principle and the rules below still apply.

---

## Rules for Every Solution

These rules describe the standard path. If the user's Branch documentation
specifies otherwise, follow the documentation.

### Never Call `/wallets` Directly
`POST /v1/organizations/{orgId}/employees/{employeeId}/wallets` exists in the
API spec. It accepts SSN, DOB, and address. **Do not use it** unless the user's
Branch documentation specifies it. It belongs to a
legacy integration path, along with the One-Click Wallet / Embedded API
approach built on it (where the partner creates wallets and builds its own
banking screens). Using it means the partner application is collecting and
transmitting KYC data that should be handled exclusively by Branch's hosted
flow.

### Never Collect KYC in the Partner App
No SSN fields, no date-of-birth inputs, no address forms for Branch onboarding.
The partner app's only responsibility is roster data (name, phone, email, group).

### Never Skip the Webhook Gate
Every solution has a webhook that tells the partner the worker is ready. Don't
enable payouts or other Branch features until it arrives, and don't poll as a
substitute. Creating the roster record does not mean the worker is ready.

### Never Omit the Worker ID from the Onboarding URL
The onboarding URL must identify both the org and the worker. Without the worker
ID, Branch cannot associate the onboarding session with the roster record.

---

## Using the Branch MCP Server

The Branch MCP server (`https://docs.branchapp.com/mcp`, hosted by ReadMe.io)
exposes read-only documentation tools. None of them call the live Branch API.
When using it:

1. Call `branch-overview` first for the object model, worker lifecycle, and
   environments.
2. Use `search-endpoints` or `list-endpoints` to discover API paths.
3. Use `get-endpoint` for detailed endpoint documentation.
4. Use `search` and `fetch` to find and read guide pages.
5. Use `list-webhook-events` for the full webhook catalog and payload fields.
6. Use `get-server-variables` for the sandbox and production base URLs.

**Even when reading endpoint documentation from the MCP server, follow the
standard path for the solution**, or the user's Branch documentation where it
differs. The API spec describes what endpoints
*exist*, not which ones to *use* or in what *order*.

---

## Environment Configuration

| Setting | Sandbox | Production |
|---|---|---|
| Base URL (v1 APIs) | `https://sandbox.branchapp.com/v1/` | `https://api.branchapp.com/v1/` |
| Base URL (v2 APIs) | `https://sandbox.branchapp.com/v2/` | `https://api.branchapp.com/v2/` |
| API Keys | Sandbox keys only | Production keys only |
| KYC/Fraud | Fixed responses | Live verification |

Onboarding URLs and Org IDs are org-specific and provided during implementation.

---

## Reference Files

- `references/disbursements.md` — the full integration path for the
  Disbursements solutions, including the Tax Filing variant
- `references/disbursement-details.md` — disbursement failure handling (HTTP
  status codes, reason codes, retry logic, reversals) and sandbox testing
