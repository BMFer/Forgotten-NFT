# Admin Guide

Setup and operations for server administrators. Every `/nft-admin …` command is **Administrator-only,
guild-only, and ephemeral**. Examples use the FORGOTTEN collection
`grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA` — swap in yours.

- [1. Approve your collection](#1-approve-your-collection)
- [2. NFT-gated roles](#2-nft-gated-roles)
- [3. Near-real-time updates (Helius webhook)](#3-near-real-time-updates-helius-webhook)
- [4. The #nft-tracks feed](#4-the-nft-tracks-feed)
- [5. Loyalty staking](#5-loyalty-staking)
- [6. Rewards & redemptions](#6-rewards--redemptions)
- [7. Per-domain branding](#7-per-domain-branding)

---

## 1. Approve your collection

The engine is collection-agnostic — tell it which collections count for this server:

```text
/nft-admin collection-add grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA FORGOTTEN
/nft-admin collection-default grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA   # what /nft showcase shows by default
/nft-admin collection-list        # ⭐ marks the default
/nft-admin collection-remove <address>
```

The **first** approved collection becomes the showcase default automatically.

**Give members a one-click way to verify.** Post a persistent panel with a **Connect wallet** button to
your onboarding channel:

```text
/nft-admin wallet-panel #verify-here   # omit the channel to post in the current one
```

Anyone who clicks the button gets their **own private** verification link (the same flow as
`/wallet connect`) — no transaction, no funds moved, keys never shared. The button keeps working across
bot restarts, so post it once and leave it. (The bot needs **Send Messages** in the target channel.)

**See who verifies.** Set `Discord__VerifyLogChannelId` to a channel id (server config) and the bot
posts a **✅ Wallet Verified** embed there each time a member verifies — their mention + avatar, the
short wallet address, and how many of your default collection they hold. It posts within ~30s of
verification and never re-announces on restart. (Requires `Discord__DevGuildId` set too.)

## 2. NFT-gated roles

Grant a Discord role to holders of an approved collection:

```text
/nft-admin role-map grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA @Holder
/nft-admin role-list
/nft-admin role-sync          # reconcile everyone against current ownership now
/nft-admin role-unmap <collection>
```

**Requirement:** the bot needs **Manage Roles**, and its own role must sit **above** the role it
grants in Server Settings → Roles. Safeguards: privileged roles (Administrator / Manage Roles / Manage
Guild) are refused, and a provider outage never strips roles.

## 3. Near-real-time updates (Helius webhook)

By default roles reconcile on a background poll. For updates **within seconds** of a transfer:

```text
/nft-admin webhook-sync
```

This registers one Helius webhook watching every verified wallet (as a **raw** webhook, so MPL Core
transfers are delivered). **Re-run it whenever the set of verified wallets changes.** Confirm with
`/nft-admin provider-status`. Prereqs (server-side): `Helius__WebhookSecret` set, a Helius API key, and
a public HTTPS site URL — see [`../DEPLOYMENT.md`](../DEPLOYMENT.md) §7.

## 4. The #nft-tracks feed

Set `Discord__NftTracksChannelId` to a channel id (server config) and the bot posts a detailed embed
when a verified member's NFT moves — and for every stake / unstake / transfer-ended stake. Transfers
of NFTs **outside** your approved collections are recorded but **not** posted. Links go to Solscan.

## 5. Loyalty staking

Staking is **off** until you turn on a global switch **and** enable a per-collection program.

1. **Global switch** (server-side): set `NftEngine__EnableStaking=true` and restart. While off, no Stake
   button appears anywhere.
2. **Configure a program** for an approved collection (created **disabled**):

   ```text
   /nft-admin staking-configure  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA  rate-per-day: 10
   ```

   Optional: `min-hold-minutes:` (eligibility delay), `starts:` / `ends:` (UTC window, e.g.
   `2026-12-01`). **`rate-per-day` is what members earn** — without it, nothing accrues.
3. **Enable it:**

   ```text
   /nft-admin staking-enable   collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA
   /nft-admin staking-list     # 🟢 enabled / 🔴 disabled, rate, window
   /nft-admin staking-disable  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA
   ```

Now holders see **🔒 Stake** on eligible NFTs. Stakes are **non-custodial**, **records-only** (no
payouts), **one live stake per NFT**, and **auto-end on transfer** (re-verified on-chain, so an outage
never falsely ends a stake). Points accrue once per complete eligible UTC day.

## 6. Rewards & redemptions

Define what points buy, then fulfill requests. Rewards belong to a collection's program.

```text
/nft-admin reward-add  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA  name: "Whitelist spot"  cost: 50
/nft-admin reward-add  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA  name: "1-of-1 raffle entry"  cost: 100  stock: 25
/nft-admin reward-list            # `#id` 🟢 name — cost, stock
/nft-admin reward-remove <id>
```

Members redeem with `/redeem <id>` (points are deducted immediately). You fulfill:

```text
/nft-admin redemptions            # the pending queue: #id, who, reward, when
/nft-admin reward-fulfill <id> [note]   # after you deliver the perk
/nft-admin reward-cancel  <id> [note]   # cancel + refund the member's points (+ restore stock)
```

**FORGOTTEN never delivers or funds rewards** — your project fulfills each one. Only publish rewards you
can actually honor.

**See the whole picture** with one report:

```text
/nft-admin rewards-report
```

Per program (🟢/🔴): active stakes, points **issued / redeemed / outstanding**, and distinct earners —
plus guild-wide redemption counts (pending / fulfilled / cancelled). The footer flags if staking is
globally off.

## 7. Per-domain branding

One deployment can serve the verification site under several hostnames, each with its own brand. Add the
DNS + Caddy alias (see [`../DEPLOYMENT.md`](../DEPLOYMENT.md) §3), then add a tenant to the `Branding`
config, binding a host to a Discord guild:

```jsonc
"Branding": {
  "Tenants": [{
    "Host": "verify.proofofmint.io",
    "GuildId": "1382087625229467748",
    "BrandName": "Proof of Mint",
    "Title": "Proof of Mint — Wallet Verification",
    "AccentColor": "#2fe3a2",
    "BannerImage": "/img/proofofmint/banner.png"
  }]
}
```

Members of that guild then get a `/wallet connect` link on their own branded domain. It's
**branding + attribution only** — wallet ownership stays global and the signed message binds to the
canonical domain. See [`../ARCHITECTURE.md`](../ARCHITECTURE.md#multi-tenant-branding--guild-attribution).
