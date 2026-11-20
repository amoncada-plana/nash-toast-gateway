# UC-06 Player data capture on coupons

| Field | Value |
|---|---|
| Status | **Not required in Phase 1 – nice to have** |
| Phase | 2 |
| Toast API | Loyalty API (UC-01 lookup), Orders webhook / Orders API, Configuration API `/discounts` |
| Decisions | D-17 |
| Open items | O-22 |

> Do not implement in Phase 1. Phase 1 must keep it possible: UC-01 already attaches the player to the
> check, and the Orders webhook already stores orders.

## Goal
When a paper coupon or Toast discount is used, link the transaction to the player account – like the
"extra tender data" button in the previous POS – so ECL can report coupon usage per player.

## Proposed flow
1. Cashier validates the paper coupon and applies it as a Toast discount (Toast-native).
2. Cashier swipes the player card with Rewards › Look Up Customer (UC-01); the player is attached to the check.
3. When the check closes, the Orders webhook delivers `appliedLoyaltyInfo.loyaltyIdentifier` and
   `appliedDiscounts[]` (name, discount GUID, amount, promo code).
4. Gateway resolves the coupon name / code from synced `/discounts`, stores PlayerID + check + discounts,
   and produces a report.
5. Option if the swipe was missed: attach the player afterwards (PATCH `appliedLoyaltyInfo`) – only if
   Toast documents it for this integration.

## Open before design
- O-22: is a gateway report enough, or must it also be written to OASIS (e.g., player memo / tracking record)?
