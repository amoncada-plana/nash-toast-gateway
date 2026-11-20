# Project scope – Toast-OASIS Integration Gateway

> Single source of truth for scope, phases, decisions (D-xx) and open items (O-xx).
> Final scope approver: **Charline Padgett (ECL)**. Any scope change needs her approval.

## 1. Problem
ECL Entertainment is moving the food & beverage outlets at The Nash Casino (and later possibly its
Kentucky property) to Toast POS. Casino players hold balances in Aristocrat OASIS – points earned through
gameplay and comp dollars awarded by the casino – and expect to check and spend them at F&B outlets, as
they did with the previous POS (InfoGenesis/Agilysys).

Toast has no native connection to OASIS: staff cannot look up a player, see points or comp balances, or
charge a check against them, and every value spent or refunded must be posted back to OASIS accurately and
never paid out as cash. Doing this manually is slow at the counter, error-prone, hard to audit and
impossible for Finance to reconcile reliably.

## 2. Solution
The **Toast-OASIS Integration Gateway**, a .NET 10 (LTS) / ASP.NET Core service that connects Toast POS to
Aristocrat OASIS through the nConnect API, delivered as a custom Toast integration requested by ECL.

- **Toast calls the gateway** (Toast is the client): loyalty integration for lookup and balances,
  tender integration for paying with the alternate payment types "Player Points" and "Player Comps".
- The gateway validates each payment against item eligibility, player balance and the OASIS daily limit,
  posts debits and credits to OASIS without duplicates, and keeps a ledger linking every Toast
  transaction to its OASIS posting.
- A nightly reconciliation compares Toast, the ledger and OASIS for Finance.
- On-prem in the casino DMZ (Windows Server + IIS, PostgreSQL), published to Toast through the casino
  reverse proxy / WAF and firewall, reaching nConnect directly on the internal network.
- All rules configurable per property from an admin page.

**Outcome:** fast checkout for staff, players can use casino benefits at F&B outlets, and ECL gets a
fully traceable record of every point and comp dollar.

## 3. Properties
| Property | Rollout | Daily comp limit (in OASIS) | Player lookup |
|---|---|---|---|
| The Nash Casino (New Hampshire) | Phase 1 (pilot outlet first) | $500 | Card swipe |
| Kentucky property | Later phase (O-12) | $300 | Card swipe + keyed account number |

Limits and alcohol rules differ by jurisdiction, so every rule is per-property configuration.

## 4. Use-case catalogue
| UC | Name | Phase | Status | Doc |
|---|---|---|---|---|
| UC-01 | Player lookup, balance inquiry and validation | 1 | In scope – must have | `use-cases/UC-01-player-lookup.md` |
| UC-02 | Pay with points | 1 | In scope | `use-cases/UC-02-pay-with-points.md` |
| UC-03 | Pay with comp dollars | 1 | In scope | `use-cases/UC-03-pay-with-comps.md` |
| UC-04 | Refunds and voids | 1 | In scope | `use-cases/UC-04-refunds-voids.md` |
| UC-05 | Manager comps (Signature, Business) | 2 | Not required – nice to have | `use-cases/UC-05-manager-comps.md` |
| UC-06 | Player data capture on coupons | 2 | Not required – nice to have | `use-cases/UC-06-player-data-capture-on-coupons.md` |
| UC-07 | Earn points on F&B spend | 3 | On hold (marketing / accounting) | `use-cases/UC-07-earn-points-on-fnb.md` |

## 5. Phases
| Phase | Scope | Exit criteria |
|---|---|---|
| 0 – Feasibility | Toast sandbox and nConnect test access; Phase 0 confirmations with Toast, Aristocrat and ECL | End-to-end Toast sandbox → gateway → nConnect test (LOYALTY_INQUIRE + TENDER_REDEEM), < 500 ms average |
| 1 – Core player tenders | UC-01 to UC-04, per-property rules, ledger, reconciliation, admin page, observability, on-prem hosting, CI/CD; pilot outlet at The Nash | Agreed Toast test cases pass; ECL UAT sign-off (Charline Padgett) |
| 2 – Extensions | UC-05, UC-06, Kentucky property | Finance sign-off on reports; property configuration validated |
| 3 – Points earning | UC-07 (if approved) | Marketing and accounting approval |

Delivery target for Phase 0 + Phase 1: **about 10 weeks**.

