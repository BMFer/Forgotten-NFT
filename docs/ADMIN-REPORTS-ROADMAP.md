# FORGOTTEN — Admin Reports Roadmap

A phased plan for the **admin-facing reporting suite**: the embeds a project admin runs to operate the
community (economy, fulfillment, membership, staking, holders, health). Derived from the data the engine
already captures and adapted to this repository's architecture.

Each report is a self-contained, tested vertical slice — the same discipline as the staking phases in
[`STAKING-ROADMAP.md`](STAKING-ROADMAP.md) and the core engine phases in [`docs/phases/`](phases/README.md).

## Principles (hold across every report)

- **Builder pattern.** Each report is a **pure, unit-tested embed builder** (like
  `StakingReport`) that takes already-assembled plain
  data; the command only gathers data and posts. No Discord/DB needed to test the rendering.
- **Cheap by default.** Prefer the **local SQLite** tables we own. Anything backed by live on-chain
  reads (DAS/Helius) must **cache, cap, and log truncation** — never silently partial.
- **Guild-scoped, admin-gated, fail-closed.** Every report lives under the `nft-admin` group
  (Administrator-only, guild-only). Caveat: **wallet verification is global, not per-guild** (ownership
  is cross-guild by design), so membership metrics are deployment-wide unless joined against guild
  presence — called out where relevant.
- **Discord-limit-aware.** 25 fields / 1024 chars per field / 6000 total. Cap rows and append a
  **"+N more"** marker so truncation is visible.
- **Time windows.** Trend reports accept a window (`7d` / `30d` / `all`), default `30d`.
- **Consistent surface.** Post to `Discord:AdminChannelId` by default with an optional `[channel]`
  override — same as `/nft-admin staking-report`.

## Existing reports (do not duplicate)

| Have | Type | Surface |
|---|---|---|
| `/nft-admin rewards-report` | Economy **aggregate** (issued/redeemed/outstanding/earners + redemption counts) | ephemeral |
| `/nft-admin staking-report` | **Who's staked**, per member, per program | admin channel |
| `/nft-admin redemptions` | Pending fulfillment queue | ephemeral |
| `/nft-admin verification-log` | Recent verification activity | ephemeral |
| `/nft-admin staking-list` · `reward-list` · `collection-list` · `role-list` · `provider-status` | Config **listings** (not analytics) | ephemeral |

## Data sources

| Entity | Powers | Cost |
|---|---|---|
| `RewardLedgerEntry` (`CreatedAt`, `Kind`, `Amount`) | economy trend, top earners/spenders, liability | local |
| `Redemption` (`RequestedAt`, `FulfilledAt`, `Status`, `RewardItemId`, `Cost`) | fulfillment SLA, catalog performance | local |
| `RewardItem` (`Cost`, `Stock`, `Enabled`) | catalog performance | local |
| `DiscordWalletConnection` (`VerifiedAt`) | member summary *(global scope)* | local |
| `Stake` (`StakedAt`, `EndedAt`, `EndReason`, `LastCheckedAt`) | staking activity/trend, reconciliation lag | local |
| `ProgramAudit` | config-change audit | local |
| `WebhookInboxEntry`, `Stake.LastCheckedAt` | system/ops health | local |
| DAS / `ISolanaAssetProvider`, `ILeaderboardService` | holder census, role drift | **on-chain** |

## Phases

| Phase | Deliverable | Status |
|---|---|---|
| **R1 — Economy & fulfillment ops** | `/nft-report` group: `fulfillment`, `economy`, `catalog`, `members` (+ relocated `rewards`, `staking`) — cheap local aggregates to run the rewards program day-to-day | ✅ Built + tested |
| **R2 — Member-centric & activity** | `member-report @user` (Member 360), staking activity/trend, config-change audit | ⬜ Planned |
| **R3 — On-chain truth** | Holder census per collection, role drift/audit (eligible-vs-assigned) — DAS-backed, cached + capped | ⬜ Planned |
| **R4 — Ops dashboard & digests** | System-health report + optional **auto-posted** daily/weekly digest worker | ⬜ Future |

Rationale for the order: R1 is lowest-risk, highest-frequency value and reuses the `StakingReport`
pattern wholesale. R2 adds the support-desk workhorse (Member 360). R3 is a different **risk class**
(live on-chain calls, Helius credits, pagination) and must be a conscious step. R4 turns reporting from
pull to push and needs a scheduler.

---

## R1 — Economy & fulfillment ops (✅ built)

Four reports, all **local-DB aggregates**, each a pure builder + a thin `nft-admin` command posting to
the admin channel. No schema changes.

