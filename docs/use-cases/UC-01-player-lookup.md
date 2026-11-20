# UC-01 Player lookup, balance inquiry and validation

| Field | Value |
|---|---|
| Status | In scope – must have |
| Phase | 1 |
| Toast API | Loyalty API 1.1.0 (`POST /toast/loyalty`); Tender API 0.1.0 search (`POST /toast/tender`) |
| Decisions | D-01, D-02, D-03, D-08, D-09, D-11 |
| Open items | O-04, O-06, O-07, O-11, O-14, O-15, O-16 |

## Goal
Staff identify a casino player at the POS and see their points (with dollar equivalent) and comp dollars
without charging anything ("How much do I have?"). The same identification is reused by the tender flow
(UC-02, UC-03) so staff do not identify the player twice.

## Actors
- **Primary:** F&B staff (cashier, server) – staff-facing only; players see balances in the mobile app
- **Systems:** Toast POS, gateway, OASIS via nConnect

## Trigger
Staff tap **Rewards › Look Up Customer** and swipe the player card, or key in the account number where
the property allows it. At payment time, staff select "Player Points" / "Player Comps" and swipe (TENDER_SEARCH).

## Preconditions
- Toast-Restaurant-External-ID is on the allow-list and mapped to a property
- Valid static API key in the Authorization header
- nConnect token available (renewed in the background)
- Property policy flags loaded (keyed account number allowed or not)

## Toast messages handled
| Endpoint | Transaction type | Gateway behavior |
|---|---|---|
| `/toast/loyalty` | LOYALTY_INQUIRE | Main lookup by swipe, scan or keyed account number |
| `/toast/loyalty` | LOYALTY_SEARCH | Name / phone / email search → nConnect `$search` (Toast only searches by these) |
| `/toast/loyalty` | LOYALTY_REDEEM, LOYALTY_ACCRUE, LOYALTY_REVERSE | ACCEPT without posting (ACCRUE retries arrive with a new GUID) |
| `/toast/loyalty` | LOYALTY_SIGNUP | Declined (enrollment at the Players Club) |
| `/toast/tender` | TENDER_SEARCH_CONFIG | Return search fields (max 4): "Player Card" (+ "Account Number" where allowed) |
| `/toast/tender` | TENDER_SEARCH | Resolve player from `swipeData` / `scanData` / `searchTerms`; return results with "Available" |

## Main flow – LOYALTY_INQUIRE
1. Middleware validates API key, restaurant and payload; sets correlation ID = Toast-Transaction-GUID.
2. Route to the property by Toast-Restaurant-External-ID (unknown → ERROR_INVALID_RESTAURANT).
3. Resolve the identifier in this order:
   1. Known player token from a previous lookup → `GET PlayerInfo/{id}`
   2. Swipe data → `GET PlayerInfo?$search=CardSwipe=`
   3. Keyed number, only if the property allows it → `GET PlayerInfo?$search=IdPrintedOnCard=`
4. Read profile and accounts: `GET PlayerInfo/{id}?$select=accounts`.
5. Run player validation (below).
6. Convert points to dollars with PointsPerDollar (cached from `PropertyInfo/Setting`).
7. Return `accountInfo` (opaque identifier, preferred name), `pointsBalance`, a summary text
   (e.g., `Comp $45.00 · Points 12,500 (≈ $12.50)`), `offers = []`, `redemptionLimit.maxRedemptions = 0`.
8. Toast shows the balances and attaches the player to the check (appliedLoyaltyInfo).

## Main flow – TENDER_SEARCH
1. Same middleware, routing, resolution and validation as above.
2. Tender kind from Toast-Alternate-Payment-Type-GUID decides which balance is "Available":
   points (in $) for "Player Points", comp dollars for "Player Comps".
3. Return `searchResults[]` with opaque `tenderIdentifier` and properties: Name, Tier, Available.

## Player validation
| Check | nConnect field | Result |
|---|---|---|
| Player inactive | PlayerActive = false | Reject with reason |
| Player barred | PlayerStop exists | Reject with reason |
| Card lost | IdLost = true | Reject with reason |
| Card inactive / expired | IdActive = false / expiry | Reject with reason |
| Account frozen | AccountFrozen | Reject with reason (or hide that balance) |
| Merged player | HTTP 301 | Switch to surviving PlayerID (no blind redirect) |
| Exclusions DAP / Self / Casino | — | **Not checked** for F&B (D-09) |
| PinProtected | PinProtected | Pending O-07 (proposed: ignore and log) |

## Alternate flows
- **A1 Keyed number not allowed** for the property → reject: "Swipe the player card".
- **A2 Player not found** → ERROR_ACCOUNT_INVALID (HTTP 404 for LOYALTY_SEARCH).
- **A3 PlayerPrivacy flag set** → identify the player but suppress balances in INQUIRE and SEARCH.
- **A4 nConnect timeout / circuit open** → ERROR_UNABLE_TO_PROCESS with "Players Club system unavailable".
- **A5 Toast resend after 5 s** → stateless GETs, safe to repeat.
- **A6 POS offline** → Toast sends nothing; no gateway action.

## Field mapping
| Toast field | nConnect / OASIS | Rule |
|---|---|---|
| header Toast-Restaurant-External-ID | Property, nConnect instance | Property routing |
| header Toast-Transaction-GUID | — | Correlation ID |
| checkTransactionInformation.loyaltyIdentifier | CardSwipe / IdPrintedOnCard / PlayerID | Identifier resolution |
| searchCriteria.firstName / lastName / phone / email | `$search=FirstName=` / `LastName=` / `Phone=` / `Email=` | LOYALTY_SEARCH |
| searchTransactionInformation.swipeData / scanData / searchTerms | `$search=CardSwipe=` / `IdPrintedOnCard=` | TENDER_SEARCH |
| AccountInfo.identifier / tenderIdentifier | PlayerId (wrapped in opaque token) | Never raw IDs or card data |
| AccountInfo.firstName / lastName | PreferredName or FirstName, LastName | Display name only |
| AccountInfo.pointsBalance | Point account AccountBalance | Formatted string, e.g., "12,500" |
| Summary text | Comp AccountBalance / AccountAvailability; points ÷ PointsPerDollar | D-11 |
| properties "Available" | Comp availability or points in $ | Per tender kind |

## Business rules
- No PIN; card swipe is the control (D-08).
- Card swipe data is never stored, returned or logged.
- Lookup method and display format configurable per property.
- Promo dollars display pending O-11.

## Acceptance criteria
- [ ] Card swipe returns points, dollar equivalent and comp dollars; average ≤ 500 ms
- [ ] Keyed account number works only where the property flag allows it
- [ ] LOYALTY_SEARCH by name, phone and email returns matching players
- [ ] Each validation case returns its own staff message
- [ ] Merged player resolves to the surviving player
- [ ] PlayerPrivacy hides balances in INQUIRE and TENDER_SEARCH
- [ ] TENDER_SEARCH shows "Available" for the selected tender kind
- [ ] Loyalty REDEEM / ACCRUE / REVERSE never post to OASIS; SIGNUP is declined
- [ ] No swipe data, full names or full player IDs in logs
