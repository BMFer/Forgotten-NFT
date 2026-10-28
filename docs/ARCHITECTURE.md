# FORGOTTEN NFT Engine — Architecture

> Living document. Phase 0 establishes the baseline below; later phases extend it.
> See [specs.md](specs.md) for the full requirement set and
> [phases/](phases/README.md) for the phased build plan.

## Greenfield assumption (recorded per Phase 0 Exit Gate)

The specification is written as if extending an **existing** FORGOTTEN Discord
bot — it references existing member/wallet records and existing POW integration
points (spec §2.2). **No such code existed in this repository** when the build
started (only `docs/` was present). Per spec §2.2 ("If no existing implementation
exists, create the architecture specified below") this is being built greenfield.

Consequence for later phases: do not look for pre-existing wallet/member tables
or POW hooks — they are created by this project (wallets in Phase 3, POW adapter
seam in Phase 6). If a pre-existing FORGOTTEN bot is later located, perform the
§2.2 architectural assessment and integrate rather than running Phase 0 again.

## Solution layout (spec §21)

```
Forgotten.slnx
├── src/
│   ├── Forgotten.Bot                 # Discord presentation + host (composition root)
│   ├── Forgotten.Nft.Domain          # Entities, enums, value objects, events (no external deps)
│   ├── Forgotten.Nft.Application      # Services, interfaces, DTOs, options + validation
│   ├── Forgotten.Nft.Infrastructure   # EF Core/SQLite, blockchain, metadata, caching, workers
│   ├── Forgotten.Nft.WalletWeb        # HTTPS wallet-verification site (fleshed out in Phase 3)
│   └── Forgotten.Nft.Integrations     # Discord roles, POW adapter (Phases 5–6)
└── tests/
    ├── Forgotten.Nft.UnitTests
    ├── Forgotten.Nft.IntegrationTests
    └── Forgotten.Nft.SecurityTests
```

### Dependency direction (enforced)

```
Domain  ←  Application  ←  Infrastructure
                ↑               ↑
                └──── Bot ──────┘   (composition root; also → Integrations)
                       Integrations → Application, Domain
                       WalletWeb    → Application, Infrastructure
```

- Domain depends on nothing external. A unit test
  (`LayeringTests`) asserts the Domain assembly references neither Discord.Net
  nor EF Core (spec §3).
- The Bot project is the only composition root — it wires Application +
  Infrastructure + Integrations together and owns the host.

## Technology baseline (spec §2.1)

| Concern | Choice | Version |
|---|---|---|
| Runtime | .NET | 10.0 LTS |
| Discord | Discord.Net | 3.20.2 |
| Persistence | EF Core + SQLite | 10.0.13 |
| HTTP resilience | Microsoft.Extensions.Http.Resilience | 10.11.0 |
| Options / DI / Logging / Hosting | Microsoft.Extensions.* | 10.0.x |
| Tests | xUnit | 2.9.3 |

## Host & configuration (Phase 0)

- **Generic Host** (`Host.CreateApplicationBuilder`) in `Forgotten.Bot/Program.cs`.
  Registration is split into `AddForgottenApplication` / `AddForgottenInfrastructure`
  / `AddForgottenBot` extension methods so the graph is composable and testable.
- **Configuration** is layered: `appsettings.json` (non-secret defaults) →
  environment-specific → environment variables (`Section__Key`) → user secrets.
- **Secrets** (Discord token, provider API keys) come only from user secrets or
  environment — never committed (spec §5.2, §16.11). `appsettings.Example.json`
  documents the shape without secrets.
- **Fail-closed validation**: all options use `ValidateDataAnnotations()` +
  `ValidateOnStart()`, plus cross-field `IValidateOptions<>` validators. A missing
  Discord token or invalid security-critical setting stops startup (spec §20) —
  verified in Phase 0 by both a unit test and an actual run.
- **Startup banner** logs environment, DB target, Solana network / DAS-configured
  flag, and the ownership-monitoring / POW feature flags.

## Persistence (Phase 0 baseline)

- `ForgottenDbContext` (EF Core) — no entities yet; Phase 1 adds NftAssets /
  NftAttributes, later phases add the remaining spec §9 tables.
- SQLite hardening via `ForgottenConnectionInterceptor` on every opened
  connection: `journal_mode=WAL`, `busy_timeout`, `foreign_keys=ON` (spec §9.2).
- `UtcDateTimeConverter` registered as a model-wide convention so all timestamps
  are stored/read as UTC (spec §9.2).
- Schema evolves **only** through EF Core migrations; the database is never
  auto-dropped or recreated (spec §9). Initial migration: `InitialCreate`.
- A design-time factory (`ForgottenDbContextFactory`) lets `dotnet ef` run
  without booting the host.

## Discord connection (Phase 0 baseline)

- `DiscordBotService` (a `BackgroundService`) owns the gateway lifecycle: login,
  start, `Ready`-driven slash-command registration (guild-scoped when
  `Discord:DevGuildId` is set, else global), interaction routing into the
  Discord.Net interaction framework, and clean logout on shutdown.
- No command modules exist yet (Phase 0); the registration pipeline is wired so
  Phase 1+ add modules without re-plumbing. Discord.Net logs are bridged into
  `Microsoft.Extensions.Logging`.

## What Phase 0 deliberately does NOT include

NFT domain/entities, DAS provider, metadata resolution, verification, wallet
flow, gallery, monitoring, roles, and game integration — each arrives in its
named phase (see [phases/README.md](phases/README.md)).
