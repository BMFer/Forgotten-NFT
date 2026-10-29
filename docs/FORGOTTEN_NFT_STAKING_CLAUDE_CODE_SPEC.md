# FORGOTTEN — NFT Loyalty Staking Module

**Implementation specification for Claude Code**  
**Target:** Existing FORGOTTEN repository (`BMFer/Forgotten`)  
**Stack:** .NET 10, Discord.Net, EF Core + SQLite, existing Ed25519 wallet verification, the interactive `/nft showcase` embed, and Helius **DAS (RPC)** integration  
**Status:** Proposed. Architecture reconnaissance is captured in §2; **note: there is no Helius webhook infrastructure today** — ownership is tracked by a polling monitor (see §2, §5).

## 1. Objective

Add an optional **Stake** action to the **existing interactive `/nft showcase` Discord embed** — the per-NFT view a verified holder browses (prev/next · inspect · flex · collection dropdown). The one-shot public **📢 Flex** post is a separate, non-interactive message and is **not** where staking lives. Reuse the existing verified Discord-user-to-Solana-wallet association (`DiscordWalletConnection`) and NFT ownership validation (`ISolanaAssetProvider` / DAS). Holders retain their NFTs in their own wallets. FORGOTTEN records opt-in participation, monitors ownership (DAS polling today; optionally Helius webhooks later), and calculates project-configured rewards. **NFT projects—not FORGOTTEN—must supply any rewards.**

This first phase is **non-custodial loyalty staking**, not a blockchain lock, escrow, or yield-generating on-chain stake. Do not advertise the NFT as locked or promise financial returns. Holders can transfer their NFTs at any time; transfers end eligibility under the configured policy.

## 2. Current architecture (reconnaissance)

These building blocks already exist and must **not** be duplicated:

| Capability | Where it lives |
|---|---|
| Interactive per-NFT embed (the "Flex embed" this spec refers to) | `/nft showcase` — `NftGalleryService` (Application), `GalleryComponents` / `GalleryInteractions` (Bot), `GalleryView` DTO. One NFT per page with a stable asset id; buttons carry a stable `custom_id` (`nftgal:{sessionId}:{action}:{page}`) and every interaction re-checks session ownership. |
| Public one-shot flex post | The **📢 Flex** button posts a non-interactive embed — **not** a staking surface. |
| Verified Discord→wallet mapping | `DiscordWalletConnection` (table `DiscordWalletConnections`); verification is Ed25519 sign-in via the separate `Forgotten.Nft.WalletWeb` process. |
| Fresh NFT ownership check | `ISolanaAssetProvider` (DAS via Helius `getAsset` / `getAssetsByOwner`). **Solana is source of truth.** |
| Collection allowlist / default | `ICollectionRegistryService` (table `NftCollections`, per-guild, `IsApproved` / `IsDefault`). |
| Admin authorization | `nft-admin` command group — Administrator-only, guild-only, fail-closed (attribute-gated + test-enforced). |
| Persistence + migrations | EF Core + SQLite (WAL + busy timeout); migrations **auto-applied on startup** (`Database.Migrate`). Add migrations with `dotnet ef migrations add` in `Forgotten.Nft.Infrastructure`. |
| Background ownership reconciliation | `OwnershipMonitoringWorker` (Bot) + `RoleReconciliationService` (Application) — **polling** via DAS on an interval; grants/removes roles; **never strips roles on a provider outage**. |
| Helius | **DAS RPC only.** There is **no webhook receiver, inbox, or event queue** today (see §5). |

**Two-process topology:** the **bot** handles Discord interactions (the Stake button) and the polling monitor; the **walletweb** is the only HTTPS-facing process and shares the same SQLite DB. Any future webhook endpoint would live on **walletweb** and write to a shared inbox the bot drains.

Remaining S0 task: produce the concise impacted-file plan + migration steps before coding, following repository conventions.

## 3. Holder experience: modify the existing `/nft showcase` embed

- On an eligible NFT displayed in the current `/nft showcase` embed, add a **Stake** button associated with that exact NFT asset and collection.
- For an active stake, show **Staked** status and an **Unstake** action; display earned/pending points when supported.
- For ineligible NFTs or unverified users, do not offer an actionable Stake button. Provide a short reason where appropriate.
- A button click must acknowledge/defer within Discord's interaction deadline, authorize the Discord user, resolve the verified wallet, and **freshly confirm NFT ownership** using the existing verification service before creating a stake.
- Show ephemeral success/failure feedback and refresh the existing `/nft showcase` view (reuse `GalleryInteractions` + `GalleryComponents`); do **not** build a second showcase/gallery or a redundant wallet-verification flow.
- Each showcase page is a single NFT; bind the action to that page's **stable asset id** (the `custom_id` already carries the session + page), never to raw page position. Validate stale/forged/replayed component interactions server-side (the existing per-session ownership re-check is the model).
- Unstake ends future accrual; whether previously earned points are retained follows the project's published policy.

