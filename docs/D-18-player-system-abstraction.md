# D-18 – Player-system abstraction (Ports & Adapters)

> Knowledge-base entry for decision **D-18** in [scope.md](scope.md).
> Status: **Accepted** (Nov 2026). Related open item: **O-21** (which other systems are expected).

## 1. Why
The gateway connects Toast POS to Aristocrat OASIS through the nConnect API. ECL may later connect Toast to
other player or casino-management systems, for example a different system at the Kentucky property or a new
vendor. The gateway must be able to add or swap a player system:

- without changing the Toast handlers (UC-01 to UC-04) or the rules engine,
- without breaking the latency targets (500 ms average, 5 s maximum),
- without weakening the zero-duplicate-postings guarantee.

## 2. Decision in one paragraph
The Application layer defines **ports**: system-neutral interfaces named in gateway domain terms. Each
external system has its own **adapter**. The adapter implements the ports, acts as an **anti-corruption
layer** (vendor vocabulary never reaches the domain) and contains a **facade** that hides the vendor's
multi-call flows. Each property's configuration names its player system by a **system key**, and
.NET keyed DI resolves the matching adapter (**strategy** pattern). nConnect is the first and only adapter
for now.

```
Toast ──► Api (/toast/loyalty, /toast/tender) ──► Application handlers + rules engine
                                                        │ depends only on ports
                                                        ▼
                                          Ports (IPlayerLookup, IPlayerBalances, IValuePosting)
                                                        ▲ implemented by
                              ┌─────────────────────────┴──────────────────────────┐
                    Adapters.NConnect (key NCONNECT)                     Adapters.<Vendor> (future)
                    ACL + facade + Polly + token provider
                              │
                              ▼
                    nConnect / OASIS
```

## 3. Patterns used and their roles
| Pattern | Where | Role |
|---|---|---|
| Ports & Adapters (hexagonal) | Application ↔ adapter projects | The seam that makes systems swappable |
| Adapter + Anti-Corruption Layer | `*.Adapters.<System>` | Turns vendor DTOs, codes and errors into domain types |
| Facade | Inside each adapter | One port call can hide several vendor calls (search → profile → balances → PointsPerDollar) |
| Strategy (keyed DI) | Composition root + property resolver | Picks the adapter per property from config |
| Decorator | Around ports | Metrics, tracing and PII-safe logging that apply to every adapter |
| Interface Segregation | Port design | Capabilities are split so a system can support only some of them |

### Rejected alternatives
| Alternative | Why rejected |
|---|---|
| Facade only (e.g., `OasisFacade` used by handlers) | Simpler to call, but handlers stay bound to OASIS concepts, so swapping means rewriting them |
| Generic connector `IConnector.Execute(string op, object payload)` | Stringly typed, no compile-time checks, hides real differences between systems, hard to test |
| Runtime plugins (AssemblyLoadContext / MEF) | Versioning and security-review cost in the DMZ; every new system needs a release anyway |
| Message bus / iPaaS on the hot path | Adds latency and infrastructure; Toast tender calls are synchronous with a 5 s cap |

## 4. System key and prefix convention
Each player system has a short, upper-case **system key** that is used as a prefix everywhere the system
appears. The first key is `NCONNECT`.

| Item | Convention | Example |
|---|---|---|
| System key | Upper case, letters and digits, ≤ 12 chars | `NCONNECT` |
| Adapter project | `Plana.Toast.Gateway.Adapters.<SystemName>` | `Plana.Toast.Gateway.Adapters.NConnect` |
| Adapter classes | `<SystemName><Capability>` | `NConnectPlayerLookup`, `NConnectValuePosting` |
| Config section | `PlayerSystems:<SYSTEM_KEY>` | `PlayerSystems:NCONNECT:BaseUrl` |
| Property setting | `Properties:<id>:PlayerSystem` | `"PlayerSystem": "NCONNECT"` |
| DI key | System key | `AddKeyedScoped<IValuePosting, NConnectValuePosting>("NCONNECT")` |
| Ledger value | `ProviderKey` column holds the system key | `NCONNECT` |
| Log / metric tags | `player_system=<SYSTEM_KEY>` | `player_system=NCONNECT` |
| Test projects | Mirror the adapter project | `Plana.Toast.Gateway.Adapters.NConnect.Tests` |

The system key is **not** added to the ExternalTransactionId sent to nConnect, because that field is limited
to 20 characters (D-05) and the ledger already records which system was used (`ProviderKey`).

