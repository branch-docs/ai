# Disbursements Solutions — Integration Path

This file covers the Disbursements family of Branch solutions:

- **Disbursements** — the partner pays workers through Branch Wallet or Branch
  Direct
- **Disbursements with Tax Filing** — the same path, plus W-9 collection during
  Branch onboarding (see "Tax Filing Variant" below)
- **Disbursements with Co-Branded Card** — the same API path; the card branding
  is configured with Branch during implementation, not through the API
- **Disbursements with Whitelabel** — the same API path; the Branch app is
  branded for the partner during implementation. Branch still hosts onboarding.

All of them share the steps below. These are the standard steps. If the user
has solution documentation from Branch, follow it wherever it differs, and
don't rebuild existing code unless the user asks (see "Before You Start" in
`SKILL.md`).

---

## Initial Roster Sync

Before launch, all existing workers must be registered with Branch. The partner
can either send Branch a roster file for bulk import, or batch `CreateEmployee`
calls for each existing worker record.

## Step 1 — Register the Worker (Roster Data Only)

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

## Step 2 — Redirect Worker to Branch-Hosted Onboarding

Generate the onboarding URL and present it to the worker:

```
https://onboarding.branchapp.com/?org_id={{OrgUUID}}&worker_id={{workerID}}
```

The `worker_id` parameter is critical — it allows Branch to pre-associate the
worker's record with their onboarding session. Omitting it breaks the link
between the roster record and the payment profile.

Branch walks the worker through:
- Choosing Branch Wallet (instant, free) or Branch Direct (instant to debit, fee applies)
- Identity verification (KYC)
- Account setup

**The partner application has no involvement in this step.** No API calls, no
data collection, no intermediate screens. The worker interacts directly with
Branch's hosted flow.

## Step 3 — Wait for the Webhook

Branch fires `PAYMENT_PROFILE_ACTIVATED` to the partner's registered webhook
endpoint once the worker has completed setup and their account is fundable.

**This webhook is the gate.** Do not enable payouts, do not attempt disbursements,
do not assume the worker is ready until this event arrives. There is no polling
alternative — implement webhook handling.

Upon receiving the webhook, the partner enables Branch payouts for that worker
in their own system and updates the worker's status to active.

## Step 4 — Disburse Funds (Always V2)

```
POST /v2/organizations/{orgId}/workers/{workerId}/disbursements
```

Fields:
- `amount` — in cents ($12.50 = 1250)
- `external_id` — the partner's unique identifier, used as an idempotency key
- `type` — disbursement category (see below)
- `description` — human-readable description
- `worker_group` — optional grouping (business unit, location, region)

Disbursement types vary by use case: `TIP`, `DELIVERY`, `MILEAGE`, `TRIP`,
`BONUS`, `PAYCHECK`, and others depending on the partner's business model.

**Always use the V2 disbursement endpoint.** V1 exists in the API but is not the
current standard.

Branch pays the worker instantly and later collects the funds from the
partner's funding account. The partner tracks earnings and decides amounts;
Branch does not calculate pay.

For failure handling, retries, reversals, and sandbox testing, read
`disbursement-details.md`.

## Step 5 — Handle Lifecycle Events

| Webhook | Action |
|---|---|
| `PAYMENT_PROFILE_DEACTIVATED` | Stop disbursements for that worker immediately |
| `PAYMENT_PROFILE_ACTIVATED` | Re-enable disbursements |

### Updating Payment Methods

If a worker with an active payment profile wants to change how they're paid
(for example, Wallet to Direct), the partner surfaces a "manage your Branch
account" option that sends them back to the org-specific onboarding URL. Branch
handles the change; the partner listens for the deactivation and activation
webhooks above.

## Roster Maintenance

Use the `Update Employee` and `Delete Employee` endpoints to keep roster data
accurate when worker information changes or workers leave the organization.

---

## Tax Filing Variant

For Disbursements with Tax Filing, Branch's hosted onboarding also collects the
worker's W-9 and e-delivery consent. The partner still never collects tax data.

- Branch runs TIN verification and reports the result asynchronously:
  - `TIN_VERIFICATION_MATCHED` — the TIN matched
  - `TIN_VERIFICATION_FAILED` — the TIN didn't match; Branch tells the worker
    to correct it in the Branch web app
- **The TIN result does not affect account setup.** `PAYMENT_PROFILE_ACTIVATED`
  is still the payout gate. Don't hold payouts waiting for a TIN webhook.
- Results can take minutes, or up to 24 hours in sandbox.
