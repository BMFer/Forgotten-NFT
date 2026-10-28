# Phase 5 — Ownership Monitoring & Role Integration

**Spec refs:** §10 (all), §11 (all), §9 (`NftOwnershipHistory`), §10.3 events,
§15 invalidation, §22.5
**Status:** ✅ code complete (2026-10-21, branch `phase-5-monitoring`) — live role assignment needs the bot to have Manage Roles (see Exit Gate)
**Deliverable (spec §22):** The bot safely reacts to NFT ownership changes —
detecting them in the background, recording history, firing events, and
reconciling Discord roles.

---

## 1. Objective

Add an optional background worker that observes ownership of relevant NFTs,
records *observed* changes (distinct from confirmed transfers), publishes domain
events, and lets guilds map approved collections to Discord roles — all while
failing safe during provider outages.

## 2. Scope

**In scope**

- `IHostedService` ownership-monitoring worker (spec §10).
- Ownership snapshots + change detection + `NftOwnershipHistory` (spec §9, §10.2).
- Domain events (spec §10.3).
- NFT-based Discord role assignment/removal + reconciliation (spec §11).
- Cache invalidation on ownership/admin changes (spec §15).

**Out of scope**

- Game/POW eligibility consumption (Phase 6) — but `NftEligibilityChangedEvent`
  is defined here for Phase 6 to consume.

## 3. Prerequisites

- [ ] Phase 4 Exit Gate passed.
- [ ] `EnableOwnershipMonitoring` + `OwnershipPollingMinutes` config (spec §20).

## 4. Tasks

### 4.1 Monitoring worker (`Forgotten.Nft.Infrastructure/Workers`)
- [ ] `IHostedService` polling worker, enabled by config (spec §10).
- [ ] ⚠ Requirements (spec §10.1): configurable interval, **bounded
      concurrency**, incremental sync, provider rate-limit awareness,
      cancellation, failure recovery, duplicate-event prevention.
- [ ] Scope work to NFTs relevant to connected wallets + registered applications
      (spec §10) — do not scan everything.

### 4.2 Change detection
- [ ] On each cycle (spec §10.2): fetch fresh ownership → compare to stored owner
      → update snapshot → record observed change → publish event → notify subscribers.
- [ ] ⚠ Record an **observed** state change; do **not** infer an exact transfer
      transaction without transaction-level evidence; never fabricate a tx
      signature from polling (spec §10.2, §9.2).
- [ ] `NftOwnershipHistory` entity + table distinguishing observed changes from
      confirmed transfers (spec §9).

### 4.3 Domain events (`Forgotten.Nft.Domain/Events`)
- [ ] Define the full set (spec §10.3): `NftVerifiedEvent`,
      `NftCollectionApprovedEvent`, `WalletConnectedEvent`,
      `WalletDisconnectedEvent`, `NftOwnershipChangedEvent`,
      `NftEligibilityChangedEvent`.
- [ ] ⚠ Make events idempotent where appropriate (spec §10.3, §858).
- [ ] Wire earlier phases to raise their events (wallet connect/disconnect from
      Phase 3; collection approved from Phase 2; verified from Phase 1/2).

### 4.4 Role integration (`Forgotten.Nft.Integrations/DiscordRoles`)
- [ ] Per-guild role config + collection→role mapping (spec §11).
- [ ] Assign role when a member owns a qualifying NFT from an approved
      collection; remove when they no longer do (spec §11).
- [ ] ⚠ Use **fresh** ownership verification before granting/removing (spec §11, §15).
- [ ] Background reconciliation loop (spec §11).
- [ ] ⚠ Configurable **grace period**: do **not** strip roles merely because the
      provider is temporarily unavailable (spec §11, §16 fail-safe).
- [ ] ⚠ Respect Discord role hierarchy/permissions; never auto-assign privileged
      admin roles; permission checks + audit logging (spec §11, §16.9, §16.14).

### 4.5 Cache invalidation
- [ ] ⚠ On ownership change / admin action: explicitly invalidate affected
      ownership + display caches (spec §15). An expired ownership assertion must
      never authorize a sensitive op.

## 5. Deliverables

- Running (optional) monitoring worker honoring the config flags.
- Ownership history recorded with observed-vs-confirmed distinction.
- Full event set defined and raised; role assignment/removal with grace periods.

## 6. Tests required (spec §18 "Ownership")

- [ ] Correct owner → no spurious change event.
- [ ] Incorrect / changed owner → single change event + history row.
- [ ] Ownership transfer observed → snapshot updated; no fabricated tx signature.
- [ ] ⚠ Provider unavailable → no role removal; grace period respected; no false
      "ownership lost".
- [ ] Stale cache not used to authorize; invalidation fires on change.
- [ ] ⚠ Duplicate ownership events suppressed (idempotency).
- [ ] Role reconciliation respects hierarchy; refuses privileged roles.

## 7. Exit Gate — all must be checked to start Phase 6

- [x] Worker detects ownership changes each sweep (bounded batch of tracked assets
      in approved collections); unit-tested with a mocked provider.
- [x] Observed changes recorded with `Source=monitor-observed` and a **null**
      transaction signature — never invented (unit test).
- [x] No-change → no event/history (idempotent/duplicate-event prevention, tested).
- [x] Domain events defined (`NftOwnershipChanged/Verified/CollectionApproved/
      WalletConnected/WalletDisconnected/EligibilityChanged`) and published on
      ownership change, wallet connect/disconnect, and collection approval.
- [x] Roles granted/removed via a hierarchy- and privilege-guarded applier
      (refuses Administrator/Manage-Roles/Manage-Guild and roles above the bot);
      ⚠ **provider outage → no removal** (grace), unit-tested.
- [x] Ownership-change invalidates the asset's inspect cache.
- [x] `AddMonitoringTables` migration additive and applied on startup.
- [x] Build clean (0 warnings); Phase 5 tests green (107 total; +8 monitor/reconcile).
- [ ] ⏳ **Live role assignment** — needs the bot granted **Manage Roles** and its
      role positioned above the managed role in the server (a Discord setting). The
      applier is built and self-guards; verified via `/nft-admin role-map` +
      `role-sync` once permissions are set. Not faked (spec §25).
- [ ] Docs (`ARCHITECTURE.md` events, `SECURITY.md`) — pending.

### Verification summary (2026-10-21)

- `dotnet build Forgotten.slnx` → **0 warnings, 0 errors**.
- `dotnet test` → **107 passed / 0 failed**.
- `dotnet run` → `AddMonitoringTables` auto-applied; worker logs
  "Ownership monitoring enabled; interval 00:05:00"; `/nft-admin role-map|role-unmap|
  role-list|role-sync` registered; WalletWeb still boots (null role-service default).