## 5. Port contracts
Ports live in `src/Plana.Toast.Gateway.Application/Ports/PlayerSystem/`. They use only domain types.

```csharp
public interface IPlayerLookup
{
    Task<Result<PlayerRef>> FindAsync(PlayerIdentifier identifier, PropertyContext property, CancellationToken ct);
}

public interface IPlayerBalances
{
    Task<Result<PlayerBalances>> GetAsync(PlayerRef player, PropertyContext property, CancellationToken ct);
}

public interface IValuePosting
{
    Task<Result<Posting>> DebitAsync(PostingRequest request, CancellationToken ct);
    Task<Result<Posting>> CreditAsync(CreditRequest request, CancellationToken ct);   // carries the original debit reference
    Task<Result<Posting?>> FindPostingAsync(IdempotencyKey key, PropertyContext property, CancellationToken ct);
}

public interface IPlayerSystemCapabilities
{
    bool SupportsIdempotentPosting { get; }
    bool SupportsDailyLimitPreCheck { get; }   // O-02
    bool SupportsPromoBalance { get; }         // O-11
    bool SupportsEarn { get; }                 // UC-07
}
```

### Domain types (summary)
| Type | Content |
|---|---|
| `PlayerIdentifier` | Swipe data, scanned value or keyed account number (never logged) |
| `PlayerRef` | Opaque gateway player reference + system key; the adapter resolves merged players internally |
| `PlayerBalances` | Points (`long`), comp dollars (`decimal`), points per dollar, system business day (gaming day) |
| `PostingRequest` | Idempotency key, kind (Points / Comps), amount, property, outlet, Toast check reference |
| `Posting` | Outcome `Posted` / `Duplicate` / `Rejected(PlayerSystemError)` / `Unknown` + provider transaction ID |
| `PlayerSystemError` | Domain error codes (§ 6) |

### Contract rules
1. **Idempotency is a port contract.** Callers pass a gateway `IdempotencyKey`. Each adapter maps it to its
   own mechanism. nConnect: `ExternalTransactionId` (≤ 20 chars) + `isidempotent=true`; 201 means `Posted`,
   200 means `Duplicate` (D-04, D-05).
2. **Unknown is a valid result.** After a timeout the adapter returns `Unknown`, and the caller then calls
   `FindPostingAsync`. Never blind-retry a posting.
3. **No idempotency, no tenders.** If an adapter reports `SupportsIdempotentPosting = false`, startup
   validation refuses any property that routes tenders to it.
4. **Credits always reference the original debit.** Refunds never go to cash.
5. **Money:** `decimal` dollars rounded half-even to cents; points as `long`. Points ↔ dollar conversion rules
   (O-16) stay in the rules engine, not in adapters.
6. **Deadlines:** the Application layer passes a `CancellationToken` that carries the remaining Toast time
   budget. Timeouts, retries, circuit breakers and token renewal stay inside the adapter.

## 6. Error mapping in two layers
```
vendor error ──(adapter, per system)──► PlayerSystemError ──(shared catalogue)──► Toast error code + staff message
```

| `PlayerSystemError` | nConnect example (O-04 pending) | Toast mapping |
|---|---|---|
| `PlayerNotFound` | 404 on player search | Per error-mapping catalogue |
| `PlayerBarred` | PlayerStop flag | Per catalogue |
| `PlayerInactive` / `CardInvalid` | Inactive player, lost or expired card | Per catalogue |
| `AccountFrozen` | Frozen account | Per catalogue |
| `InsufficientBalance` | Insufficient balance | Per catalogue |
| `OverDailyLimit` | "patron is over limit" (D-06) | `ERROR_UNABLE_TO_PROCESS_REASON` + staff message |
| `SystemUnavailable` | Circuit open, 5xx, connection failure | Per catalogue |
| `OutcomeUnknown` | Timeout on a posting | Triggers `FindPostingAsync` before any answer |

Exact nConnect error bodies are open item **O-04**. Adapters must not invent mappings for undocumented errors.

## 7. Composition and routing
```csharp
// Adapters.NConnect/DependencyInjection.cs
public static IServiceCollection AddNConnectPlayerSystem(this IServiceCollection services, IConfiguration config)
{
    const string Key = "NCONNECT";
    services.AddOptions<NConnectOptions>().Bind(config.GetSection($"PlayerSystems:{Key}")).ValidateOnStart();
    services.AddHttpClient<NConnectClient>().AddResilienceHandler("nconnect", /* Polly v8 pipeline */ _ => { });
    services.AddKeyedScoped<IPlayerLookup, NConnectPlayerLookup>(Key);
    services.AddKeyedScoped<IPlayerBalances, NConnectPlayerBalances>(Key);
    services.AddKeyedScoped<IValuePosting, NConnectValuePosting>(Key);
    services.AddKeyedSingleton<IPlayerSystemCapabilities, NConnectCapabilities>(Key);
    return services;
}
```

