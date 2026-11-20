# UC-02 Pay with points

| Field | Value |
|---|---|
| Status | In scope |
| Phase | 1 |
| Toast API | Tender API 0.1.0 (`POST /toast/tender`), alternate payment type "Player Points" |
| Decisions | D-03, D-04, D-05, D-10, D-12, D-13, D-14 |
| Open items | O-01, O-04, O-08, O-10, O-13, O-16, O-17, O-18 |

## Goal
Staff apply the player's points as the "Player Points" tender to pay all or part of a check, combined
with any other tender on the same check (comps, cash, card, other). Points are a tender, never a discount.

## Actors
- **Primary:** F&B staff · **Systems:** Toast POS, gateway, OASIS via nConnect, PostgreSQL ledger

## Preconditions
- Player identified (UC-01) or identified in TENDER_SEARCH
- "Player Points" alternate payment type configured in Toast and mapped to tender kind POINTS
- Menu data synced (Menus V2) with "Points eligible" tags / menu groups
- Points TransactionType and outlet mapping (CompLocId) configured for the property (O-13)

## Toast message sequence
| # | Toast message | Gateway action | Response |
|---|---|---|---|
| 1 | TENDER_SEARCH_CONFIG | Tender kind from alternate payment type GUID; allowed search fields | `searchTermNames` |
| 2 | TENDER_SEARCH | Resolve and validate player (UC-01); points converted to $ | Results with opaque `tenderIdentifier`, Available |
| 3 | TENDER_RETRIEVE_DISCOUNTS | No tender discounts (discounts are Toast-native) | `tenderDiscounts = []` |
| 4 | TENDER_RETRIEVE_PAYMENTS | Amount = min(requested, eligible items − other player tenders, available points in $) | `tenderPayments[]` (STORED_VALUE) or error (O-18) |
| 5 | TENDER_REDEEM | Post the points debit | ACCEPT or error (Toast removes the payment) |
| 6 | TENDER_GRATUITY | Rejected unless the property allows tips on points (D-12) | Error with reason, or gratuity response |
| 7 | TENDER_REVERSE | See UC-04 | — |

## Main flow – TENDER_REDEEM
1. Middleware: API key, restaurant allow-list, validation; correlation ID = Toast-Transaction-GUID.
2. Dispatcher routes by Toast-Transaction-Type and tender kind (alternate payment type GUID) → points handler.
3. **Idempotency:** if Toast-Transaction-GUID + Type + tenderIdentifier is in the ledger, return the stored response.
4. Resolve player from `tenderIdentifier`.
5. **Eligible amount:** selections not voided, not deferred, not HOUSE_ACCOUNT_PAY_BALANCE, eligible for points
   (item override → item tag → menu group → property default; alcohol per jurisdiction; modifiers follow parent),
   net after discounts, tax only if "points pay tax" (O-10); minus amounts already allocated to other player tenders.
6. Validate: amount ≤ eligible amount and ≤ available points in $.
7. Convert $ → points: points = $ × PointsPerDollar with the agreed rounding (O-16).
8. Ledger entry PENDING with a short unique ExternalTransactionId (≤ 20 chars) and allocation per selection.
9. `POST PlayerInfo/{playerId}/Transactions?isidempotent=true` on the Point account;
   Comment = "Toast check {displayNumber} – {outlet}"; audit user = gateway service account.
10. 201 (new) or 200 (duplicate) → ledger POSTED with OASIS TransactionId; store response; return ACCEPT.

## Alternate flows
- **A1 Insufficient points** → ERROR_INSUFFICIENT_FUNDS with available amount in the message.
- **A2 Amount above eligible items** → reject with the eligible amount; staff re-enter or use another tender.
- **A3 Nothing eligible on the check** → reject: "No items on this check can be paid with points".
- **A4 nConnect timeout** → look up `.../ExternalTransactions/{externalTransactionId}`; found = POSTED;
  not found = retry once with the same ID only if the time budget allows (O-08), else FAILED + error.
- **A5 Toast resend (same GUID)** → stored response; no new posting.
- **A6 Account frozen at payment time** → reject with reason.
- **A7 Tip on points** → rejected by default; allowed when the property flag is on (D-12).
- **A8 Orders API / Online Ordering check or contactless payment** → Tender API not available there.

## Field mapping
| Toast field | nConnect / OASIS | Rule |
|---|---|---|
| header Toast-Alternate-Payment-Type-GUID | Point account | Tender kinds table |
| header Toast-Transaction-GUID | Ledger key → ExternalTransactionId | ≤ 20 chars, unique |
| tenderIdentifier | PlayerID (from opaque token) | — |
| revenueCenter.guid | CompLocId / location id | Outlet mapping |
| check.selections[] (guid, item, price, tax, voided, deferred, selectionType, modifiers) | — | Eligibility and allocation |
| check.payments[] (otherPayment.guid, amount) | — | Subtract other player tenders |
| TenderPayment.amount ($) | Amount in points | × PointsPerDollar, sign per O-13 |
| check.displayNumber + outlet | Comment | Audit text |
| Toast business date | GamingDate | Converted with restaurant timeZone, closeoutHour, OASIS cut-off |
| — | TransactionId | Stored for refunds and reconciliation |

## Data recorded (ledger)
Toast order / check / payment / transaction GUIDs · ExternalTransactionId · OASIS TransactionId ·
PlayerID · account · points and $ amount · allocation per selection · status · timestamps ·
employee (filled later from the Orders API, D-10).

## Acceptance criteria
- [ ] Whole and partial check payment with points works
- [ ] Split payment with comps, cash and card on the same check works
- [ ] Only points-eligible items can be paid; alcohol follows the property's jurisdiction rule
- [ ] Toast resend never double-posts and returns an identical response
- [ ] nConnect timeout never causes a duplicate debit
- [ ] Every Toast payment is linked to one OASIS TransactionId in the ledger
- [ ] Tip on points rejected by default; accepted when the property flag is on
- [ ] Average ≤ 500 ms, max < 5 s under load test
