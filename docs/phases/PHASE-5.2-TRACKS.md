# Phase 5.2 — Transfer tracking (#nft-tracks announcer)

**Spec refs:** extends §22.5 / Phase 5.1 (§10 monitoring, §11 roles); §8.2 (sanitize untrusted metadata)
**Status:** ✅ code complete + unit-tested + **live-verified** in production (2026-10-29)
**Deliverable:** When a Helius webhook fires for a verified member's wallet, post an embed to a
configured channel (`#nft-tracks`) describing the member, the identified NFT, the wallet, and a link
to the transaction.

> Builds directly on [Phase 5.1](PHASE-5.1-WEBHOOKS.md): the drain worker already re-verifies
> ownership on every webhook event — this phase adds a human-visible announcement alongside it.

---

## 1. Objective

Give the community a live feed of NFT movement for verified members. The embed is a side-effect of
the existing drain cycle; it never changes role logic and never fails the reconciliation.

## 2. Scope

**In scope**

- `Discord:NftTracksChannelId` config (requires `Discord:DevGuildId`), off by default.
- Best-effort NFT identification from the raw webhook's account keys via DAS.
- An embed with the member, NFT (name/image/collection), matched wallet, and a transaction link.

**Out of scope**

- Changing what triggers reconciliation (unchanged from Phase 5.1).
- Exhaustive transaction decoding — the moved NFT is resolved best-effort and may be "unidentified".
- Staking state transitions (Staking §S2).

## 3. Prerequisites

- [x] Phase 5.1 live (webhook delivering raw events to the drain worker).
- [ ] `Discord:NftTracksChannelId` set to the target channel; `Discord:DevGuildId` set.

## 4. Tasks

### 4.1 Config (`DiscordOptions`)
- [x] Add `NftTracksChannelId` (nullable). Unset = tracking disabled (no-op). Documented in
      `appsettings.Example.json`.

### 4.2 Candidate filter (`WebhookAddressFilter`, `Forgotten.Nft.Application`)
- [x] Pure, deterministic `SelectAssetCandidates(addresses, memberWallets, cap)` — drops the member's
      wallets and known program/sysvar accounts, de-dupes preserving order, caps the count. Lives in
      Application (not the bot) so it is unit-tested without the Discord stack.

### 4.3 Announcer (`WebhookTransferAnnouncer`, `Forgotten.Bot`)
- [x] Resolve the moved NFT best-effort: probe each candidate via
      `ISolanaAssetProvider.GetAssetAsync`, keep supported, non-burned NFTs; feature the one now
      owned by a member wallet (received) else the first found.
- [x] ⚠ Sanitize all asset-derived text (`Format.Sanitize`) and length-cap it before display
      (spec §8.2); send with `AllowedMentions.None`.
- [x] Build the embed: member (author + avatar), NFT (name, thumbnail, collection + explorer link),
      matched wallet, transaction link, "Ownership re-verified on-chain ✓".
- [x] No-op when disabled or when the guild/channel is unavailable.

### 4.4 Wire into the drain worker (`WebhookDrainWorker`)
- [x] Parse the event addresses once; after reconciling each affected member, announce the transfer.
- [x] ⚠ Announce is **failure-isolated** — a tracking error is logged and swallowed, never marking
      the inbox entry failed or blocking reconciliation.
- [x] Registered `WebhookTransferAnnouncer` as scoped (depends on scoped repositories/provider).

### 4.5 Docs
- [x] Config documented; this guide added; phase map + README roadmap updated.

## 5. Deliverables

- Optional, config-gated transfer feed to `#nft-tracks`.
- Best-effort NFT identification with sanitized, mention-safe embeds.
- No change to reconciliation safety or outcomes.

## 6. Tests

- [x] `WebhookAddressFilterTests` — drops wallets + known programs, de-dupes, honors cap, skips
      blanks, empty when nothing qualifies (5 tests).
- [x] Full suite green: **128 passed / 0 failed** (`dotnet test Forgotten.slnx`).
- [x] ✅ Live: embed fires on a real transfer (verified in production — see summary).

## 7. Exit Gate

- [x] `NftTracksChannelId` config added, validated path, documented, off by default.
- [x] Candidate filter pure + unit-tested; announcer sanitizes untrusted text and suppresses mentions.
- [x] Drain-worker integration is failure-isolated (tracking never breaks reconciliation).
- [x] Solution builds clean (0 warnings); full test suite green.
- [x] ✅ **Live end-to-end** — deployed, `Discord:NftTracksChannelId` set, embed posted on a real
      transfer (observed in production — not fabricated, spec §25).

### Verification summary (2026-10-29)

- `dotnet build Forgotten.slnx` → **0 warnings, 0 errors**.
- `dotnet test Forgotten.slnx` → **128 passed / 0 failed** (+5 `WebhookAddressFilterTests`).
- Deployed `main` to the EC2 (bot binary `02:50 UTC`); confirmed the running process loaded
  `Discord__NftTracksChannelId`.
- **Real transfer** on a watched wallet (tx `3eHU…Y7FJ`, inbox id 3) → the drainer logged
  `Announced transfer 3eHU…Y7FJ … (NFT: q6XB…MLut)` then `reconciled 1 member(s)` ~6s after
  delivery. The NFT was identified (not "unidentified") and the embed posted to `#nft-tracks`.

### Refinement — tracked-collection gating + holdings list (2026-10-29)

- **Gate by tracked collection:** only announce when the moved NFT belongs to a collection the guild
  has **approved** (`ICollectionRegistryService.ListApprovedAsync`). A transfer involving no tracked
  collection is still recorded in the `WebhookInbox` (the DB record) — it just isn't posted to the
  channel. The decision lives in the pure, unit-tested `WebhookAddressFilter.SelectTrackedNfts`.
- **"Still holds" list:** the embed now lists the NFTs the member currently holds from that
  collection (best-effort DAS paging, capped), so it shows **what changed** and **what remains**.
- ✅ **Live-verified in production (2026-10-30).** Both paths observed on real transfers to the
  watched wallet: a **non-tracked** collection NFT (inbox id 11) logged
  `no tracked-collection NFT; recorded in inbox, not announced` with **no** channel post; a
  **tracked** collection NFT (inbox id 12) logged `Announced transfer … (NFT: q6XB…MLut, holds 9)`
  and posted the embed with the "Still holds 9" list populated.
