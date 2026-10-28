# Phase 1 — Foundation (NFT Inspection)

**Spec refs:** §4.1, §5 (all), §8, §9 (relevant tables), §15, §17, §19, §22.1
**Status:** ✅ code complete (2026-10-19, branch `phase-1-foundation`) — live DAS inspection pending a configured endpoint (see Exit Gate)
**Deliverable (spec §22):** `/nft inspect <asset_address>` works against a
configured Solana DAS provider — retrieving, normalizing, and displaying a single
NFT with artwork, ownership, and collection info.

---

## 1. Objective

Stand up the vertical slice that turns an asset address into a rendered Discord
embed: domain models → DAS provider → metadata resolution → inspection service →
thin command → embed. This is the backbone every later phase reuses.

## 2. Scope

**In scope**

- Core domain entities, enums, value objects (the subset needed to describe an asset).
- `ISolanaAssetProvider` + a concrete DAS adapter (`getAsset` at minimum).
- Provider resilience, configuration, and response normalization (spec §5.3–5.4).
- `INftInspectionService` returning `NftInspectionResult` (spec §4.1).
- Metadata resolution + security hardening (spec §8).
- `NftAssets` / `NftAttributes` persistence + caching (spec §9, §15).
- `/nft inspect`, `/nft attributes`, `/nft refresh`, `/nft help` (inspect-related subset of §6.5).
- `NftEmbedFactory` (spec §6.1, §21).
- Unit tests with mocked provider + metadata (spec §18).

**Out of scope**

- Verification *levels*, approval registry, and `/nft verify` (Phase 2).
- Wallets, galleries, monitoring, roles, game integration (Phases 3–6).
- `getAssetsByOwner` / `getAssetsByGroup` beyond interface definition (Phase 2/4).

## 3. Prerequisites

- [ ] Phase 0 Exit Gate passed.
- [ ] A DAS-compatible RPC endpoint + API key available **or** a documented plan
      to run with mocked responses only (spec §25 final directive — do not fake
      live results).

## 4. Tasks

### 4.1 Domain (`Forgotten.Nft.Domain`)
- [ ] `NftAsset` entity with fields from spec §9 `NftAssets` table + the
      inspection fields of §4.1 (interface/type, supply/standard, creators +
      verification, compression, burn, mutability).
- [ ] `NftAttribute` (TraitType / TraitValue / DisplayType).
- [ ] `NftCollection` identity fields (address, verified flags) — approval flags
      come in Phase 2.
- [ ] Value objects: `AssetAddress`, `WalletAddress`, `CollectionAddress`
      (validate Solana base58 on construction — spec §16.6).
- [ ] `NftVerificationStatus` enum (full set from §4.3, even though only a subset
      is produced this phase).
- [ ] DTO/result `NftInspectionResult` (Application layer) holding normalized data + source provider + timestamp.

### 4.2 Blockchain provider (`Forgotten.Nft.Infrastructure/Blockchain`)
- [ ] Define `ISolanaAssetProvider` (spec §5.1) — all three methods, but only
      `GetAssetAsync` must be fully implemented this phase; the others may return
      `NotImplemented`/throw a documented not-supported error until Phase 2/4.
- [ ] Implement a DAS adapter using `getAsset`. Map DAS JSON → `NftAsset`.
- [ ] ⚠ Validate every DAS response before use (spec §3, §5.3). Classify
      unsupported assets explicitly rather than treating them as valid (spec §4.1).
- [ ] Resilience (spec §5.3): configurable timeout, exponential backoff + jitter,
      connection pooling, cancellation, rate-limit handling, concurrency cap.
      **Retry only transient failures; never retry an invalid address.**
- [ ] ⚠ A provider timeout / RPC failure must surface as `ProviderUnavailable`,
      **never** as `NotFound` (spec §4.3, §16).
- [ ] `SolanaOptions` fully fleshed out per spec §5.2 JSON (DasEndpoint,
      RpcEndpoint, timeouts, retries, page sizes, cache duration). Keys/secrets
      from secure config only.
- [ ] Separate DAS endpoint config from plain RPC endpoint (spec §5.1).

### 4.3 Metadata resolution (`Forgotten.Nft.Infrastructure/Metadata`)
- [ ] Resolver supporting HTTPS, IPFS, Arweave via configurable gateways (spec §8.1).
- [ ] Resolve the standard field set (name, symbol, description, image,
      animation_url, external_url, attributes, properties); preserve extras.
- [ ] ⚠ Treat metadata as untrusted (spec §8.2): max response size, timeouts,
      content-type + URL-scheme validation, redirect limits, HTML/Markdown
      escaping, Discord mention suppression, length limits.
