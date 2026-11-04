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

## Near-real-time ownership reconciliation (Helius webhook pipeline)

Alongside the Phase 5 polling monitor, the engine reacts to on-chain activity
within seconds via a Helius webhook. The polling `OwnershipMonitoringWorker`
remains as the periodic backstop; the webhook is the low-latency path.

```
Solana tx touching a watched wallet
        │
        ▼
  Helius (raw webhook, transactionTypes: Any)
        │  POST (authHeader = Helius:WebhookSecret)
        ▼
  WalletWeb  POST /api/helius/webhook ──► WebhookInbox (durable, idempotent)
        │                                   keyed by tx signature
        ▼
  Bot  WebhookDrainWorker (polls every 20s)
        │  match touched addresses → verified wallets
        ▼
  re-verify ownership on-chain (DAS) ──► reconcile NFT-gated roles
```

- **Registration** — `/nft-admin webhook-sync` upserts a single Helius webhook by
  callback URL (`HeliusWebhookRegistrar`): it finds an existing webhook with the
  same URL and `PUT`s it, else `POST`s a new one and then `PUT`s the full config
  (Helius's create does not reliably persist `accountAddresses`, so the watched
  wallets are applied with a follow-up `PUT`). Re-running is safe.
- **Raw, not enhanced** — the webhook is registered as a **raw** webhook
  (`transactionTypes: ["Any"]`). Helius's *enhanced* parser types MPL Core
  (`MplCoreAsset`) transfers as `UNKNOWN` and never emits them, so an enhanced
  webhook silently drops Core NFT transfers. A raw webhook delivers every
  transaction touching a watched wallet, and the engine re-verifies downstream.
- **Receiver** — `WalletWeb` exposes `POST /api/helius/webhook`. It authenticates
  the request against `Helius:WebhookSecret` (returns `503` until the secret is
  set, `401` on mismatch), parses **both** the enhanced shape (`signature` +
  `accountData`/`tokenTransfers`) and the raw shape (`transaction.signatures` +
  `transaction.message.accountKeys`) via `HeliusWebhookParser`, and enqueues each
  event into the durable `WebhookInbox` (idempotent on the transaction signature),
  returning fast. The webhook is a **signal, not proof**.
- **Drain** — the bot's `WebhookDrainWorker` (a `BackgroundService`) polls the
  inbox every 20s (bounded per cycle, DAS-paced), resolves the touched addresses
  to verified wallets, and runs the normal reconciliation path — **re-verifying
  ownership on-chain** before granting/removing roles rather than trusting the
  webhook payload. Processing is idempotent (re-processing an event just
  re-reconciles).

## Multi-tenant branding & guild attribution

One deployment can serve the wallet-verification site under several hostnames, each with its own
brand, without duplicating the app or the ownership model.

- **Config** — `TenantBrandingOptions` (the `Branding` section, bound in both the bot and walletweb)
  holds a `Default` brand plus per-host `Tenants` (`Host`, optional `GuildId`, title/heading/accent/
  banner/footer). `ResolveByHost` is case-insensitive and falls back to the default; unset tenant
  fields inherit it.
- **Per-host rendering** — behind Caddy the walletweb honors `X-Forwarded-Host` (`UseForwardedHeaders`)
  and resolves the brand from the host the member actually visited. `VerificationPage`/`IndexPage`
  substitute the brand's (HTML-escaped) title, heading, accent, and banner; unknown hosts get the
  FORGOTTEN default.
- **Attribution** — `/wallet connect` records the invoking guild on the `WalletVerificationChallenge`
  (nullable `GuildId`), and the verification link is built from that guild's tenant host when one is
  configured, so members land on their community's branded page.
- **Deliberately branding + attribution only** — a wallet connection stays **global** (one verified
  owner per Discord user), roles still reconcile wherever collections are mapped, and the Ed25519
  signed message still binds to the single canonical domain. Tenants change the look and the link,
  never the cryptographic domain binding or the ownership model.

## NFT loyalty staking (S1–S2)

An optional, **non-custodial** loyalty-staking module layered on the existing showcase, wallet, and
webhook infrastructure. Holders keep their NFTs in their own wallets — "staking" records an opt-in,
never a lock, escrow, or payout. Off by default behind a global `NftEngine:EnableStaking` switch plus
per-program enable.

- **Data** — `StakingPrograms` (per guild+collection), `Stakes` (with a **partial unique index** on
  `(ProgramId, AssetAddress) WHERE EndedAt IS NULL` → one live stake per asset per program), and an
  immutable `ProgramAudit`. State machine: `Active · PendingVerification · EndedByHolder ·
  EndedByTransfer · EndedByAdmin · Suspended`.
- **Admin** — `/nft-admin staking-configure | staking-enable | staking-disable | staking-list`
  (inherits the Administrator-gated, test-enforced `nft-admin` auth). A program's collection must be
  approved in the registry.
- **Holder (S1)** — a **Stake/Unstake button on the `/nft showcase` embed**. The asset is taken from
  the server-resolved page item (never the client), and `StakingService` **re-verifies ownership
  on-chain** before any write; the partial unique index makes double-clicks idempotent. Each
  stake/unstake posts a detailed embed to `#nft-tracks`.
- **Transfer reconciliation (S2)** — `StakeReconciliationService` ends a stake when its NFT leaves the
  holder's verified wallets. It runs **from the webhook drain** (near-real-time — a transfer out debits
  the watched wallet) and from a **scheduled backstop sweep** (least-recently-checked live stakes;
  retries `PendingVerification`). Webhooks are a **signal, not proof** — every end is gated on a fresh
  DAS re-verify, so a provider outage parks the stake at `PendingVerification` rather than falsely
  ending it. A webhook-driven end is **folded into the single transfer embed** (no duplicate post);
  sweep-detected ends post standalone.
- **Reuse, not duplication** — both the transfer and staking announcers share `TracksEmbedFormat`
  (Solscan links, sanitization, capped lists, owned-NFT scan). Rewards/accrual are **deferred to S3**.

## What Phase 0 deliberately does NOT include

NFT domain/entities, DAS provider, metadata resolution, verification, wallet
flow, gallery, monitoring, roles, and game integration — each arrives in its
named phase (see [phases/README.md](phases/README.md)).
