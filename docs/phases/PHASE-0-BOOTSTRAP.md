# Phase 0 — Repository Bootstrap & Architectural Baseline

**Spec refs:** §2.2, §3, §20, §21, §25 Step 1–2
**Status:** ✅ complete (2026-10-18) — one item pending a live Discord token (see Exit Gate)
**Deliverable:** The solution builds, the bot logs into Discord, and the generic
host (DI, configuration, logging, SQLite, HTTP resilience) is wired and verified.
No NFT logic yet.

> This phase is not in the spec's §22 list. It exists because the repo is
> greenfield (see [README](README.md)). If an existing FORGOTTEN bot is found,
> replace this phase with the §2.2 architectural assessment and integrate into
> that solution instead.

---

## 1. Objective

Create the skeletal, layered solution from spec §21 and prove the host comes up
cleanly. Everything after this phase *extends* this scaffold rather than
introducing new frameworks (spec §2.2: "Extend the existing application rather
than creating a competing framework.").

## 2. Scope

**In scope**

- Solution + project layout (spec §21).
- Generic Host (`IHost`) with DI, Options, structured logging.
- Discord.Net client connecting via the interaction framework.
- SQLite + EF Core `DbContext` registered (empty — no entities yet).
- `HttpClientFactory` + `Microsoft.Extensions.Http.Resilience` baseline handler.
- Strongly-typed config classes + **startup validation** (spec §20).
- Secret handling via env vars / user-secrets (no secrets committed — spec §5.2).

**Out of scope** (later phases)

- Any NFT, wallet, gallery, monitoring, or game code.
- Domain entities and migrations (Phase 1).

## 3. Prerequisites / decisions to capture

- [ ] .NET 10 LTS SDK installed and selected (spec §2.1).
- [ ] A Discord bot application + token available (document how it is supplied; **never commit it**).
- [ ] Decide package versions — stable and mutually compatible; record them (spec §2.1).
- [ ] Confirm target OS/runtime for deployment (document assumptions — spec §25 Step 2).

## 4. Tasks

### 4.1 Solution scaffolding
- [ ] Create solution `Forgotten.sln` at repo root with `src/` and `tests/` (spec §21).
- [ ] Create projects: `Forgotten.Bot`, `Forgotten.Nft.Domain`,
      `Forgotten.Nft.Application`, `Forgotten.Nft.Infrastructure`,
      `Forgotten.Nft.WalletWeb`, `Forgotten.Nft.Integrations`.
- [ ] Create test projects: `Forgotten.Nft.UnitTests`,
      `Forgotten.Nft.IntegrationTests`, `Forgotten.Nft.SecurityTests` (xUnit).
- [ ] Set project references to enforce layering (spec §3):
      Domain depends on nothing; Application → Domain; Infrastructure →
      Application + Domain; Bot → Application (+ Infrastructure at composition
      root only); Integrations → Application + Domain.
- [ ] ⚠ Add a build/architecture test (or analyzer) asserting **Domain does not
      reference Discord.Net** (spec §3: "Domain entities must not depend on Discord.Net.").

### 4.2 Host & composition root (`Forgotten.Bot`)
- [ ] Set up the Generic Host with `Microsoft.Extensions.DependencyInjection`,
      `Microsoft.Extensions.Options`, and `Microsoft.Extensions.Logging`.
- [ ] Register Discord.Net client + interaction service; connect on startup as an `IHostedService`.
- [ ] Register `IHttpClientFactory` with a named client + standard resilience
      handler (timeout, retry w/ jitter, circuit breaker) as the default for
      future provider clients (spec §5.3).
- [ ] Add a slash-command registration pipeline (empty set for now) so Phase 1+
      can plug commands in without re-plumbing.

### 4.3 Persistence baseline (`Forgotten.Nft.Infrastructure`)
- [ ] Add EF Core + SQLite provider; create an (entity-less) `ForgottenDbContext`.
- [ ] Configure SQLite: WAL mode, busy timeout, UTC timestamps convention (spec §9.2).
- [ ] ⚠ Ensure DB is **never auto-dropped/recreated**; migrations only (spec §9).
- [ ] Add connection-string config via options (path configurable; sensible default).

### 4.4 Configuration & secrets (spec §20, §5.2)
- [ ] Create strongly-typed options classes stubs: `NftEngineOptions`,
      `SolanaOptions` (bind only; fields fleshed out in later phases).
- [ ] Bind from `appsettings.json` + environment + user-secrets.
- [ ] ⚠ Implement `IValidateOptions<>` / startup validation. Invalid
      security-critical settings must **stop startup**, not warn (spec §20).
- [ ] Add `appsettings.Example.json` with no secrets; `.gitignore` real secrets.

### 4.5 Logging & diagnostics
- [ ] Structured logging configured (console + configurable sinks).
- [ ] A startup banner log line reporting: environment, DB path, Discord
      connection state, enabled feature flags.

## 5. Deliverables

- Buildable `Forgotten.sln` with the full project graph from §21.
- A running bot process that connects to Discord and logs "ready".
- Empty-but-valid EF migration pipeline (no entities yet, or a single empty
  initial migration — decide and document).
- `appsettings.Example.json` + documented environment variables.

## 6. Tests required

- [ ] Host builds and all DI registrations resolve (a DI validation test).
- [ ] Options validation **rejects** a deliberately-invalid config (fails closed).
- [ ] Architecture test: Domain has no Discord.Net reference.
- [ ] `ForgottenDbContext` can create a fresh SQLite file in a temp dir.

## 7. Risks / notes

- The spec repeatedly references an "existing bot", existing wallet/member
  records, and existing POW integration points (§2.2). None exist here. **Document
  this assumption explicitly** in `docs/ARCHITECTURE.md` so later phases don't
  look for records that were never built.
- Keep Discord token loading out of source control from the very first commit.

## 8. Exit Gate — all must be checked to start Phase 1

- [x] `dotnet build Forgotten.slnx` succeeds with no errors (0 warnings). *(Note: .NET 10 generated a `.slnx`, not `.sln`.)*
- [ ] ⏳ **Bot connects to Discord using a token supplied via secret config.** —
      *Not yet verified: no Discord bot token was available. The connection code,
      token binding (`Discord__Token`), and fail-closed validation are in place.
      The complementary behavior — **startup is refused when no token is present**
      — was verified by running the bot (throws `OptionsValidationException`).
      A human must run once with a real token to tick this box.*
- [x] DI graph validates at startup; invalid config fails closed. *(Unit tests + live run.)*
- [x] A fresh SQLite DB is created on first run via EF Core (not hand-made). *(`InitialCreate` migration + `DatabaseMigrationTests`.)*
- [x] No secrets in the repo; `appsettings.Example.json` present (+ `.gitignore`).
- [x] `docs/ARCHITECTURE.md` drafted with the greenfield assumption recorded.

### Verification summary (2026-10-18)

- `dotnet build Forgotten.slnx` → **Build succeeded, 0 warnings, 0 errors**.
- `dotnet test` → **10 passed, 0 failed** (options fail-closed ×4, DI graph ×2,
  layering ×2, DB migration ×2). Integration/Security test projects exist but are
  intentionally empty until later phases.
- `dotnet run` (no token) → startup **fails closed** with a clear
  `OptionsValidationException` for the missing Discord token.