> **Implementation note — read-model over repo methods.** Rather than bolting report-only aggregations
> onto the domain repositories (as sketched below), R1 landed them on a dedicated
> `IAdminReportQueries` read-model
> (impl `AdminReportQueries`).
> This keeps report queries off the domain repos and — pragmatically — touched **zero** existing test
> fakes (the repo interfaces are implemented by ~10 doubles). The "new repo method" lines below name the
> capability; all live on `IAdminReportQueries`.

### Surface decision (revised at build time)
Originally planned as individual `/nft-admin *-report` commands. **Discord caps a single command at 25
sub-commands**, and `/nft-admin` was already at 25 — so all reports moved to a dedicated top-level
**`/nft-report`** group (`rewards`, `staking`, `fulfillment`, `economy`, `catalog`, `members`), admin-gated
like `nft-admin`. This also future-proofs R2–R4. Reports post to `Discord:AdminChannelId` by default with
an optional `[channel]` override (`rewards` is ephemeral).

### R1.1 `/nft-report fulfillment` — redemption SLA
**Answers:** is the fulfillment queue healthy? **Shows:** pending count + **age of the oldest pending**,
fulfilled / cancelled in the window, **avg & median time-to-fulfill**, and the 5 oldest pending (id,
member, reward, waiting-for).
- Data: `Redemption`. Reuses `GetStatusCountsAsync` + `ListPendingByGuildAsync`.
- **New repo method:** `IRedemptionRepository.GetFulfillmentStatsAsync(guildId, since)` → fulfilled
  `(RequestedAt, FulfilledAt)` pairs (or pre-aggregated avg/median + counts) for the window.

### R1.2 `/nft-report economy` — points economy
**Answers:** is point issuance outrunning redemption? **Shows:** issued vs redeemed vs net over the
window, **outstanding liability** (all-time issued − redeemed), and **top 5 earners** + **top 5
spenders**.
- Data: `RewardLedgerEntry`. Complements `rewards-report` (which is a point-in-time per-program total)
  with a **trend + leaderboard** view.
- **New repo methods:** `GetActivitySinceAsync(guildId, since)` (sums by `Kind`), `GetTopEarnersAsync`
  / `GetTopSpendersAsync(guildId, limit)`.

### R1.3 `/nft-report catalog` — reward performance
**Answers:** which rewards actually move? **Shows:** per `RewardItem` — redemptions (pending/fulfilled/
cancelled), **points spent**, stock remaining, enabled/disabled; sorted by redemptions.
- Data: `RewardItem` (`ListByGuildAsync`) joined to `Redemption` grouped by `RewardItemId`.
- **New repo method:** `IRedemptionRepository.GetCountsByItemAsync(guildId)` → per-item counts + points.

### R1.4 `/nft-report members` — verified-member summary
**Answers:** how is verification growing? **Shows:** total verified members, total wallets, **new in
window**, multi-wallet members.
- Data: `DiscordWalletConnection`. **Scope caveat:** connections are **global** (cross-guild); for a
  single-guild deployment this equals the guild. A strict per-guild count needs a Discord
  presence join — deferred; documented in the embed footer.
- **New repo method:** `GetVerificationSummaryAsync(since)` → totals + new-in-window + multi-wallet count.

### Components & tests
- One builder each in `src/Forgotten.Bot/Components/` (`FulfillmentReport`, `EconomyReport`,
  `CatalogReport`, `MembersReport`) — pure, returning `Embed` from record inputs.
- Commands in `NftAdminCommands` mirroring `StakingReportAsync` (defer ephemeral → assemble → post to
  target → ephemeral confirm; `MissingPermissions` handled).
- Repo methods implemented in Infrastructure with EF; **SQLite caveat**: `decimal` is summed
  client-side (as existing ledger reads do).
- Tests mirror `StakingReportTests` / `RewardReportRepositoryTests`: builder format + caps/limits
  (pure), and real-SQLite aggregation for each new repo method (windowing, empty-state, cross-guild
  isolation).

### Exit gate
- [x] Four commands post correct embeds to the admin channel; `[channel]` override honored.
- [x] Each read-model query covered by a migrated-SQLite test (window + empty + guild isolation).
- [x] Builders unit-tested for row caps / empty-state / field content.
- [x] Build clean; full suite green (197). Docs: admin How-To + README admin table.
- [ ] ⏳ Live-verify one report in the admin channel (as with `staking-report`).

### Deferred from R1 (→ later phases)
- Per-guild (presence-joined) member counts; holder/role on-chain reports (R3); scheduled digests (R4).