## 6. Phase 1 components (cross-cutting)
| Component | Summary |
|---|---|
| Loyalty endpoint | `POST /toast/loyalty`, Toast Loyalty API 1.1.0 (UC-01) |
| Tender endpoint | `POST /toast/tender`, Toast Tender API 0.1.0 (UC-01 to UC-04) |
| Orders webhook receiver | `POST /toast/webhooks/orders`; HMAC-SHA256 Toast-Signature; 2xx < 2 s; persist then process; upsert by order GUID; Orders API polling as safety net |
| Inbound middleware | Static API key per environment (old + new accepted during rotation), restaurant allow-list, correlation ID, request validation |
| Transaction dispatcher | One handler per Toast transaction type; tender kind by alternate payment type GUID |
| Ledger (PostgreSQL / EF Core) | Toast GUIDs ↔ OASIS transaction IDs, per-item allocations, status (PENDING / POSTED / REVERSED / FAILED), audit |
| Idempotency | Toast key: Transaction-GUID + Type + tenderIdentifier; nConnect: `isidempotent=true` + ExternalTransactionId (≤ 20 chars) |
| nConnect client | OAuth2 password-grant token in memory, background renewal; player search, profile, balances, PointsPerDollar, gaming date, idempotent postings, transaction queries |
| Resilience (Polly v8) | Retries with jitter, POST retry only when idempotent, ~2 s timeout, circuit breaker, separate bulkhead for background jobs |
| Toast API client + reference-data sync | Authentication, Configuration, Menus V2, Restaurants, Labor, Orders APIs; stored in PostgreSQL; nightly or webhook refresh |
| Mapping & rules engine | Property routing, identifier resolution, points ↔ $, eligibility, allocation, business date → gaming day, display text |
| Per-property configuration | Routing, outlet → CompLocId, tender kinds, eligibility, transaction types, policy flags |
| Error-mapping catalogue | nConnect / gateway conditions → Toast error codes + staff messages |
| Nightly reconciliation | Worker Service, per property after closeout; Toast vs ledger vs OASIS; classes M, E1–E8; safe auto-credits; reports |
| Admin API and page | Role-based access for ECL users; configuration screens; reconciliation review queue; change audit |
| Observability | Serilog (files + Event Log), OpenTelemetry, IIS logs, health checks, PII redaction, latency dashboards (500 ms / 5 s), alerts |
| Hosting | Windows Server + IIS in DMZ, one server per environment (DEV, SANDBOX-QA, PROD); app pool AlwaysRunning + Preload; workers as Windows Service |
| Network | Reverse proxy / WAF, TLS 1.2+, public CA cert, inbound allow-list of Toast IPs (4 sandbox, 9 production), internal rule to nConnect and IdentityServer; no VPN |
| Secrets | DPAPI-protected on the server, readable only by app pool and Windows Service identities |
| CI/CD | GitHub Actions; self-hosted runner on DMZ jump box; WinRM over HTTPS; environment approvals; EF migration bundle; health check; folder-switch rollback |
| DR | VM backups / snapshots, rebuild procedure; PostgreSQL WAL archiving with PITR (RPO ≤ 5 min, RTO ≤ 1 h) |
| Test tooling & certification | Postman Toast simulator, WireMock nConnect simulator, contract, certification, sandbox, load and security tests |
| Operations runbook | Deployment, patching, recycling, monitoring, key rotation, nConnect outage, Toast offline mode, reconciliation exceptions |

## 7. Out of scope
- Credit cards (Toast + ECL card provider), Toast gift cards, cash handling
- Toast discounts and paper-coupon validation (Toast-native, configured by ECL)
- OASIS offers / coupons (not used by ECL; mobile-app offers already arrive as comp balance)
- LOYALTY_SIGNUP (enrollment happens at the Players Club)
- Refund or void of a **single item** paid with a player tender – whole payment or whole check only
- Exclusion handling (DAP / Self / Casino) for F&B (D-09)
- PIN entry (D-08)
- Open tabs for player benefits
- Orders API checks, Online Ordering checks and contactless payments (Tender API not usable there)
- Kiosk / guest-facing display flows (staff-facing only; O-14)
- Processing while the Toast location is offline (Toast does not send loyalty or tender calls offline)

