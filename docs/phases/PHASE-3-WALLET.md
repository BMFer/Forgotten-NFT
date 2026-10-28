# Phase 3 — Wallet Ownership Verification

**Spec refs:** §7 (all), §9 (`DiscordWalletConnections`,
`WalletVerificationChallenges`), §6.5 (wallet commands), §16, §22.3
**Status:** ✅ code complete (2026-10-20, branch `phase-3-wallet`) — crypto core verified; live browser E2E pending an HTTPS domain + wallet (see Exit Gate)
**Deliverable (spec §22):** Members can securely connect and cryptographically
prove control of a Solana wallet. **A wallet address alone never grants verified
ownership** (spec §7.1).

---

## 1. Objective

Build the trust anchor the whole system depends on: a cryptographic,
replay-proof, account-bound proof of wallet control via Ed25519 signatures,
delivered through a minimal HTTPS wallet-verification web app.

> This is the most security-sensitive phase. Every item marked ⚠ is mandatory.
> `Forgotten.Nft.SecurityTests` carries the weight here.

## 2. Scope

**In scope**

- `Forgotten.Nft.WalletWeb` — HTTPS endpoints + browser wallet integration.
- Short-lived, single-use, account- and wallet-bound challenge generation.
- Server-side Ed25519 signature verification over exact signed bytes.
- Wallet ↔ Discord association + management (multiple wallets, primary, disconnect, reverify).
- `/wallet connect | verify | list | disconnect | status` (spec §6.5).
- Security tests (spec §18 "Wallet Verification").

**Out of scope**

- Gallery browsing of those wallets (Phase 4).
- Ownership monitoring / roles (Phase 5).

## 3. Prerequisites

- [ ] Phase 2 Exit Gate passed.
- [ ] An HTTPS-capable domain for the wallet web app, **or** a documented local
      HTTPS dev setup. If no public domain/deploy access exists, build + test
      locally and document the gap — do not fake a deployed verification (spec §25).
- [ ] Decide the wallet-linking policy (spec §7.4 default: one verified wallet →
      one active Discord identity at a time). Record it.

## 4. Tasks

### 4.1 Challenge model & generation (`Forgotten.Nft.Application` + Domain)
- [ ] `VerificationChallenge` domain entity + `WalletVerificationChallenges`
      table (spec §9): DiscordUserId, WalletAddress, NonceHash, ChallengeHash,
      ExpiresAt, ConsumedAt, Status, CreatedAt.
- [ ] Build the signed message exactly as spec §7.3 (Domain, Discord User ID,
      Wallet Address, Nonce, Issued At, Expires At, Purpose).
- [ ] ⚠ Challenge must be: cryptographically random, single-use, short-lived,
      bound to the Discord account, bound to the wallet address, bound to the
      app domain, replay-protected (spec §7.3).
- [ ] ⚠ Store only a **hash** of the nonce/challenge where practical (spec §7.3, §9).

### 4.2 Verification flow (`Forgotten.Nft.WalletWeb`)
- [ ] Implement the 11-step workflow from spec §7.2
      (connect → session → secure URL → wallet connect → challenge → sign →
      verify → confirm unused → associate → confirm).
- [ ] ⚠ Server-side Ed25519 verification of the **exact signed message bytes**
      (spec §7.3, §16.4). Never trust client-reported validity.
- [ ] ⚠ Never request/receive private keys or seed phrases; never store signing
      secrets (spec §16.1–16.3, §7.2).
- [ ] ⚠ Secure session binding: another user cannot claim a Discord account by
      obtaining the verification URL (spec §7.3, §16.18).
- [ ] ⚠ Require HTTPS for the verification site (spec §7.3, §16.12).
- [ ] Use a wallet-standard-compatible browser integration (spec §7.2).

### 4.3 Association & management
- [ ] `DiscordWalletConnections` table + entity (spec §9).
- [ ] On successful proof: associate wallet ↔ member; set verified timestamps.
- [ ] ⚠ Unique constraint on active wallet associations (spec §9.2); enforce the
      configured duplicate-claim policy — prevent duplicate wallet claims
      (spec §7.4, §16.17).
