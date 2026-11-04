# Phase S3 — Rewards: accrual & balance

**Spec refs:** [`STAKING-ROADMAP.md`](../STAKING-ROADMAP.md) S3; [`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](../FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md) §4, §6, §7
**Status:** ✅ code complete + unit-tested; 🟢 deployed; ⏳ **first accrual lands overnight** (daily worker)
**Deliverable (this slice, S3.0):** Stakes earn **off-chain loyalty points** that accrue daily into an
immutable ledger, and members see their balance with **`/rewards`**. **Redemption** (structured reward
catalog + project-funded fulfillment) is the next slice, **S3.1**.

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

### Deferred
- **S3.1** — redemption: structured reward catalog (name/cost/desc), redeem → admin-fulfill workflow,
  admin reporting. Caps + final partial-day accrual on stake end also land here.
- **F1** — collection holder census (unverified wallets).
