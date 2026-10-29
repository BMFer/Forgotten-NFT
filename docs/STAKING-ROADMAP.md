# FORGOTTEN — NFT Loyalty Staking Roadmap

A phased plan to add **non-custodial loyalty staking** to the FORGOTTEN engine, derived from
[`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md) and
adapted to this repository's architecture.

Each phase is a self-contained, tested vertical slice — the same discipline as the core engine
phases in [`docs/phases/`](phases/README.md).

## What this is (and isn't)

- **Non-custodial loyalty staking** — holders **keep their NFTs in their own wallets**. "Staking"
  records an opt-in; it is **not** an on-chain lock, escrow, or yield product.
- **Projects fund rewards, not FORGOTTEN.** Phase 1 rewards are off-chain loyalty points, Discord
  roles/badges, access, or manually fulfilled perks. No APY, no guaranteed financial returns.
- **Programs default to OFF** until a project admin configures and enables them.
- Holders can **transfer at any time**; a confirmed transfer ends eligibility per the program's policy.
- Token payouts / escrow / automatic on-chain distribution are **explicitly out of scope** for the MVP
  (a separate, security-reviewed proposal — Phase S4).

## Where the Stake action lives

The Stake / Unstake / Staked status lives on the **interactive `/nft showcase` embed** — the per-NFT
view a verified holder browses (prev/next/inspect/flex), **not** the one-shot public 📢 Flex post. Each
action binds to a **stable asset id** (never page position), defers within Discord's deadline,
re-authorizes the user, resolves their verified wallet, and **freshly confirms ownership** before any
state change. Ineligible NFTs / unverified users get no actionable button.

## Reused components (do not duplicate)

| Need | Existing component |
|---|---|
| Verified Discord→wallet mapping | `DiscordWalletConnection` + wallet verification flow |
| Fresh NFT ownership check | `ISolanaAssetProvider` (DAS/Helius), ownership verification path |
| Collection allowlist / default | `ICollectionRegistryService` (`NftCollections`) |
| Interactive per-NFT UI + components | `/nft showcase` (`GalleryComponents` / `GalleryInteractions`) |
| Admin authorization | `nft-admin` group (Administrator-only, guild-only, fail-closed) |
| Persistence + migrations | EF Core + SQLite, auto-migrate on startup |
| Background reconciliation pattern | `OwnershipMonitoringWorker` / `RoleReconciliationService` |

## Phases

| Phase | Deliverable | Status |
|---|---|---|
| **S0 — Inspect & design** | Confirm the showcase embed/handlers, wallet verification, ownership path, Helius capabilities, and migration approach. Produce an impacted-file plan, policy assumptions, and test strategy. | ⬜ Planned |
| **S1 — Stake button & records** | SQLite schema (`StakingPrograms`, `Stakes`, `ProgramAudit`); admin program config; stake/unstake application service reusing fresh ownership checks; **Stake/Unstake/Staked buttons on the `/nft showcase` embed**; audit events; replay-safe interactions; one active stake per asset per program. No payouts. | ⬜ Planned |
| **S2 — Transfer monitoring & reconciliation** | Durable, authenticated, idempotent Helius webhook inbox; scheduled reconciliation of active stakes; state transitions (`EndedByTransfer` / `PendingVerification`); revoke staking-derived Discord benefits; operator alerts + metrics. | ⬜ Planned |
| **S3 — Project rewards & dashboard** | Configurable off-chain rewards + immutable `RewardLedger`; redemptions with **project-funded** fulfillment workflow; admin reporting; holder status in the showcase. | ⬜ Planned |
| **S4 — On-chain rewards (future, out of MVP)** | Separate proposal only: token funding, contracts/transactions, custody risk review, audits, legal/compliance. Not built as part of the MVP. | ⬜ Future |

## Data model (S1–S3, adapt to conventions)

`StakingPrograms` · `Stakes` · `RewardLedger` · `WebhookInbox` · `OwnershipChecks` · `ProgramAudit` —
UTC timestamps, foreign keys, a **partial unique index for one active stake per asset per program**,
and deterministic, idempotent ledger entries (no double accrual/redemption on retries). WAL + busy
timeout as today.

## State machine

`Active` · `PendingVerification` · `EndedByHolder` · `EndedByTransfer` · `EndedByAdmin` · `Suspended`.
Accrual is computed from validated eligibility intervals under a **versioned** policy (past accruals
keep the rule version they were earned under); re-verify immediately before claims, redemptions, and
privileged grants; reacquisition requires a fresh opt-in (never silent reactivation).

## Non-negotiable safeguards

- Never trust embed text, button payloads, cached ownership, or a client-supplied wallet as authorization.
- Scope every action to the invoking user, their verified wallet, the program, and the exact asset.
- Authenticate webhook deliveries; rate-limit; make every mutation replay-safe; process events idempotently.
- No custody of NFTs or keys; no APY / financial guarantees. Global kill switch + per-program pause.
- A confirmed transfer stops eligibility; a transient RPC failure must **not** falsely assert a transfer.

Full requirements and acceptance criteria: [`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md).
