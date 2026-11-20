# Toast ↔ Aristocrat OASIS Integration Gateway

.NET 10 (LTS) / ASP.NET Core gateway that lets Toast POS look up casino players and pay F&B checks with
OASIS points and comp dollars through the Aristocrat nConnect API. Delivered as a custom Toast
integration requested by ECL Entertainment (The Nash Casino first; Kentucky property later).
Built by Plana Technologies.

Hard constraints: answer Toast in 500 ms average / 5 s maximum · never pay player value out as cash ·
use only what Toast officially documents · every rule configurable per property.

## Tech stack
- .NET 10 (LTS), ASP.NET Core Minimal APIs, C# 14, System.Text.Json (source generators), FluentValidation
- PostgreSQL + EF Core (Npgsql). No external cache tool: reference data lives in PostgreSQL.
- Resilience: Microsoft.Extensions.Http.Resilience (Polly v8)
- Logging: Serilog (rolling files + Windows Event Log); tracing: OpenTelemetry
- Hosting: on-prem Windows Server + IIS (ASP.NET Core Hosting Bundle, in-process) in the casino DMZ.
  Background jobs run in a separate .NET Worker Service installed as a Windows Service.
- Environments: DEV, SANDBOX-QA, PROD (one IIS server each)
- Tests: <TBD: xUnit> · WireMock.Net as nConnect simulator · Postman collection as Toast simulator
- CI/CD: GitHub Actions; deploy through a self-hosted runner on the DMZ jump box (WinRM over HTTPS)

## Commands
- Build: `dotnet build Plana.Toast.Gateway.sln -warnaserror`
- Unit tests: `dotnet test tests/Plana.Toast.Gateway.UnitTests`
- All tests: `dotnet test`
- Run API: `dotnet run --project src/Plana.Toast.Gateway.Api`
- Run workers: `dotnet run --project src/Plana.Toast.Gateway.Workers`
- Add migration: `dotnet ef migrations add <Name> -p src/Plana.Toast.Gateway.Infrastructure -s src/Plana.Toast.Gateway.Api`
- Format: `dotnet format`

## Solution layout
- `src/*.Api` – Toast endpoints (`/toast/loyalty`, `/toast/tender`, `/toast/webhooks/orders`), admin API,
  inbound middleware, health checks. Thin: no business rules.
- `src/*.Application` – one handler per Toast transaction type; mapping & business rules engine; reconciliation logic
- `src/*.Contracts.Toast` – Toast Loyalty API 1.1.0 and Tender API 0.1.0 DTOs
- `src/*.NConnect` – typed nConnect client, token provider, resilience pipeline, 301 (merged player) handling
- `src/*.Infrastructure` – EF Core ledger, idempotency store, per-property configuration, Toast API client
- `src/*.Workers` – reference-data sync, Orders API polling, nightly reconciliation
- `tests/` – mirrors `src/` one-to-one, plus contract and simulator tests

## Architecture rules (non-negotiable)
- Use Clean Architecture pattern
- Only use Toast capabilities in Toast's official docs. If behavior is not documented, stop and ask.
- Points and comps are Toast **tenders** (alternate payment types), never discounts.
- Tender kind is identified by Toast-Alternate-Payment-Type-GUID; restaurant by Toast-Restaurant-External-ID.
- Toast requests are idempotent by Toast-Transaction-GUID + Toast-Transaction-Type + tenderIdentifier
  (unique index in the ledger); a replay returns the stored response.
- nConnect debits/credits use `PlayerInfo/{playerId}/Transactions?isidempotent=true` with a unique
  ExternalTransactionId (max 20 chars). After a timeout, look the posting up by ExternalTransactionId; never blind-retry.
- Credits always carry ReferenceTransactionId = original debit. Refunds never go to cash.
- The OASIS daily comp limit is enforced by OASIS; the gateway maps "patron is over limit" to a staff message.
- nConnect token is renewed in the background (assume ~60 s lifetime); never fetched on the hot path.
- nConnect per-call timeout ~2 s; the whole Toast request must fit in 5 s including retries.
- Toast handlers never wait on a Toast API call: menus, configuration and employees are synced to PostgreSQL.
- tenderIdentifier is an opaque token, never a raw player ID or card number.
- Webhooks: verify Toast-Signature, persist, return 2xx within 2 s, process asynchronously.
- Never hard-code property values ($500 / $300 limits, lookup methods, alcohol rules): read per-property configuration.

## Coding conventions (deviations from defaults only)
- Nullable reference types on; warnings are errors
- `async` end to end; pass `CancellationToken` through every layer
- `Result<T>` for expected outcomes (declines, insufficient balance, over limit); exceptions only for unexpected failures
- Money as `decimal`, rounded half-even to cents (as Toast does); points as `long`
- Options pattern for config; typed `HttpClient` via `IHttpClientFactory` only
- File-scoped namespaces; one public type per file

## Security & data
- Never log card swipe data, tokens, API keys, full names or full player IDs (mask or hash them)
- No secrets in code, appsettings or git; on the server they are DPAPI-protected
- Do NOT commit the nConnect API PDF (ATI internal use only, watermarked). Reference page numbers instead.
- Player data stays on property (on-prem PostgreSQL)

## Domain glossary
- **Player**: OASIS loyalty member · **PlayerID**: OASIS player key
- **Points**: gameplay balance; dollars = points ÷ PointsPerDollar (OASIS setting)
- **Comp dollars**: casino-awarded balance; daily limit enforced by OASIS
- **Tender**: payment method on a Toast check ("Player Points", "Player Comps")
- **Gaming day**: OASIS business day (EndOfDay / TimeZone); differs from Toast businessDate
- **Ledger**: gateway table linking Toast GUIDs ↔ ExternalTransactionId ↔ OASIS TransactionId
- **D-xx / O-xx**: decision / open item IDs listed in `docs/scope.md`

## Definition of done
- Builds with zero warnings; all tests pass
- Each handler covers happy path, every alternate flow in its use-case doc, nConnect timeout and idempotent replay
- Acceptance criteria checked off in the use-case doc
- New decisions and open items added to `docs/scope.md`

## Ask before
- Assuming an answer to any **open item (O-xx)** in `docs/scope.md`
- Adding a NuGet package, changing the DB schema or any Toast-facing contract
- Changing deployment scripts, IIS configuration or GitHub workflows

## Reference docs (read when relevant)
- Scope, phases, decisions (D-xx), open items (O-xx): `docs/scope.md`
- Use cases UC-01 to UC-07: `docs/use-cases/`