## 8. Non-functional requirements
| NFR | Target |
|---|---|
| Latency | Average ≤ 500 ms, max 5 s (Toast). Internal targets: p50 ≤ 300 ms, p99 ≤ 2.5 s, hard cut-off 4.5 s |
| Availability | 99.9 % during outlet operating hours |
| Data integrity | Zero duplicate postings; 100 % of postings reconciled daily |
| Configurability | Eligibility, limits and lookup rules per property without code changes |
| Security | TLS 1.2+, DPAPI secrets, no PII / card data in logs, least-privilege nConnect permissions |
| Recoverability | RPO ≤ 5 min, RTO ≤ 1 h |
| Certification | Pass the applicable Toast loyalty and tender test cases |
| Webhooks | 2xx within 2 s |

## 9. Key assumptions
- Toast check is the source of truth for what was sold; OASIS is the source of truth for balances.
- ECL maintains comp / points eligibility tags in Toast menus (Charline Padgett).
- ECL, Toast and Aristocrat provide sandbox / test environments, credentials, test cards and network access on time.
- No partial approval of a requested amount is documented by Toast; amounts are capped through TENDER_RETRIEVE_PAYMENTS (to verify in Phase 0, O-18).

## 10. Decisions
| ID | Area | Decision | Basis |
|---|---|---|---|
| D-01 | Toast access | The custom integration gets access to the Toast Tender and Loyalty APIs. | Confirmed by Toast |
| D-02 | Player lookup | Card swipe (`swipeData`) supported in TENDER_SEARCH; LOYALTY_INQUIRE takes swipe, scan or keyed account number. LOYALTY_SEARCH only searches by name, email or phone. | Toast; Toast docs |
| D-03 | Player lookup | Loyalty and tender flows share the player identification; staff do not identify the player twice. | Toast |
| D-04 | OASIS posting | Debits and credits use `POST PlayerInfo/{playerId}/Transactions?isidempotent=true`. Redemption has no external ID and is not duplicate-safe. | nConnect pp. 119-120, 164-167 |
| D-05 | Idempotency | Toast duplicates: Toast-Transaction-GUID + Type + tenderIdentifier. nConnect duplicates: PlayerID + account + ExternalTransactionId (≤ 20 chars); 201 new, 200 duplicate. After a timeout, look up by ExternalTransactionId. | Toast tender spec; nConnect pp. 105, 119-120, 172 |
| D-06 | Daily limit | Configured in OASIS ($500 The Nash, $300 Kentucky); nConnect answers "patron is over limit"; gateway maps it to ERROR_UNABLE_TO_PROCESS_REASON with a staff message. | ECL / Aristocrat; Toast tender spec |
| D-07 | nConnect token | Design for ~60 s lifetime and no refresh token: keep in memory, renew in background. | nConnect p. 17 |
| D-08 | PIN | No PIN; card swipe is the control at The Nash. | ECL |
| D-09 | Exclusions | DAP / Self / Casino exclusion handling out of scope. Barred (PlayerStop), inactive players, lost or expired cards and frozen accounts still rejected. | ECL |
| D-10 | Audit user | Dedicated OASIS service account (e.g., TOAST_GATEWAY); Comment carries Toast check number and outlet; employee recorded from Orders API after close. | ECL; Toast tender spec |
| D-11 | Balance display | LOYALTY_INQUIRE: `pointsBalance` + text summary ("Comp $45.00 · Points 12,500 (≈ $12.50)"). TENDER_SEARCH: "Available" property per result. | Product; Toast tender spec |
| D-12 | Tips | Comps: gratuity prompt off on "Player Comps"; TENDER_GRATUITY rejected. Points: rejected by default, per-property flag to allow. | Product |
| D-13 | Eligibility | Menus V2 item tags ("Comp eligible", "Points eligible"), menu group fallback, no sales categories; matched against synced menu data. | Product; Toast tender spec, Menus V2 |
| D-14 | Tender model | Points and comps are Toast tenders (alternate payment types), never discounts. | Scope calls 15 Oct 2026 |
| D-15 | Refund granularity | Whole payment or whole check only; single-item refunds not implemented. | Toast Developer Support response (Oct 2026) |
| D-16 | Hosting | On-prem Windows Server + IIS in the DMZ, PostgreSQL, no cache tool, single IIS server per environment; code on GitHub; access via jump box. | Nov 2026 hosting decision |
| D-17 | Phase 1 scope | UC-05 and UC-06 not required in Phase 1 (nice to have); UC-07 on hold. | Oct 2026 scope review |
| D-18 | Architecture | Player-system abstraction: Ports & Adapters. Application defines system-neutral ports (lookup, balances, idempotent value posting, domain error codes); nConnect is the first adapter (anti-corruption layer, internal facade over multi-call flows). The player system is chosen per property by config key (keyed DI). No generic connector, runtime plugins or bus on the hot path. | Nov 2026 architecture analysis; details in `D-18-player-system-abstraction.md` |

