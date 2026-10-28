# Phase 2 — Verification & Collection Registry

**Spec refs:** §4.2, §4.3, §6.3, §6.4, §6.5 (admin), §13, §9 (`NftCollections`,
`NftVerificationLogs`), §17, §22.2
**Status:** ✅ code complete (2026-10-20, branch `phase-2-verification`) — live `/nft verify` ready; approve a collection to see L4
**Deliverable (spec §22):** `/nft verify` distinguishes NFT validity, collection
authenticity (on-chain), and administrative approval (registry) — three separate
facts that are never conflated.

---

## 1. Objective

Turn a raw inspected asset into a layered verification result, and give
administrators a per-guild registry to *approve* collections. The spec is
emphatic (§4.2, §13): **on-chain collection verification ≠ registry approval ≠
ownership.** Keep them distinct in data, logic, and UI.

## 2. Scope

**In scope**

- Verification levels 1–4 (Asset Exists → NFT Classification → Collection
  Verified → Approved Collection) from spec §4.2. Levels 5–6 (ownership) are
  prepared in the result model but only fully evaluated after Phase 3; Level 7
  (eligibility) is Phase 6.
- `NftVerificationResult` object (spec §4.3 full field set).
- Collection approval registry, guild-scoped (spec §13).
- Admin commands (spec §6.5 admin subset).
- Verification audit logging (spec §9 `NftVerificationLogs`).
- `/nft verify` and `/nft collection` commands (spec §6.3, §6.4).

**Out of scope**

- Member ownership proof (needs Phase 3 wallets) — result exposes the field but
  reports it unevaluated until then.
- Role assignment driven by approval (Phase 5).

## 3. Prerequisites

- [ ] Phase 1 Exit Gate passed (inspection + provider + persistence working).

## 4. Tasks

### 4.1 Verification engine (`Forgotten.Nft.Application`)
- [ ] `INftVerificationService` producing the layered result.
- [ ] Implement levels (spec §4.2):
  - [ ] **L1 Asset Exists** — retrievable from provider.
  - [ ] **L2 NFT Classification** — supported NFT vs fungible/unsupported.
  - [ ] **L3 Collection Verified** — derive from trusted on-chain grouping /
        creator verification, **not** metadata claims (spec §8.2, §6.1).
  - [ ] **L4 Approved Collection** — present in this guild's approved registry.
  - [ ] L5/L6 fields present but reported `Indeterminate`/not-evaluated here.
- [ ] Build `NftVerificationResult` with the exact §4.3 fields (AssetExists,
      IsSupportedNft, CollectionVerified, CollectionApproved, OwnerAddress,
      OwnershipVerified, MemberOwnershipVerified, VerificationStatus,
      VerificationTimestamp, Provider, ErrorCode, ErrorMessage).
- [ ] ⚠ Keep levels independent: a valid NFT is not auto-approved and not
      auto-owned (spec §4.2). Enforce in code, not just docs.
- [ ] ⚠ Verification must use sufficiently fresh data; do not serve an ownership
      claim from a stale cache (spec §6.4, §15).

### 4.2 Collection registry (`Forgotten.Nft.Application` + Infrastructure)
- [ ] `NftCollection` entity extended with approval fields (IsApproved,
      ApprovedByDiscordUserId, ApprovedAt) per spec §9.
- [ ] `ICollectionRegistryService`: add / remove / list approved collections,
      **scoped per guild** (spec §13).
- [ ] Registry entry records approving admin + timestamp (spec §13).
- [ ] ⚠ Registry approval stored and surfaced separately from on-chain
      `CollectionVerified` (spec §13: "must not be confused").

### 4.3 Persistence & audit
- [ ] Migration: `NftCollections` (unique on blockchain+address per guild scope)
      and `NftVerificationLogs`.
- [ ] Write a `NftVerificationLog` row per verification (type, status, provider,
      checkedAt, errorCode) — for `/nft-admin verification-log` (spec §9, §6.5).

### 4.4 Admin commands (`Forgotten.Bot/Commands/NftAdminCommands.cs`)
- [ ] `/nft-admin collection-add`, `collection-remove`, `collection-list`.
- [ ] `/nft-admin provider-status` (uses Phase 1 provider health — spec §5.3).
- [ ] `/nft-admin cache-clear` (explicit invalidation — spec §15).
- [ ] `/nft-admin verification-log`.
- [ ] ⚠ Enforce guild-specific administrator permissions on **every** admin
      command; fail closed (spec §6.5, §16.9, §16.15). Never auto-grant admin.

### 4.5 User commands
- [ ] `/nft verify <asset_address>` rendering the §6.4 layout (Asset Exists /
      Supported NFT / Collection Verified / Collection Approved / Current Owner /
      Connected Wallet Match / Member Ownership). Wallet/member lines show
      "not connected / not evaluated" until Phase 3.
- [ ] `/nft collection <collection_address>` (spec §6.3): identifier, approved
      registry name if any, verification status, available metadata, paginated
      NFTs via `GetAssetsByCollectionAsync` (implement `getAssetsByGroup` now).
- [ ] ⚠ Do not invent a total supply the provider didn't give (spec §6.3).
      Bound pagination; no full-collection scans (spec §5.3).

## 5. Deliverables

- `/nft verify` showing four clearly separated truth values.
- Working per-guild approval registry + admin management commands.
- Verification audit log queryable by admins.

## 6. Tests required (spec §18 "NFT Validation" continued)

- [ ] Unverified collection → `CollectionVerified = NO`.
- [ ] On-chain-verified but **not** registry-approved → Verified=YES, Approved=NO.
- [ ] Registry-approved collection → Approved=YES (and verified flag independent).
- [ ] Fungible/unsupported token never reports collection verification.
- [ ] ⚠ Metadata-claimed collection membership does **not** set `CollectionVerified`.
- [ ] Admin command invoked by non-admin → rejected (`INSUFFICIENT_PERMISSIONS`), fails closed.
- [ ] Registry entries are guild-scoped (guild A approval invisible to guild B).
- [ ] `/nft collection` pagination is bounded; no fabricated supply.
- [ ] Verification writes an audit log row with correct status/provider.

## 7. Exit Gate — all must be checked to start Phase 3

- [x] `/nft verify` separates validity, on-chain verification, and approval — all
      independently correct in tests (independence verified both directions).
- [x] Admin registry commands enforce per-guild admin perms and fail closed
      (hard `RequireUserPermission(Administrator)` + `DefaultMemberPermissions`
      gate + guild-only; guarded by a reflection test).
- [x] `NftCollections` + `NftVerificationLogs` migration additive and applied
      (`AddCollectionsAndLogs`).
- [x] Audit log written on every verify; viewable via `/nft-admin verification-log`.
- [x] Build clean (0 warnings); Phase 2 tests green (73 total: 43 unit, 23 security, 7 integration).
- [ ] Docs (`DISCORD_COMMANDS.md`, `SECURITY.md`, `DATABASE_SCHEMA.md`) — pending.

### Verification summary (2026-10-20)

- `dotnet build Forgotten.slnx` → **0 warnings, 0 errors**.
- `dotnet test` → **73 passed / 0 failed**.
- `dotnet run` → bot registers `/nft` (now incl. verify + collection) and the
  `/nft-admin` group to the dev guild; reaches Ready. Live `/nft verify` works
  against the configured DAS endpoint.
