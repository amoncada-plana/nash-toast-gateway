# UC-07 Earn points on F&B spend (accrual)

| Field | Value |
|---|---|
| Status | **On hold** – marketing / accounting decision (Stacy, ECL) |
| Phase | 3 (if approved) |
| Toast API | Loyalty API 1.1.0 LOYALTY_ACCRUE |
| Decisions | D-17 |
| Open items | — |

> Do not implement. In Phase 1 the gateway acknowledges LOYALTY_ACCRUE with ACCEPT and posts nothing.

## Goal
Players earn OASIS points on eligible F&B spend at Toast outlets.

## Known behavior to design for (when approved)
- LOYALTY_ACCRUE is sent after payment, retried with a **new GUID**, and replayed after offline mode,
  so dedupe must use check GUID + payment GUIDs, not Toast-Transaction-GUID.
- Eligible spend would follow Toast's net-sales guidance and exclude amounts paid with points or comps.
- Posting would use the same idempotent nConnect transaction pattern (D-04, D-05) with an accrual TransactionType.
- Earning rules (rate, eligible items, per property) to be defined by ECL marketing and accounting.
