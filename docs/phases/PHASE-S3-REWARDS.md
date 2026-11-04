# Phase S3 — Rewards: accrual, balance & redemption

**Spec refs:** [`STAKING-ROADMAP.md`](../STAKING-ROADMAP.md) S3; [`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](../FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md) §4, §6, §7
**Status:** ✅ code complete + unit-tested; 🟢 deployed. **S3.0 accrual verified live** (controlled probe:
a backdated stake credited 3 idempotent daily rows = 30 pts, balance summed, cleaned up). **S3.1
redemption** deployed. **S3.2 dashboard** code-complete + unit-tested (admin report + gallery balance).
**Deliverable:** Stakes earn **off-chain loyalty points** that accrue daily into an immutable ledger
(**S3.0**); members see their balance with **`/rewards`** and spend points on a **structured reward
catalog** via **`/redeem`**, which admins fulfill manually (**S3.1**). FORGOTTEN funds nothing.

> Builds on S1/S2: accrual reads the program rate stored in S1 and the stake intervals S1/S2 maintain.
> Off-chain points only — FORGOTTEN never funds or auto-pays anything.

---

## 1. Design decisions (approved)

- **Daily worker → immutable ledger** (not compute-on-read). `RewardAccrualWorker` writes one
  append-only `RewardLedger` row per stake per eligible day; balance = sum of rows.
- **Accrual + balance first**; the redemption flow (structured reward items with name/cost/desc) is
  **S3.1**.
- **Idempotent & deterministic.** Unique `IdempotencyKey = accrual:{stakeId}:{yyyy-MM-dd}` → re-runs
  and restarts never double-credit. Only **complete, elapsed UTC days** accrue (today is excluded
  until it closes). Eligibility starts at `StakedAt + MinHoldMinutes`. Each row freezes the stake's
  `PolicyVersion` so past accruals keep their rule version.

## 2. Components

- **`RewardLedgerEntry`** (+ `RewardLedgerKind` Accrual/Adjustment/Redemption) + migration
  `AddRewardLedger`. Unique `IdempotencyKey`; `Amount` stored as text (decimal), **summed client-side**
  (SQLite has no server-side decimal aggregate).
- **`RewardAccrualService`** — per live stake with a configured rate, credits each complete eligible
  UTC day since its last accrual.
- **`RewardAccrualWorker`** — every 6h, gated on `EnableStaking`; idempotent so the interval affects
  only latency. Pure DB work (no gateway dependency).
- **`/rewards`** — guild-only, ephemeral: total balance + per-program breakdown (collection-named).

## 3. Tests

- `RewardAccrualServiceTests` (5): no-rate → nothing; one entry per completed eligible day; **idempotent
  re-run writes nothing**; min-hold defers accrual; rows carry the stake's `PolicyVersion`.
- Full suite green (168). Migration additive; unique idempotency index verified.

## 4. Exit Gate

- [x] Immutable, idempotent ledger; one accrual row per stake per eligible day; no double-credit. §6, §7
- [x] Eligibility honors `MinHoldMinutes`; versioned policy frozen per row. §7
- [x] `/rewards` shows balance + per-program breakdown.
- [x] Build clean; full suite green (168); migration additive (unique idempotency index).
- [ ] ⏳ **Live** — the daily worker credits yesterday's eligible stakes; confirm `/rewards` shows a
      growing balance (first entries land after the next UTC-day boundary + a worker run).

## 5. Redemption (S3.1)

- **`RewardItem`** catalog per (guild, program): name, cost, optional description + `Stock`. Admin
  `/nft-admin reward-add | reward-remove | reward-list`.
- **`Redemption`** record + `RedemptionService`: `/redeem <id>` checks the member's per-program
  balance, **deducts points immediately** via an immutable negative ledger entry
  (`redeem:{redemptionId}`), decrements stock, and files a **Pending** request. Admins see the queue
  with `/nft-admin redemptions`, fulfill with `/nft-admin reward-fulfill <id>` (after delivering the
  perk), or `reward-cancel <id>` to **refund** (positive adjustment `refund:{redemptionId}`) + restore
  stock. **Project-funded, manual fulfillment — FORGOTTEN never delivers or pays.**
- `/rewards` now also lists the redeemable catalog.
- Tests: `RedemptionServiceTests` (6) — redeem deducts + pending, insufficient points, disabled,
  out-of-stock + stock decrement, fulfill-only-when-pending, cancel refunds + restores stock.

## 6. Dashboard (S3.2)

- **`/nft-admin rewards-report`** — guild-only, ephemeral admin embed. Per staking program (collection-named,
  enabled/disabled marker): **active stake count**, points **issued** (sum of positive ledger entries),
  **redeemed** (absolute value of `Redemption`-kind entries), **outstanding** (issued − redeemed), and
  distinct **earners**. A footer field tallies guild redemptions by status (pending / fulfilled / cancelled)
  and the embed footer surfaces whether staking is globally on/off (`NftEngine:EnableStaking`).
- **Gallery footer balance** — `GalleryView.RewardsBalance` carries the member's total points; the
  `/nft` gallery footer shows `💠 N points` when staking is enabled. The lookup is best-effort — a ledger
  failure is logged at debug and simply omits the balance rather than breaking the gallery.
- New repository reads (aggregate server-side where SQLite allows, else client-side over a guild-scoped
  projection): `IRewardLedgerRepository.GetGuildSummaryAsync` (per-program issued/redeemed/earners) and
  `IRedemptionRepository.GetStatusCountsAsync` (counts by status). Both scope strictly to one guild.
- Tests: `RewardReportRepositoryTests` (4, real migrated SQLite) — summary rolls issued/redeemed/earners
  per program and orders by program, empty guild → empty; status counts tally by status and ignore other
  guilds, no redemptions → zeros. Fakes updated across the staking/gallery suites.

### Deferred
- Caps (per-program total-points cap) + final partial-day accrual on stake end.
- **F1** — collection holder census (unverified wallets).