## 11. Open items
| ID | Owner | Question / request | Why it matters | Affects |
|---|---|---|---|---|
| O-01 | Aristocrat / ECL IT | Which endpoint does the current InfoGenesis/Agilysys F&B adapter use for redemptions? If Redemption, its duplicate and reversal behavior. | Confirms D-04; comp reports | UC-02/03/04 |
| O-02 | Aristocrat | Can the remaining daily limit be read before posting (AccountLimitBalance / RunningTotal)? Endpoint? Limits by rank (RankLimits)? | Pre-check, show remaining | UC-03 |
| O-03 | Aristocrat | Real `expires_in` of the nConnect token | Renewal timing (D-07) | All |
| O-04 | Aristocrat | Sample error bodies: 404, over limit, insufficient balance, frozen account, 301 merge | Error-mapping catalogue | All |
| O-05 | Aristocrat / ECL IT | Test OASIS per property with test players and cards | Sandbox and certification | All |
| O-07 | ECL | Handling of `PinProtected` accounts (proposed: ignore and log) | Avoid unexpected rejections | UC-01 |
| O-08 | Engineering | Time budget per request vs 5 s limit with retries and several nConnect calls | Latency NFR | All |
| O-09 | Toast / ECL | Request the Orders webhook subscription from Toast support | Safety net, reconciliation | UC-04 |
| O-10 | ECL Finance | Do comps and points pay tax? | Eligible amount | UC-02/03 |
| O-11 | ECL | Show promo dollars at lookup? (15 Oct call listed points, comps and promo) | Display text | UC-01 |
| O-12 | ECL | Phase 1 properties; when does Kentucky join? | Config, testing | Scope |
| O-13 | Aristocrat | TransactionType codes and Amount sign for points debit, comp debit and refund credit; CompLocId per revenue center | Posting payloads | UC-02/03/04 |
| O-14 | ECL | Any kiosk / guest-facing display use? (assumed staff-only) | Privacy, display | UC-01 |
| O-15 | Toast / Aristocrat | What a swipe sends (full track or card number) vs nConnect CardSwipe format | Player search | UC-01 |
| O-16 | ECL / Aristocrat | Rounding rule for points ↔ dollars (PointsPerDollar) | Amounts | UC-01/02 |
| O-17 | ECL | Alcohol eligibility rules per jurisdiction (NH, KY) | Eligibility engine | UC-02/03 |
| O-18 | Toast | Confirm TENDER_RETRIEVE_PAYMENTS can return an amount lower than requested (cap by eligible / balance / limit) | Partial payment flow | UC-02/03 |
| O-19 | ECL Finance | Reconciliation: E2 handling, auto-credit limits, report format, recipients and timing | Reconciliation | UC-04 |
| O-20 | ECL IT | Monitoring platform available on-prem for OpenTelemetry | Observability | Ops |
| O-21 | ECL | Which other player / casino-management systems besides OASIS nConnect are expected (vendor, property, timing)? | Validate the player-system ports (D-18) against a real second API early | Architecture |

## 12. Stakeholders
| Role | Who |
|---|---|
| Final scope approver (ECL) | Charline Padgett |
| Client contact (ECL) | Sam McKenna |
| Points accrual decision (ECL marketing / accounting) | Stacy |
| Toast integrations | Kyle Kapanek, Charles Blase |
| Aristocrat nConnect | Tiffany Summers |
| Solutions Architect (Plana) | Alejo |
| Project Manager (Plana) | Daniel Luddy |

## 13. Sources
- Toast developer guide and OpenAPI specs (doc.toasttab.com, checked 2026-11-17)
- Aristocrat nConnect API v1.8.4021 (internal; do not commit)
- Toast POS – Aristocrat OASIS Integration Summary v0.7 (16 Oct 2026)
- Scope definition calls, 15 Oct 2026; Toast Developer Support response, 21 Oct 2026
