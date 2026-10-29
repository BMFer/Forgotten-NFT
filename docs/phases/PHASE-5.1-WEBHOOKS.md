# Phase 5.1 — Near-real-time reconciliation (Helius webhook)

**Spec refs:** extends §22.5 (§10 monitoring, §11 roles); §9 (`WebhookInbox` table);
relates to Staking §S2 (durable webhook inbox, built early here for the core engine)
**Status:** ✅ code complete **and live-verified** (2026-10-29) — see Verification summary
**Deliverable:** On-chain transfers touching a verified wallet reconcile that member's
NFT-gated roles within seconds, instead of waiting for the Phase 5 polling interval.

> **Retroactive guide.** This slice was built and shipped (PRs #35–#37) before a phase
> guide existed for it — the gap this document closes. The polling
> [Phase 5](PHASE-5-MONITORING.md) monitor remains the periodic backstop; this phase is
> the low-latency path layered on top of it, reusing the same reconciliation service.

---

## 1. Objective

Add a Helius webhook pipeline so ownership changes are reconciled in near-real-time.
Webhooks are treated as a **signal, not proof**: every delivery triggers a **fresh
on-chain re-verification** through the existing reconciliation path before any role
change — identical safety semantics to the Phase 5 poll, just triggered sooner.

## 2. Scope

**In scope**

- Admin command to register/refresh a single Helius webhook watching all verified wallets.
- Authenticated webhook receiver on `WalletWeb` + a durable, idempotent inbox (spec §9).
- A background drainer on the bot that re-verifies ownership and reconciles roles.
- Parsing both Helius payload shapes (enhanced **and** raw).

**Out of scope**

- Replacing the Phase 5 polling monitor (kept as the backstop).
- Staking state transitions driven by transfers (Staking §S2 consumes this inbox later).

## 3. Prerequisites

- [x] Phase 5 Exit Gate passed (reconciliation service + role applier exist).
- [x] `Helius:WebhookSecret` set on the server; a Helius API key available (via
      `Helius:ApiKey` or embedded in `Solana:DasEndpoint`).
- [x] Public HTTPS `WalletVerification:PublicBaseUrl` (the callback origin).

## 4. Tasks

### 4.1 Webhook registration (`HeliusWebhookRegistrar`, `Forgotten.Nft.Infrastructure`)
- [x] `/nft-admin webhook-sync` upserts one webhook **by callback URL** — find existing
      → `PUT`, else `POST` then `PUT` the full config. Re-running is safe (idempotent).
- [x] ⚠ **Create-then-PUT**: Helius's `POST` does not reliably persist
      `accountAddresses`, so watched wallets are applied with a follow-up `PUT` (fixed
      in #36 after observing `addrs = 0` live).
- [x] ⚠ **Raw, not enhanced** (`webhookType: "raw"`, `transactionTypes: ["Any"]`):
      Helius's enhanced parser types MPL Core (`MplCoreAsset`) transfers as `UNKNOWN`
      and never emits them — an enhanced webhook silently drops Core NFT transfers
      (confirmed live, #37).
- [x] API key resolves from `Helius:ApiKey`, falling back to the `api-key` in
      `Solana:DasEndpoint`.

### 4.2 Receiver (`WalletWeb` — `POST /api/helius/webhook`)
- [x] ⚠ Authenticate against `Helius:WebhookSecret`: `503` until the secret is set,
      `401` on mismatch. Fail closed.
- [x] Parse via `HeliusWebhookParser` (handles enhanced `signature` +
      `accountData`/`tokenTransfers` **and** raw `transaction.signatures` +
      `transaction.message.accountKeys`); malformed entries skipped, never throws.
- [x] Enqueue each event to the durable `WebhookInbox`, **idempotent on the transaction
      signature**; return fast (`{ enqueued }`). `AddWebhookInbox` migration is additive.

### 4.3 Drain worker (`WebhookDrainWorker`, `Forgotten.Bot`)
- [x] `BackgroundService` polling the inbox every 20s (25s startup grace; bounded to 5
      events/cycle; 600ms DAS pacing for free-tier friendliness).
- [x] Resolve touched addresses → verified wallet connections → affected Discord users.
- [x] ⚠ **Re-verify ownership on-chain** via `ReconcileMemberAsync` before any role
      change — never trust the webhook payload as proof.
- [x] Idempotent: re-processing an event just re-reconciles. `MarkProcessed` on success,
      `MarkFailed` (with retry) on error.

### 4.4 Docs
- [x] `ARCHITECTURE.md` — "Near-real-time ownership reconciliation" section + flow diagram.
- [x] `DEPLOYMENT.md` — webhook env vars + step 7 "Enable near-real-time updates".
- [x] `README.md` — webhook path noted under *Ownership monitoring & roles*.

## 5. Deliverables

- `/nft-admin webhook-sync` registering a raw Helius webhook over all verified wallets.
- Authenticated receiver + durable idempotent `WebhookInbox`.
- Drain worker that re-verifies on-chain and reconciles roles within seconds.
- Dual-shape parser with unit tests.

## 6. Tests required

- [x] Parser extracts signature + addresses from the **enhanced** shape.
- [x] Parser extracts signature + account keys from the **raw** shape (string and
      `{pubkey}` forms).
- [x] Malformed / shape-missing entries are skipped (no throw); signature-less tx ignored.
- [x] (Covered by Phase 5) Reconciliation re-verifies on-chain and respects the outage
      grace period — the webhook path reuses it unchanged.

## 7. Exit Gate

- [x] `webhook-sync` registers/refreshes exactly one raw webhook by callback URL;
      watched addresses persisted (verified via `GET /v0/webhooks/{id}`).
- [x] Receiver authenticates (`503`/`401`), parses both shapes, enqueues idempotently.
- [x] Drain worker re-verifies on-chain before role changes; idempotent; DAS-paced.
- [x] `AddWebhookInbox` migration additive and applied on startup.
- [x] `HeliusWebhookParserTests` green (6/6).
- [x] Docs updated (ARCHITECTURE / DEPLOYMENT / README).
- [x] ✅ **Live end-to-end verified** (not faked — spec §25) — see below.

### Verification summary (2026-10-29, production)

- `dotnet test --filter HeliusWebhookParser` → **6 passed / 0 failed**.
- Deployed `main` to the EC2 (bot binary `01:44 UTC`, both services `active`).
- `/nft-admin webhook-sync` → webhook flipped `enhanced → raw`, 2 verified wallets
  persisted (confirmed via the Helius API by webhook id).
- **Real MPL Core transfer** on a watched wallet → delivery enqueued to `WebhookInbox`
  (`Pending → Processed`); the account-key list included the MPL Core program
  (`CoRE…`), the exact case the old enhanced webhook dropped. Bot logged
  `Webhook event …: reconciled 1 member(s)` **~7s** after delivery. Previously: zero
  delivery for this transfer type.
