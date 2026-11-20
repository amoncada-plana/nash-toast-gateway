# UC-04 Refunds and voids

| Field | Value |
|---|---|
| Status | In scope |
| Phase | 1 |
| Toast API | Tender API 0.1.0 TENDER_REVERSE; Orders webhook (`order_updated`); Orders API |
| Decisions | D-04, D-05, D-10, D-15 |
| Open items | O-01, O-04, O-09, O-13, O-19 |

## Goal
When a check or payment paid with points or comps is voided or refunded, the value goes back to the same
player's OASIS account – never as cash. Refunds Toast does not send as a reversal are caught by the safety
net and the nightly reconciliation.

## Scope boundary
- **Whole payment or whole check only.** Single-item refunds are not implemented (D-15).
- Toast sends TENDER_REVERSE only for: a voided payment, a voided order, a discount timeout.
- Only TENDER_REDEEM and TENDER_GRATUITY can be reversed.

## Actors
- **Primary:** F&B staff / manager · **Secondary:** ECL Finance (review queue)
- **Systems:** Toast POS, gateway, Worker Service, OASIS via nConnect

## Main flow – TENDER_REVERSE
1. Middleware and idempotency check (same as UC-02). Toast retries a reverse with the same GUID.
2. Find the original REDEEM / GRATUITY in the ledger via `transactionToUpdate`.
3. Unknown original → ERROR_TRANSACTION_DOES_NOT_EXIST.
4. Already reversed → return the stored reversal response.
5. Amount to return = `tenderPaymentsToRemove[].amount` (whole payment).
6. Ledger credit entry PENDING with a **new** ExternalTransactionId.
7. `POST PlayerInfo/{playerId}/Transactions?isidempotent=true` with the refund TransactionType (O-13),
   same account, ReferenceTransactionId = original OASIS TransactionId.
8. Original marked REVERSED, credit POSTED; ACCEPT.

## Refund safety net (intraday)
1. `POST /toast/webhooks/orders`: verify Toast-Signature (HMAC-SHA256, Base64, over body followed by
   timestamp, per-subscription secret); persist the event; return 2xx within 2 s; process asynchronously;
   upsert by order GUID.
2. Toast retries only twice (after 5 and 10 minutes), so the Worker polls the Orders API as backup.
3. A refund on a player tender (Payment.refundStatus PARTIAL / FULL, refund.refundAmount) with no matching
   reversal → auto-credit if within the property's safe limit, otherwise flag in the admin review queue.
4. Orders data also fills the employee on ledger entries (D-10).

## Nightly reconciliation
Runs per property after closeout + buffer (e.g., 6:00 a.m. property time), re-checks D-1 and D-2,
re-runnable from the admin page without double-posting.

| Code | Category | Action |
|---|---|---|
| M | Matched in Toast, ledger and OASIS | None |
| E1 | OASIS debited, Toast payment missing / voided | Auto-credit (highest priority) |
| E2 | Ledger PENDING, nothing in OASIS | Toast payment exists → manual review; none → close as FAILED |
| E3 | Toast refund without OASIS credit | Auto-credit |
| E4 | Amount mismatch (rounding / conversion) | Manual above threshold (e.g., > $0.01) |
| E5 | OASIS posting not in the ledger | Manual review |
| E6 | Toast payment not in the ledger | Manual review + alert |
| E7 | Gaming date ≠ business date | Report only |
| E8 | Manager allowance mismatch (UC-05, later) | Auto-correct and log |

Auto-fix rules: only return value to the player (never debit automatically); enabled per property with a
maximum amount; deterministic external ID (e.g., "R" + original) with `isidempotent=true`; every action
logged. Decisions on E2, auto-credit limits, report format and recipients: O-19.

Reconciliation inputs: `GET /orders/v2/ordersBulk?businessDate=` (5 req/s per location, paged), the ledger,
and `GET .../Transactions/ExternalTransactions/{externalTransactionId}` per posting, plus an orphan scan of
the accounts touched that day (postings by the gateway service user).

## Alternate flows
- **A1 Credit times out** → look up by ExternalTransactionId before any retry.
- **A2 REVERSE arrives before the REDEEM finished** → error; reconciliation resolves it.
- **A3 Single-item refund in Toast** (no reverse sent) → safety net / E3 classification and review.
- **A4 Refund after closeout** → caught by the D-1 / D-2 look-back.

## Field mapping
| Toast field | nConnect / OASIS | Rule |
|---|---|---|
| reverseTransactionInformation.transactionToUpdate | Ledger → original TransactionId, AccountId | Lookup |
| tenderPaymentsToRemove[].identifier, amount | Credit Amount | Whole payment |
| paymentsToRemove[] | — | Match ledger payment |
| header Toast-Transaction-GUID (reverse) | New ExternalTransactionId | ≤ 20 chars |
| — | ReferenceTransactionId | Original TransactionId |
| checkInfo.displayNumber | Comment | Audit text |
| payments[].tenderTransactionGuid (Orders API) | Ledger key | Reconciliation match key |

## Acceptance criteria
- [ ] Voided payment and voided check return the exact amount to the same OASIS account
- [ ] No refund is ever paid out as cash
- [ ] Repeated TENDER_REVERSE never double-credits
- [ ] Every credit references the original debit
- [ ] Webhook returns 2xx within 2 s and loses no event on restart
- [ ] Refunds without a reversal are auto-credited (within limit) or appear in the review queue
- [ ] Reconciliation report per property and business date shows M and E1–E8 with totals
- [ ] A reconciliation re-run never posts twice