- [ ] ⚠ **SSRF protection** (spec §8.2, §16.8): reject loopback/private/
      link-local/cloud-metadata destinations; **revalidate resolved IPs across
      redirects**; never execute embedded scripts/HTML.
- [ ] Tolerate missing image/description/creator/attributes (spec §5.4, §8.1).

### 4.4 Inspection service (`Forgotten.Nft.Application/Services`)
- [ ] Implement `INftInspectionService.InspectAsync` (spec §4.1) orchestrating:
      address validation → provider fetch → metadata resolve/normalize → persist
      snapshot → build `NftInspectionResult`.
- [ ] Store source provider + `LastFetchedAt` on persisted assets (spec §5.4, §9).

### 4.5 Persistence & caching
- [ ] EF Core migration adding `NftAssets` + `NftAttributes` (unique index on
      blockchain asset identifier; FKs; UTC; parameterized — spec §9.2).
- [ ] `IMemoryCache` policies: NFT metadata 5 min, attributes 5 min (spec §15).
      Caching must never authorize sensitive ops (no ownership proof here).

### 4.6 Discord presentation (`Forgotten.Bot`)
- [ ] `/nft inspect <asset_address>`: validate format → **defer** interaction
      (spec §19) → inspect → build embed → reply.
- [ ] `NftEmbedFactory` renders name, collection, owner, attributes, status,
      artwork, and explorer links (spec §6.1). Placeholder image when artwork
      missing/unsupported; animated media degrades gracefully (spec §8.3).
- [ ] ⚠ Never assert collection verification based on name/image alone (spec §6.1).
      This phase shows "collection verification: unknown/not-evaluated" — real
      verification is Phase 2.
- [ ] `/nft attributes`, `/nft refresh` (force cache bypass + re-fetch), `/nft help`.
- [ ] Rate-limit commands (spec §16.10); keep handlers thin (spec §3).

## 5. Deliverables

- A working `/nft inspect` against a configured DAS provider (or mocked, documented).
- `NftAssets` + `NftAttributes` tables via a clean migration.
- `NftEmbedFactory` producing the §6.1 layout.

## 6. Tests required (spec §18 "NFT Validation" + "Security" subset)

- [ ] Valid NFT → correct normalized result + embed fields.
- [ ] Invalid address → `INVALID_ASSET_ADDRESS`, no provider call.
- [ ] Missing asset → `NotFound`.
- [ ] Burned asset → `Burned` status surfaced.
- [ ] Compressed NFT handled when provider supports it.
- [ ] Unsupported/fungible token → `Unsupported`, not "valid NFT".
- [ ] Malformed metadata handled without crashing.
- [ ] ⚠ Provider timeout → `ProviderUnavailable` (never `NotFound`).
- [ ] ⚠ SSRF: metadata URL pointing at loopback/private/metadata IP is rejected,
      including via a redirect.
- [ ] ⚠ Metadata with Discord mentions / HTML is escaped/suppressed in the embed.
- [ ] All tests use mocked provider + metadata; no live chain (spec §18).

## 7. Exit Gate — all must be checked to start Phase 2

- [x] Live DAS path verified against a real endpoint (2026-10-19): `getAsset` →
      validate → normalize → classify. Wrapped SOL returned `AssetExists=true`,
      `InterfaceType=FungibleToken`, `Status=Unsupported` — correct live
      classification. *(Still to render: a full artwork embed for a real NFT mint —
      the `/nft inspect` Discord command is registered and the bot is live for it.)*
- [x] Unsupported / missing / burned assets are classified correctly and distinctly. *(Unit tests.)*
- [x] Provider failures never read as "NFT does not exist" → `ProviderUnavailable`. *(Unit test.)*
- [x] SSRF + metadata-sanitization tests pass. *(23 security tests incl. metadata IP + redirect scheme guard; embed markdown escaping.)*
- [x] Migration applies to a fresh DB and is additive. *(`AddNftAssets` migration; Phase 0 `DatabaseMigrationTests` cover apply.)*
- [x] Build clean (0 warnings); all Phase 1 tests green (56 total: 29 unit, 23 security, 4 integration).
- [ ] Docs (`SOLANA_INTEGRATION.md`, `DISCORD_COMMANDS.md`, `CONFIGURATION.md`) — to be written before closing the phase.

### Verification summary (2026-10-19)

- `dotnet build Forgotten.slnx` → **0 warnings, 0 errors**.
- `dotnet test` → **56 passed / 0 failed**.
- `dotnet run` → bot connects, registers the `/nft` command group to the dev guild,
  reaches Ready. Banner reports `Solana.DasConfigured=False` — live inspection
  returns `BLOCKCHAIN_PROVIDER_UNAVAILABLE` until an endpoint is set.
