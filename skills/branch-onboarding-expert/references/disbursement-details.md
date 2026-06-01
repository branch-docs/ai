# Disbursement Details — Failure Handling, Retries, Reversals, and Sandbox Testing

This reference covers the operational details of Branch disbursements that go
beyond the basic "create a disbursement" step in the standard onboarding path.

---

## Disbursement Failure Handling

### HTTP Status Codes

| Code | Condition | Action |
|---|---|---|
| 200 | Disbursement already exists, or status is CANCELED or SKIPPED | No action needed |
| 201 | New disbursement; status is SCHEDULED or COMPLETED | No action needed |
| 202 | Status is PENDING or FAILED | Check `reason_code` and take appropriate action |
| 400 | Request validation failed or no invoice configuration found | Fix the request per response details, or contact CSM if invoice config is missing |
| 429 | Rate limit exceeded or daily disbursement limit reached | Contact CSM about increasing rate limit and/or daily disbursement limit |
| 500 | Unexpected error | Contact Branch support with all available context |

### Disbursement Reason Codes

When a disbursement returns HTTP 202, inspect the `reason_code` field:

**PENDING states:**

| Reason Code | Status | Description | Action |
|---|---|---|---|
| `payout_pending` | PENDING | Disbursement still in progress | Poll every ~10 seconds by resending the create call with the same `external_id`. With `retry: true` — if PENDING or COMPLETED, acts as GET; if FAILED, retries the disbursement. With `retry: false`, acts as GET. Contact Branch support if still PENDING after ~90 seconds. |

**COMPLETED states:**

| Reason Code | Status | Description | Action |
|---|---|---|---|
| `amount_zero` | COMPLETED | Payout recorded with $0 amount | Log for potential system errors |

**SKIPPED states:**

| Reason Code | Status | Description | Action |
|---|---|---|---|
| `retry_period_elapsed` | SKIPPED | Disbursement too old to retry | Create a new payout to replace it |
| `retry_limit_exceeded` | SKIPPED | Max retry attempts reached | Create a new payout to replace it |
| `likely_match_found` | SKIPPED | Duplicate disbursement detected | Review to ensure it's not a duplicate; update metadata to be unique and resend |

**FAILED states:**

| Reason Code | Status | Description | Action |
|---|---|---|---|
| `worker_not_found` | FAILED | Worker ID not on active roster | Verify the worker ID and that a wallet or card was successfully created |
| `payment_profile_not_found` | FAILED | No destination account for worker | Pay via non-Branch mechanism and/or prompt worker to complete setup |
| `payment_profile_suspended` | FAILED | Destination account suspended | Worker must fix their account; org can then retry |
| `payment_profile_not_active` | FAILED | Destination account closed or inactive | Stop sending disbursements for this worker; direct them to their account for details |
| `payment_profile_fraudulent` | FAILED | Account permanently suspended for fraud | Account is not payable. Pay via non-Branch mechanism |
| `transfer_failed` | FAILED | Error issuing funds to worker | Retry will not work. Contact Branch support |
| `amount_exceeds_org_single_disbursement_limit` | FAILED | Exceeds org's per-disbursement limit | Contact support to adjust payout limit if amount is correct |
| `amount_exceeds_org_daily_disbursement_limit` | FAILED | Exceeds org's daily disbursement limit | Wait 24 hours or contact support to review daily limit |
| `amount_exceeds_worker_daily_disbursement_limit` | FAILED | Exceeds worker's daily disbursement limit | Wait 24 hours or contact support to review worker limit |
| `amount_does_not_cover_user_fees` | FAILED | Fee exceeds or equals disbursement amount | Accumulate additional funds before sending so payout exceeds minimum fee |
| `unexpected_error` | FAILED | Something unexpected occurred | Retry the payout; if it persists, contact Branch support |

---

## Retry Logic

The `external_id` field is the idempotency key. To retry a failed disbursement,
resend the create call with the same `external_id` and `retry: true`.

Behavior depends on the current disbursement status:
- **PENDING or COMPLETED**: The call acts as a GET (returns current state)
- **FAILED**: The disbursement is retried

With `retry: false`, the call always acts as a GET regardless of status.

---

## Disbursement Reversals

Branch supports reversing completed disbursements via the Create Disbursement
Reversal endpoint.

Key constraints:
- **Wallet only** — reversals are only supported for payouts to a Branch Wallet.
  Branch Direct payouts cannot be reversed.
- **90-day window** — disbursements can only be reversed within 90 days of payout.
- **Partial reversals** — supported, though full reversals are recommended where
  possible.
- **Insufficient funds** — reversal attempts fail if the worker's wallet balance
  cannot cover the reversal amount.

---

## Sandbox Testing

### Worker Onboarding in Sandbox

1. Add a user by calling CreateEmployee in the sandbox environment
2. Confirm the record in the Pay Admin Sandbox Portal
3. Access the onboarding link and begin sign-up using provided test credentials

Important sandbox behaviors:
- When prompted for a One Time Passcode (OTP), select Phone — test phone numbers
  have static PINs that bypass SMS verification
- Use an Incognito browser window for all sandbox testing to avoid authentication
  conflicts with Pay Admin sessions

### Wallet Onboarding (Sandbox)
- Use preconfigured values from the test account sheet provided by Branch
- Branch does not run KYC checks on PII in sandbox (name, DOB, address can be
  anything) — but SSN, Email, and Phone must match the test credentials

### Direct Onboarding (Sandbox)
- Use test debit card numbers from the list provided by Branch
- Use any future expiration date and any 3-digit CVV (except 999)
- Card type must be debit for a successful response
- CVV of 999 generates an invalid CVV response

### Disbursements in Sandbox

- Money does not actually move in sandbox
- **Wallet payments**: Any values can be used for testing
- **Direct payments**:
  - Single disbursement limit: $0.25 max
  - Daily limit: $10.00
  - Branch Direct has a minimum fee automatically applied
  - Test with disbursement amounts that are $0.25 or less after fee to avoid
    failures
  - Amounts under the minimum fee will generate errors

### Webhook Simulation

Partners can simulate the onboarding process by triggering webhook events in the
sandbox environment. See Branch's API documentation for detailed instructions on
webhook simulation.

### Payment Profile Activated

Once onboarding is complete in sandbox, the `PAYMENT_PROFILE_ACTIVATED` webhook
fires just as it would in production.
