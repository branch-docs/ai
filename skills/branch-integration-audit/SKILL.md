---
name: branch-integration-audit
description: >
  Reviews existing Branch integration code against the standard onboarding path
  and produces a structured audit report. Use this skill whenever someone asks
  to review, audit, check, or validate a Branch integration — including phrases
  like "review my integration", "audit my Branch code", "does this follow the
  right path", "check my onboarding flow", "is my Branch setup correct", or
  "validate my payout implementation". Also trigger when reviewing pull requests
  or code changes that touch Branch API calls, webhook handlers, or onboarding
  flows. This skill catches dangerous anti-patterns that can result in KYC data
  mishandling, failed payouts, and compliance violations.
version: 2.1.1
---

# Branch Integration Audit

This skill reviews Branch integration code against the standard onboarding path
defined in the Branch Solution Overview. It produces a structured report that
flags violations, warnings, and passing checks.

## How to Use This Skill

When asked to review a Branch integration, examine all code that interacts with
Branch APIs, onboarding flows, webhook handlers, and disbursement logic. Produce
a report using the format below.

If the `branch-onboarding-expert` skill is also available, use it as the
authoritative reference for what the standard path looks like. If not, the
checklist below contains everything needed.

**This is a review, not a rewrite.** Don't change, replace, or rebuild any code
unless the user asks you to after reading the report.

---

## First — Ask About Branch Documentation

Ask whether Branch has given the user solution documentation for their
integration, such as a solution overview or implementation plan. If so, ask
them to share it. Where it differs from this checklist, it wins: a pattern it
specifies is an **agreed deviation**, not a violation. Some customers use a
non-standard path by agreement with Branch.

---

## Then — Identify the Solution

Branch integrations follow one of several standard solutions, and they don't
all work the same way. Ask the user which one they're building. Code clues can
help you suggest one, but confirm it with the user:

| Solution | Clues in the code | Checks to run |
|---|---|---|
| **Disbursements** (also with Tax Filing, Co-Branded Card, or Whitelabel) | Calls to the Branch disbursements endpoint | The full checklist below |
| **Flex EWA** | Advance, available balance, or advance eligibility calls; an embedded webview | C1 and C2 only |
| **Paycard** | `DIRECT_DEPOSIT_CREATED` handling; account and routing numbers sent to payroll | C1 and C2 only |
| **Stripe Connect** | Stripe App Install calls; Stripe `/v1/payouts` calls | C1 and C2 only |

Only Disbursements has a standard path so far. Flex EWA, Paycard, and Stripe
Connect don't have one yet; Branch sets up their flow with each customer. For
them, the rest of the checklist does not apply: they may use a different
webhook to mark a worker as ready, and the partner may not call the Branch
disbursements endpoint at all. **Do not flag a missing disbursement call or
`PAYMENT_PROFILE_ACTIVATED` handler as a violation, and don't judge their flow
against the public guides.** Run C1 and C2. If the user shared their Branch
documentation, review the code against it. Either way, say in the report's
summary that no standard checklist exists for this solution yet and that the
flow should be confirmed with the Branch implementation team.

---

## Audit Checklist

C1 and C2 apply to every solution. The rest apply to the Disbursements
solutions.

### CRITICAL — Differs from the Standard Path

These patterns indicate the integration is not following the standard path.
Unless the user's Branch documentation specifies the pattern, each must be
resolved, or confirmed with Branch, before going to production. If the
documentation specifies it, report it under Agreed Deviations instead.

**C1 — Direct wallet creation**
Look for: Calls to `POST /v1/organizations/{orgId}/employees/{employeeId}/wallets`
or any endpoint that passes SSN, DOB, or home address from the partner app to
Branch. This includes the legacy One-Click Wallet / Embedded API approach,
where the partner creates wallets and builds its own banking screens.
Why it's critical: Create Wallet is a legacy, non-standard path. In the
standard path, the partner app never handles KYC data. Branch hosts its own onboarding flow for
identity verification and payment method setup.

**C2 — KYC data collection in the partner app**
Look for: Form fields, API parameters, database columns, or variables that
collect SSN, date of birth, or home address for the purpose of passing them to
Branch.
Why it's critical: Same as C1. Even if the data is collected but not yet sent,
the architecture is wrong.

**C3 — Disbursements before webhook gate**
Look for: Code that creates disbursements without first checking that
`PAYMENT_PROFILE_ACTIVATED` was received for the worker. This includes
disbursement calls immediately after CreateEmployee, immediately after wallet
creation, or in flows that don't reference webhook state at all.
Why it's critical: Workers without an activated payment profile are not fundable.
Disbursements will fail or create reconciliation issues.

