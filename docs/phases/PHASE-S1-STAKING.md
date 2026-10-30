# Phase S1 — Stake button & records

**Spec refs:** [`STAKING-ROADMAP.md`](../STAKING-ROADMAP.md) S1; [`FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md`](../FORGOTTEN_NFT_STAKING_CLAUDE_CODE_SPEC.md) §3, §4, §6, §7, §8, §9, §11
**Status:** ✅ code complete + unit-tested; 🟢 deployed to production (2026-10-30, migration applied); ⏳ **live end-to-end pending** (no real stake observed yet)
**Deliverable:** A verified holder can Stake/Unstake an eligible NFT directly on the existing
`/nft showcase` embed. Non-custodial — records opt-in only; no payouts (rewards are S3).

> S0 (inspect & design) is complete: the showcase gallery, `nft-admin` auth, wallet/ownership path,
> and SQLite conventions were mapped and an impacted-file plan approved before coding. Note the spec
> predates the webhook pipeline built in Phase 5.1/5.2 — S2 will reuse that existing drain rather
> than building new infra.

---

## 1. Design decisions (approved at S0)

- **Records only** — S1 enrolls/ends stakes and stores the config rate; no accrual/points math or
  ledger (that is S3). `StakingProgram.AccrualRatePerDay` is persisted but unused in S1.
- **Approved collections only** — a program's collection must be approved in the registry
  (`ICollectionRegistryService`).
- **Hide when ineligible** — no Stake button unless an enabled program covers the NFT's collection
  (spec §3: "ineligible NFTs get no actionable button").
- **Global kill switch** — `NftEngine:EnableStaking` (off by default) gates the whole module on top
  of each program's `Enabled` flag (spec §9).

## 2. Data model (migration `AddStakingTables`)

- **`StakingPrograms`** — per (guild, collection); `Enabled` off by default; `PolicyVersion` bumps on
  policy-affecting edits; rate/min-hold/start/end. Unique (GuildId, CollectionAddress).
- **`Stakes`** — program, asset, collection, wallet, user, `State`, `PolicyVersion`, `StakedAt`,
  `EndedAt?`, `EndReason?`. **Partial unique index** `(ProgramId, AssetAddress) WHERE "EndedAt" IS NULL`
  → one live stake per asset per program.
- **`ProgramAudit`** — immutable actor/action/target/old/new trail for every program + stake mutation.
- States: `Active · PendingVerification · EndedByHolder · EndedByTransfer · EndedByAdmin · Suspended`
  (transfer-driven transitions wired in S2).

## 3. Components

- **Application:** `IStakingProgramService` (admin config) + `IStakingService` (holder stake/unstake/
  status) with `StakeOutcome`/`StakeActionResult`/`StakeAssetStatus` models; three repositories.
- **Bot — admin:** `/nft-admin staking-configure | staking-enable | staking-disable | staking-list`
  added to the existing (Administrator-gated, test-enforced) `NftAdminCommands` module.
- **Bot — holder:** a Stake/Unstake button on the `/nft showcase` row (`GalleryComponents`) driven by
  a new `GalleryView.Stake` status; `stake`/`unstake` branches in `GalleryInteractions` reusing the
  session-owner + expiry gate and re-resolving the asset server-side.

## 4. Safety (spec §9) — enforced

- ⚠ Never trust the button payload: the asset is taken from the server-resolved `GalleryView.Item`,
  and `StakingService` **re-resolves the asset on-chain** (`ISolanaAssetProvider.GetAssetAsync`) and
  checks the owner against the user's verified wallet before any write.
- ⚠ Replay-safe: the partial unique index + repository `TryAddAsync` make a double-click idempotent
  (one live stake); unstake is idempotent and only the staking holder can unstake.
- ⚠ A provider failure returns an error — it never falsely asserts ownership or a transfer.
- Reacquisition requires a fresh Stake (ended stakes drop out of the live slot; no silent reactivation).

## 5. Tests (map to spec §11 acceptance)

- `StakingServiceTests` (14): global-off, no/disabled program, not-owner, provider-throws→error,
  success persists Active, **double-click idempotent (one stake)**, unstake by owner, **unstake by
  non-staker rejected**, unstake idempotent, status Unavailable/Stakeable/Staked. → #3, #4, #5(partial), #7
- `StakingProgramServiceTests` (5): not-approved→throws, creates-disabled, policy-version bump,
  enable/disable, no-program→null. → #8
- Admin authorization already covered module-level by `AdminCommandGuardTests` (new subcommands inherit it).
- Migration applied clean to a fresh DB; partial unique index verified in the schema. → #9

## 6. Exit Gate

- [x] Stake/Unstake/Staked on the existing `/nft showcase` embed (no second gallery). #1
- [x] Reuses existing wallet + ownership verification (no duplicate verifier). #2
- [x] Another user / stale embed / forged interaction can't stake/unstake (session gate + server re-resolve + owner check). #3
- [x] Duplicate clicks never create duplicate stakes (partial unique index + idempotent service). #4
- [x] Voluntary unstake; reacquisition requires a fresh Stake. #7
- [x] Admins enable/disable + configure programs; FORGOTTEN funds nothing. #8
- [x] Migration applies to a fresh DB; one-active-stake index enforced. #9
- [x] Build clean (0 warnings); full suite green (149 passed).
- [x] Deployed to production (2026-10-30): `AddStakingTables` auto-applied on startup; the three
      tables and the partial unique index are confirmed present in the live DB; services healthy.
- [ ] ⏳ **Live end-to-end** — approve a collection, `staking-configure` + `staking-enable`, set
      `NftEngine:EnableStaking=true`, then Stake/Unstake from `/nft showcase`. **Not yet observed**
      (`EnableStaking` is still off in production; do not claim live success until then — spec §25).

### Deferred to later staking phases
- **S2** — transfer detection (reuse the Phase 5.1/5.2 webhook drain + polling), `EndedByTransfer` /
  `PendingVerification`, benefit revocation, `OwnershipChecks`.
- **S3** — accrual, `RewardLedger`, redemptions, holder status/points in the showcase.
