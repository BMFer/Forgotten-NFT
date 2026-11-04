# Phase S2 — Transfer monitoring & stake reconciliation

**Spec refs:** [`STAKING-ROADMAP.md`](../STAKING-ROADMAP.md) S2; [`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](../FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md) §5, §7
**Status:** ✅ **complete & live-verified in production (2026-11-04)** — a transferred staked NFT auto-ended (`EndedByTransfer`) and the end folded into the single transfer embed.
**Deliverable:** When a staked NFT leaves the holder's verified wallet, the stake auto-ends
(`EndedByTransfer`); ambiguous ownership parks it at `PendingVerification` and retries — never falsely
ended. No new webhook infra (reuses Phase 5.1), no duplicate posts.

> Builds on Phase 5.1 (webhook drain) and S1 (stakes). The spec predated the webhook pipeline; S2 reuses
> it rather than building the "optional" pipeline it describes.

---

## 1. Design decisions (approved)

- **Detection = webhook (primary) + scheduled backstop.** A staked NFT only exists in a **verified**
  wallet; transferring it out debits that watched wallet, so the existing Helius webhook fires within
  seconds. A periodic sweep re-verifies the least-recently-checked live stakes to catch any missed
  delivery and to retry `PendingVerification`. (Tracking *unverified* collection holders is a separate
  future feature — **F1** — not needed here.)
- **No duplicate posts.** When a stake ends from a webhook transfer, it is **folded into the single
  transfer embed** (a "🔁 Stake ended by transfer" field). Only ends found by the backstop sweep (no
  transfer embed in play) post a standalone stake-ended embed.
- **Signal, not proof.** Every end is gated on a fresh DAS re-verify; a provider failure never asserts
  a transfer.
- **Records-only.** No reward/benefit revocation (there are no staking-derived benefits until S3).

## 2. Components

- **`StakeReconciliationService`** (Application): `ReconcileMemberAsync(guild, user)` (webhook path) and
  `ReconcileBatchAsync(max)` (sweep). Per live stake: re-resolve on-chain → owner in the user's verified
  wallets ⇒ `StillOwned` (clears a prior pending back to `Active`); not owned / burned ⇒
  `EndedByTransfer` (+ `ProgramAudit`); provider error ⇒ `PendingVerification`.
- **`Stake.LastCheckedAt`** (migration `AddStakeLastChecked`) + `IStakeRepository.ListLiveForReconciliationAsync`
  (least-recently-checked first) drive the sweep.
- **`WebhookDrainWorker`**: after role reconcile, reconciles the member's stakes and folds any ended
  stake into the transfer announce.
- **`StakeReconciliationWorker`** (every 10 min, gated on `EnableStaking`): backstop sweep; announces
  its ends standalone via `StakeAnnouncer` (`EndedByTransfer` kind).
- **Announcers** reuse the shared `TracksEmbedFormat`; `StakeAnnouncer.AnnounceAsync` now takes a
  `StakeEventKind` (Staked / Unstaked / **EndedByTransfer**).

## 3. Tests

- `StakeReconciliationServiceTests` (5): still-owned keeps Active; transferred → `EndedByTransfer` +
  audit; burned → ended; provider throws → `PendingVerification` (never ended); pending clears to Active.
- Full suite green (163). Migration applies additively; `LastCheckedAt` present.

## 4. Exit Gate

- [x] Staked NFT transfer out → `EndedByTransfer` (fresh DAS re-verify), audit row. §5, §7
- [x] Ambiguous/outage → `PendingVerification` + retry on the sweep; never falsely ended. §5
- [x] Webhook path folds the end into the transfer embed; sweep path posts standalone — **no duplicate posts**.
- [x] Reuses the existing webhook drain + a cheap backstop over live stakes (not all wallets).
- [x] Build clean; full suite green (163); migration additive.
- [x] ✅ **Live end-to-end (2026-11-04)** — a staked NFT transferred out (tx `3Zoq…tnEW`, inbox id 16) →
      stake Id 6 flipped `Active → EndedByTransfer` (`stake.end` audit), and the **single** transfer embed
      carried the "🔁 Stake ended by transfer" field (no duplicate post). Bot log `Stake ended by transfer …`.

### Deferred
- **S3** — accrual, `RewardLedger`, redemptions, holder status/points.
- **F1** — collection holder census (unverified wallets) — separate future phase.