## 4. Project administration and funding

Authorized project admins configure: eligible verified collections, program enable/disable, start/end dates, point accrual rate, cooldowns, minimum holding time, caps, reward descriptions, redemption rules, and transfer/unstake policy. Default to **off** until configured.

Phase 1 rewards are **off-chain loyalty points, Discord roles, badges, access, or manually fulfilled project perks**. FORGOTTEN does not guarantee delivery or finance perks. Maintain an explicit funding/fulfillment owner per program. A project may publish a reward only after confirming its own ability to fulfill it.

Token payouts, escrow, and automatic on-chain reward distribution are **out of scope** for Phase 1. Add them later only as separately designed and security-reviewed modules with project-funded reserves and verified authorization.

## 5. Ownership monitoring

**Current baseline:** ownership is tracked by **polling** — `OwnershipMonitoringWorker` re-checks owners via DAS on an interval. **There are no Helius webhooks installed.** Transfer detection for staking can therefore be delivered two ways:

- **(Default) Extend the existing polling reconciliation.** Add active stakes to the interval sweep; detect transfers by comparing the current DAS owner to the staked wallet. Simple, reuses existing infra, bounded lag = poll interval.
- **(Optional, new infra) Add a Helius webhook pipeline.** A webhook receiver on the **walletweb** process (the only HTTPS-facing service) authenticates deliveries, persists the delivery/event ID, and writes to a durable **inbox** table the **bot** drains **idempotently**. Confirm event payloads and subscription capabilities against current Helius docs before relying on them. Webhooks are **signals, not proof** — always re-verify via DAS.
- Resolve the NFT's current owner via the **existing ownership verification path** before deciding whether to terminate a stake; webhooks are signals, not standalone proof.
- On confirmed loss of ownership: close the active stake at the policy-defined effective time, stop future accrual, invalidate staking eligibility (hide the Stake action in `/nft showcase`), and revoke any staking-derived Discord benefit when applicable. Preserve an audit record.
- If verification is unavailable or inconclusive, mark the stake **PendingVerification**, pause new claims/redemptions, retry with backoff, and alert operators; do not falsely label the asset transferred.
- Add scheduled reconciliation for active stakes and re-verify immediately before claims, redemptions, and privileged role grants. Account for webhook delays, duplicates, out-of-order events, and missed deliveries.
- Define an explicit event-time versus detection-time accrual policy and document possible chain-finality/reorganization behavior.

## 6. SQLite data model

Extend the existing SQLite schema using its established migration approach. Suggested logical entities (adapt naming to existing tables):

| Entity | Required data |
|---|---|
| `StakingPrograms` | Project/collection, enabled flag, policy version, rate, eligibility rules, start/end |
| `Stakes` | Stake ID, program ID, NFT mint/asset ID, verified wallet, Discord user, started/ended timestamps, state, policy version |
| `RewardLedger` | Immutable accrual/adjustment/redemption entries, amount, period, reason, idempotency key |
| `WebhookInbox` | Provider event/delivery key, payload reference, received/processed timestamps, status, retry count |
| `OwnershipChecks` | Stake/asset, observed owner, verification result, checked time, source |
| `ProgramAudit` | Actor, action, target, old/new policy or state, timestamp |

Enforce **one active stake per asset per program** with a partial unique index; use foreign keys, transactions, UTC timestamps, and appropriate indices. Enable WAL mode and busy timeout; keep writes short and serialize competing state changes. Ledger entries must be deterministic and idempotent: no double accrual or double redemption on retries. Back up the database and test restore. Avoid storing secrets or unnecessary wallet personal data.

## 7. State machine and reward accounting

States: `Active`, `PendingVerification`, `EndedByHolder`, `EndedByTransfer`, `EndedByAdmin`, `Suspended`.

- Stake creation: verified user + authorized wallet + eligible asset + fresh ownership + program enabled -> `Active`.
- Accrual: calculate from validated eligibility intervals using versioned project rules; cap at configured limits; write immutable ledger entries with unique period keys.
- Transfer detected: verify actual ownership -> `EndedByTransfer`; ambiguous result -> `PendingVerification`.
- Unstake: authenticated owner -> `EndedByHolder` (idempotent).
- Reacquisition: never silently reactivate an ended stake; require a new opt-in.
- Program changes: preserve rule version applied to past accruals; never silently rewrite history.
- Claim/redemption: validate current policy, eligibility, balances, and project fulfillment availability; atomically reserve/deduct points and record fulfillment status.