**C4 — Using V1 disbursement endpoint**
Look for: Calls to any disbursement endpoint under `/v1/` instead of
`POST /v2/organizations/{orgId}/workers/{workerId}/disbursements`.
Why it's critical: V2 is the current standard. V1 is legacy.

**C5 — Missing worker_id on onboarding URL**
Look for: Onboarding URLs constructed with only `org_id` but no `worker_id`
parameter.
Why it's critical: Without `worker_id`, Branch cannot associate the onboarding
session with the roster record created via CreateEmployee.

### WARNINGS — Should Flag for Review

These patterns may indicate problems or suboptimal implementations.

**W1 — Polling instead of webhooks**
Look for: Loops or scheduled jobs that check payment profile status via API
calls instead of handling `PAYMENT_PROFILE_ACTIVATED` and
`PAYMENT_PROFILE_DEACTIVATED` webhooks.
Why it matters: Webhooks are the standard mechanism. Polling is unreliable,
adds unnecessary API load, and may miss state transitions.

**W2 — No PAYMENT_PROFILE_DEACTIVATED handling**
Look for: Webhook handlers that process `PAYMENT_PROFILE_ACTIVATED` but not
`PAYMENT_PROFILE_DEACTIVATED`.
Why it matters: If a worker's payment method is deactivated, disbursements must
stop immediately. Missing this handler means money may be sent to unfundable
accounts.

**W3 — No disbursement failure handling**
Look for: Disbursement calls that don't inspect HTTP status codes or
`reason_code` in the response.
Why it matters: Disbursements can fail for many reasons (limits, suspended
accounts, fraud). Without failure handling, workers don't get paid and the
partner has no visibility into why.

**W4 — Missing external_id or non-unique external_id**
Look for: Disbursement calls without `external_id`, or implementations that
generate non-unique values (timestamps, sequential IDs that could collide).
Why it matters: `external_id` is the idempotency key. Without it, retry logic
breaks and duplicate disbursements become possible.

**W5 — Hardcoded environment URLs**
Look for: Branch API URLs hardcoded to sandbox or production without a
configuration mechanism to switch between them.
Why it matters: Makes it easy to accidentally hit production from a test
environment or vice versa.

**W6 — Amounts not in cents**
Look for: Disbursement amounts passed as dollar values (e.g., 12.50) instead
of cent values (e.g., 1250).
Why it matters: The API expects cents. Dollar values will create payouts 100x
smaller than intended.

**W7 — No roster maintenance**
Look for: CreateEmployee calls without corresponding Update Employee or Delete
Employee logic for when worker information changes or workers leave.
Why it matters: Stale roster data leads to failed disbursements and
reconciliation problems.

### PASSES — Confirm These Are Correct

When present and correctly implemented, call these out as passing:

- **P1**: CreateEmployee uses roster data only (no KYC fields)
- **P2**: Onboarding URL includes both `org_id` and `worker_id`
- **P3**: Webhook handler for `PAYMENT_PROFILE_ACTIVATED` gates disbursements
- **P4**: Webhook handler for `PAYMENT_PROFILE_DEACTIVATED` stops disbursements
- **P5**: V2 disbursement endpoint is used
- **P6**: `external_id` is present and unique per disbursement
- **P7**: Amounts are in cents
- **P8**: Disbursement responses are inspected for status codes and reason codes
- **P9**: Environment URLs are configurable (sandbox vs production)
- **P10**: Roster updates and deletions are handled

---

## Report Format

Structure the audit output as follows:

```
## Branch Integration Audit Report
Solution: [which solution, as confirmed by the user]
Branch documentation: [what the user shared, or "none provided"]

### CRITICAL — DIFFERS FROM THE STANDARD PATH
[List each with code reference, how it differs, and what the standard path
does instead]

### AGREED DEVIATIONS
[Patterns the user's Branch documentation specifies, with where it says so.
Omit this section if there are none.]

### WARNINGS
[List each warning with code reference and recommendation]

### PASSING CHECKS
[List each check that passed]

### SUMMARY
[Overall assessment: is this integration ready to ship? What must be resolved
or confirmed with Branch first?]
```

If no critical findings are found, say so explicitly — that's a good result.
If critical findings exist, be direct: unless Branch has agreed to them, they
must be resolved before production. Then ask the user whether they want help
making changes. Don't make them unasked.
