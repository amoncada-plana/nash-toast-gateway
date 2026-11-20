# UC-05 Manager comps (Signature and Business)

| Field | Value |
|---|---|
| Status | **Not required in Phase 1 – nice to have** |
| Phase | 2 |
| Toast API | Tender API 0.1.0, alternate payment types "Manager Comp – Signature" / "Manager Comp – Business" (proposed) |
| Decisions | D-14, D-17 |
| Open items | O-21 |

> Do not implement in Phase 1. Phase 1 must keep it possible: the Worker Service can host a daily reset
> job later, and the tender-kind table accepts new kinds without code changes in the dispatcher.

## Goal
A manager covers all or part of a guest's purchase from their own discretionary daily allowance, even when
the guest has no comps or points. Two allowance types: Signature and Business.

## Key facts
- Allowances are **not in OASIS**: ECL keeps a separate list of managers, comp type and daily limit.
- Allowances reset every day; Toast cannot replenish balances automatically, so the gateway owns the reset.
- Toast Developer Support: the docs describe **no manager-fund payment type** and no workflow that passes a
  manager identifier to an integration. Tender messages carry no employee field, so the manager must
  identify themselves in TENDER_SEARCH (e.g., Employee ID).

## Proposed flow
1. ECL loads managers per property in the admin page (employee ID, name, comp type, daily limit, active).
2. Manager selects "Manager Comp – Signature" or "– Business"; TENDER_SEARCH_CONFIG returns "Employee ID".
3. TENDER_SEARCH: match the entered ID to `externalEmployeeId` in cached Labor API employees
   (not deleted, holding a manager job) and to the ECL list; return remaining allowance.
4. TENDER_RETRIEVE_PAYMENTS: amount = min(eligible items, requested, remaining allowance).
5. TENDER_REDEEM: reduce allowance, store usage (check, items, amount, manager, time). No OASIS call.
6. TENDER_REVERSE: restore allowance.
7. Daily job at the restaurant closeoutHour resets every allowance to its limit.
8. Reconciliation category E8 checks allowance vs Toast manager-comp payments.

## Data
| Field | Source |
|---|---|
| PropertyCode, EmployeeId, Name, CompType, DailyLimit, Active | ECL admin input |
| Employee GUID, externalEmployeeId, jobReferences, deleted | Labor API (synced) |
| RemainingToday, BusinessDate | Gateway |
| Usage: CheckGuid, DisplayNumber, SelectionGuids, Amount, Transaction GUID, RevenueCenter, Time | Toast tender messages |

## Open before design
- O-21: manager identification at the POS (employee ID, badge swipe, approving Toast user), reset time, Finance reports
- Alternative to evaluate with Toast: a Toast-native mechanism (e.g., discount with approval) if it can enforce per-manager daily limits