## 8. Architecture boundaries

- **Discord UI adapter:** existing `/nft showcase` embed — `GalleryComponents` (buttons) + `GalleryInteractions` (handler).
- **Staking application service:** authorization, enrollment, state transitions, rules, and ledger (new, in `Forgotten.Nft.Application`).
- **Ownership adapter:** existing `ISolanaAssetProvider` (DAS) + wallet verification (`DiscordWalletConnection`); Solana is the source of truth.
- **Transfer-detection adapter:** the existing **polling** `OwnershipMonitoringWorker` today; a Helius **webhook receiver + durable inbox** is **new** (on `walletweb`) and only added if/when webhooks are adopted (§5).
- **Persistence:** existing `ForgottenDbContext` + repositories and EF Core migrations (auto-applied on startup).
- **Project admin:** reuse the `nft-admin` command group (Administrator-only, guild-only, fail-closed).

Use dependency injection, structured logs, cancellation tokens, and interfaces aligned with current architecture. Do not introduce a new framework, database, or custody contract for this phase.

## 9. Security and operational requirements

- Never trust embed text, button payloads, cached ownership, or a client-provided wallet address as authorization.
- Scope every action to the invoking Discord user, verified wallet, project, and exact NFT asset.
- Protect admin changes with server-side permissions; log all policy edits and manual adjustments.
- If Helius webhooks are adopted, validate their authenticity; rate-limit interactions (and any webhooks); make all mutations replay-safe.
- Do not store wallet private keys or take custody of NFTs. Do not promise APY or guaranteed financial returns.
- Expose metrics: active stakes, transfer-detection lag, verification failures, queue depth, webhook retries, accrual discrepancies, and Helius usage/credits.
- Support a global kill switch and per-program pause; paused programs must not issue unverified rewards.

## 10. Development phases

### Phase 0 — Inspect and design
Map existing Flex/verification/Helius/SQLite code. Produce impacted-file plan, policy assumptions, and test strategy. Seek confirmation before architectural changes.

### Phase 1 — Stake button and records
Implement schema, program configuration, stake/unstake services, and buttons **inside the existing `/nft showcase` embed**. Reuse the existing ownership checks. Show status and audit events. No automatic financial payouts.

### Phase 2 — Transfer monitoring and reconciliation
There is **no existing webhook pipeline**. Either extend the existing **polling** reconciliation (`OwnershipMonitoringWorker`) to cover active stakes, **or** add a new Helius webhook pipeline (receiver on `walletweb` + durable idempotent inbox the bot drains). Add verification, scheduled reconciliation, state transitions, Discord benefit revocation, and alerts.

### Phase 3 — Project rewards and dashboard
Add configurable off-chain rewards, ledger, redemptions, project-funded fulfillment workflow, admin reporting, and holder status display.

### Phase 4 — Optional future on-chain rewards
Separate proposal for token reward funding, contracts/transactions, custody risk review, audits, and legal/compliance evaluation. Not part of MVP.

## 11. Acceptance criteria

1. A verified NFT owner can click **Stake directly on the existing `/nft showcase` embed** and see the same embed update.
2. Existing wallet and NFT ownership verification services are reused; no duplicate verification implementation is introduced.
3. Another Discord user, another wallet, a stale embed, or a forged interaction cannot stake/unstake the NFT.
4. Duplicate button clicks and duplicate/out-of-order Helius events never create duplicate stakes or ledger credits.
5. Confirmed transfer stops eligibility and updates the `/nft showcase` experience; transient RPC failure does not falsely assert a transfer.
6. A scheduled ownership check catches a transfer missed by webhook processing.
7. A stake can be ended voluntarily; reacquisition requires a fresh Stake action.
8. Project admins can enable/disable programs and configure project-supplied rewards; FORGOTTEN never funds payouts.
9. SQLite migration, concurrency, backup/restore, and restart recovery are tested.
10. Tests cover ownership changes, pending verification, reward calculations, authorization, replay protection, and Discord UI state.

## 12. Instructions to Claude Code

> Read this specification (and §2 for the mapped architecture) before coding. Treat the existing **`/nft showcase`** Discord embed, the verified wallet mapping (`DiscordWalletConnection`), the NFT ownership checks (`ISolanaAssetProvider` / DAS), the **polling** ownership monitor, and the SQLite database as canonical reusable components — and remember there is **no Helius webhook pipeline today**. First provide a repository-specific implementation plan with file paths, schema changes, event/ownership handling, and tests. Then implement Phase 1 in small, testable commits. Do not build a new showcase/gallery, wallet-verification service, NFT custody contract, or token payout engine. Preserve existing behavior and document all configuration and operational requirements.
