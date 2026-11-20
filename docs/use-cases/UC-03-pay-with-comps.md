# UC-03 Pay with comp dollars

| Field | Value |
|---|---|
| Status | In scope |
| Phase | 1 |
| Toast API | Tender API 0.1.0 (`POST /toast/tender`), alternate payment type "Player Comps" |
| Decisions | D-03, D-04, D-05, D-06, D-10, D-12, D-13, D-14 |
| Open items | O-01, O-02, O-04, O-08, O-10, O-13, O-17, O-18 |

## Goal
Staff apply the player's comp dollars as the "Player Comps" tender to pay all or part of a check,
respecting the OASIS daily comp limit. Comps never pay tips.

## Actors
- **Primary:** F&B staff · **Systems:** Toast POS, gateway, OASIS via nConnect, PostgreSQL ledger

## Preconditions
- Player identified (UC-01) or identified in TENDER_SEARCH
- "Player Comps" alternate payment type configured in Toast with the **gratuity prompt turned off**
- Menu data synced with "Comp eligible" tags / menu groups
- Daily limit configured in OASIS: $500 The Nash, $300 Kentucky
- Comp TransactionType and CompLocId per revenue center configured (O-13)

## Toast message sequence
Same as UC-02, with these differences:
| # | Toast message | Difference for comps |
|---|---|---|
| 2 | TENDER_SEARCH | "Available" = comp AccountAvailability in $ |
| 4 | TENDER_RETRIEVE_PAYMENTS | Amount = min(requested, eligible − other player tenders, comp balance, remaining daily limit if known); `tipAmount = 0` |
| 6 | TENDER_GRATUITY | **Always rejected**: ERROR_UNABLE_TO_PROCESS_REASON "Comps cannot pay tips" |

## Main flow – TENDER_REDEEM
1. Steps 1–6 of UC-02 using comp eligibility and comp balance (amounts in $, no conversion).
2. **Daily limit pre-check** if OASIS exposes it (AccountLimitBalance / RunningTotal, O-02); skipped safely otherwise.
3. Ledger PENDING with ExternalTransactionId and allocation.
4. `POST PlayerInfo/{playerId}/Transactions?isidempotent=true` on the Comp account; Comment and audit user as in UC-02.
5. 201 / 200 → ledger POSTED; store response; ACCEPT.

## Alternate flows
- **A1 Over daily limit** → nConnect answers "patron is over limit" → ERROR_UNABLE_TO_PROCESS_REASON
  with staff message (D-06), e.g., "Daily comp limit reached – remaining $12.00" when the remaining amount is known.
- **A2 Insufficient comp balance** → ERROR_INSUFFICIENT_FUNDS with the available amount.
- **A3 TENDER_GRATUITY on comps** → always rejected (D-12).
- **A4 Nothing comp-eligible** → reject with reason.
- **A5 nConnect timeout / Toast resend** → same as UC-02 A4 / A5.
- **A6 Alcohol** → apply the property's jurisdiction rule (O-17).

## Business rules
- OASIS is the enforcer of the daily limit; the gateway never treats its own value as the source of truth.
- Comps bank on the player account; expiration is managed in OASIS (no gateway logic).
- Tax paid by comps pending O-10.
- Gaming day from OASIS (EndOfDay / TimeZone), not Toast businessDate.
- Mobile-app offers already arrive as comp balance; no OASIS offers / coupons handling.

## Field mapping
As UC-02, except: Comp account instead of Point account; Amount in $ (no conversion);
"Available" = comp AccountAvailability; tip amount always 0.

## Acceptance criteria
- [ ] Whole and partial check payment with comps works
- [ ] Split with points, cash and card on one check works (e.g., $100 check: $25 comps, $10 points, $65 card)
- [ ] Over-limit response posts nothing and shows a clear staff message
- [ ] Any TENDER_GRATUITY on comps is rejected
- [ ] Only comp-eligible items can be paid
- [ ] Toast resend and nConnect timeout never cause duplicate debits
- [ ] Limit pre-check used when available, skipped safely when not