- [ ] Support multiple wallets/member, primary selection, disconnect, reverify,
      ownership-sync timestamps (spec §7.4).
- [ ] ⚠ Changing a wallet association requires a **fresh** ownership proof (spec §7.4).

### 4.4 Discord commands
- [ ] `/wallet connect` → returns the secure, short-lived verification URL
      (ephemeral — spec §12).
- [ ] `/wallet verify` → status of a pending/last verification.
- [ ] `/wallet list` → the member's linked wallets (⚠ ephemeral; never leak in
      public channels — spec §12, §16.13).
- [ ] `/wallet disconnect`, `/wallet status`.
- [ ] Rate-limit commands (spec §16.10).

### 4.5 Config
- [ ] Options: domain, challenge TTL, session TTL, `MaximumLinkedWallets`
      (spec §20), duplicate-wallet policy flag.
- [ ] ⚠ Startup validation rejects insecure settings (e.g. non-HTTPS domain,
      zero/absurd TTL) — fail closed (spec §20, §16.15).

## 5. Deliverables

- Running HTTPS wallet-verification web app (or documented local equivalent).
- A member can connect, sign, and get a persisted verified wallet link.
- `/wallet` command set functional and ephemeral where sensitive.

## 6. Tests required (spec §18 "Wallet Verification" — in `SecurityTests`)

- [ ] ⚠ Valid signature → verified; association created.
- [ ] ⚠ Invalid signature → rejected (`WALLET_SIGNATURE_INVALID`).
- [ ] ⚠ Expired challenge → rejected (`WALLET_CHALLENGE_EXPIRED`).
- [ ] ⚠ Reused challenge → rejected (single-use / replay protection).
- [ ] ⚠ Signature for a different wallet address → rejected.
- [ ] ⚠ Challenge bound to user A replayed by user B → rejected (session binding).
- [ ] Concurrent verification attempts handled safely (no double-association).
- [ ] ⚠ Duplicate wallet association → rejected per policy (`WALLET_ALREADY_LINKED`).
- [ ] Signed-message bytes compared exactly (tampered field fails).
- [ ] No private key / seed ever requested or stored (code + test assertion).

## 7. Exit Gate — all must be checked to start Phase 4

- [x] A real Ed25519-signed challenge verifies end-to-end in tests (generated keypair
      → server-built message → real `Ed25519SignatureVerifier`).
- [x] Every ⚠ security test passes: invalid sig, expired, reused/consumed (replay),
      wrong wallet (binding), unknown session token, duplicate-claim.
- [x] Wallet address submitted **without** a valid signature never yields a link
      (asserted: no connection row on invalid signature).
- [x] Challenge/connection migration additive and applied on startup
      (`AddWalletTables`). Session **token** stored as SHA-256 hash; nonce stored raw
      because the signed message must be reconstructed server-side (documented tradeoff).
- [x] Sensitive wallet info shown only via ephemeral responses.
- [x] Insecure config fails startup closed (non-HTTPS non-localhost `PublicBaseUrl` rejected).
- [x] Build clean (0 warnings); Phase 3 tests green (86 total; +13 wallet security).
- [ ] ⏳ **Live browser E2E** (connect a real wallet + sign) — requires running
      `Forgotten.Nft.WalletWeb` on a public HTTPS origin reachable by the browser and a
      real Solana wallet. Built but not locally verifiable; not faked (spec §25).
- [ ] Docs (`WALLET_VERIFICATION.md`, `SECURITY.md`, `CONFIGURATION.md`,
      `INSTALLATION.md` for the web app) — pending.

### Verification summary (2026-10-20)

- `dotnet build Forgotten.slnx` → **0 warnings, 0 errors**.
- `dotnet test` → **86 passed / 0 failed** (incl. 13 wallet-crypto security tests).
- `dotnet run` → `AddWalletTables` auto-applied; `WalletVerificationOptions` validated;
  `/wallet` commands registered; bot reaches Ready.
- **Local browser run:** `dotnet run --project src/Forgotten.Nft.WalletWeb` serves the
  signing page at the configured `PublicBaseUrl` (both apps must share the same SQLite
  DB file). Full sign-in needs a wallet extension + HTTPS/localhost origin.