- `IPlayerSystemResolver` (Application) reads `PlayerSystem` from the property configuration and resolves
  the keyed ports. Handlers ask the resolver; they never inject an adapter directly.
- **Startup validation:** every property's `PlayerSystem` key must be registered, and its capabilities must
  cover the tender kinds enabled for that property.
- Decorators for metrics, tracing and logging wrap the resolved ports and tag them with `player_system`.

## 8. Ledger and reconciliation
**Proposed ledger change (pending approval, DB schema per CLAUDE.md "Ask before"):** use neutral columns
instead of OASIS-specific names.

| Column | Meaning |
|---|---|
| `ProviderKey` | System key, e.g. `NCONNECT` |
| `ProviderPlayerRef` | Masked / opaque player reference |
| `IdempotencyKey` | Gateway key sent to the system (nConnect: ExternalTransactionId) |
| `ProviderTransactionId` | Transaction ID returned by the system (nConnect: OASIS TransactionId) |
| `ProviderReferenceTransactionId` | Original debit for credits |

The unique index on Toast-Transaction-GUID + Type + tenderIdentifier does not change. The nightly
reconciliation worker uses the same ports (`FindPostingAsync`, plus a future "list postings by business day"
capability) and never calls nConnect directly.

## 9. Solution layout
```
src/
  Plana.Toast.Gateway.Application/Ports/PlayerSystem/   ports, domain types, PlayerSystemError, resolver
  Plana.Toast.Gateway.Adapters.NConnect/                adapter, facade, ACL mappers, token provider, Polly
  Plana.Toast.Gateway.Adapters.<Vendor>/                future systems
tests/
  Plana.Toast.Gateway.Application.Tests/PlayerSystemContractTests.cs   abstract suite every adapter must pass
  Plana.Toast.Gateway.Adapters.NConnect.Tests/                          runs the suite against WireMock
```
This replaces the `src/*.NConnect` project named in CLAUDE.md. Update CLAUDE.md when the solution is
scaffolded.

**Dependency rule:** Application must not reference any `Adapters.*` project. Only Api and Workers (the
composition roots) reference adapters. An architecture test (e.g., NetArchTest) is optional and needs NuGet
approval.

## 10. Adding a new player system (checklist)
1. Get the vendor API docs and test environment; record any new open items in `scope.md`.
2. Choose a system key and follow the prefix convention (§ 4).
3. Create `Adapters.<SystemName>` with the ports, ACL mappers, facade, resilience and token handling.
4. Map every vendor error to `PlayerSystemError` (§ 6) using documented error bodies only.
5. Declare its capabilities honestly; emulate idempotency or declare it unsupported.
6. Build a simulator (WireMock) and run `PlayerSystemContractTests` against it.
7. Register it with `Add<SystemName>PlayerSystem(...)` in Api and Workers.
8. Set `PlayerSystem` in the property configuration; startup validation must pass.
9. Run latency tests against the 500 ms / 5 s targets and the reconciliation tests.

## 11. Testing strategy
| Test | Purpose |
|---|---|
| Contract suite (`PlayerSystemContractTests`) | Same behavior from every adapter: Posted, Duplicate, timeout → Unknown → Find, each error code, credit references debit |
| In-memory fake adapter | Drives the UC-01 to UC-04 handler tests and proves handlers work without nConnect |
| Adapter tests vs WireMock | nConnect payloads, 301 merge handling, token renewal, timeouts |
| Startup validation tests | Unknown system key, or a property with tenders routed to a non-idempotent adapter, both fail fast |

## 12. Guardrails against over-engineering
- Ship one adapter (`NCONNECT`). No plugin loading, no generic connector.
- Refine the ports only when a real second system exists (O-21). Until then, the contract suite and the
  dependency rule keep the boundary.
- Optional capabilities are added only when a use case needs them.

## 13. Open points
| ID | Topic |
|---|---|
| O-21 | Which other systems ECL expects (vendor, property, timing) |
| O-02 | Daily-limit pre-check capability |
| O-04 | nConnect error bodies for the error mapping |
| O-11 | Promo balance capability |
| — | Approval of the neutral ledger columns (§ 8) |
