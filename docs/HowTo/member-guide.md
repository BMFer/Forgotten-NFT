# Member Guide

Everything you can do as a community member. All of these reply **privately** (only you see them).

- [1. Verify your wallet](#1-verify-your-wallet)
- [2. Show off your NFTs](#2-show-off-your-nfts)
- [3. Get your holder role](#3-get-your-holder-role)
- [4. Stake for loyalty points](#4-stake-for-loyalty-points)
- [5. Spend your points](#5-spend-your-points)

---

## 1. Verify your wallet

Proving you own a wallet is **cryptographic** — you sign a message, you never send a transaction, move
funds, or share your keys.

```text
/wallet connect
```

The bot sends you a **private, single-use link** that expires shortly. Open it, connect a Solana wallet
(Phantom / Solflare), and click **Sign to verify**.

> **Prefer a button?** If the server posted a wallet panel, just click **🔐 Connect wallet** on it — it
> does exactly the same thing as `/wallet connect` and sends you the same private link.

- **On a phone?** The page offers **Open in Phantom / Open in Solflare** — tap that to open inside the
  wallet app's browser, where signing works reliably. (A plain mobile browser can't sign.)
- Manage wallets anytime:

```text
/wallet list              # your connected wallets
/wallet status            # verification status + primary wallet
/wallet primary <addr>    # choose your primary wallet
/wallet disconnect <addr> # remove a wallet
```

> Trouble connecting on desktop (e.g. "Unexpected error")? It's almost always **another wallet
> extension** fighting Phantom — disable the others (leave only Phantom), make sure it's unlocked, and
> reload. The error message now shows a code to help diagnose.

## 2. Show off your NFTs

```text
/nft showcase             # browse your NFTs one at a time (prev / next / inspect / 📢 flex)
/nft inspect <address>    # full details for any NFT: art, owner, traits, status
/nft leaderboard          # top 10 verified holders of this server's collection
/nft help                 # step-by-step guide
```

In `/nft showcase`, the **📢 Flex** button posts the current NFT publicly (with your avatar and a
✅ Verified-holder badge). Use the dropdown to switch collections or see **all** your NFTs.

## 3. Get your holder role

If the server maps a role to a collection, verifying a wallet that holds it **automatically** grants
the role — no extra step. If you later transfer the NFT away, the role is removed on the next check
(the bot re-verifies on-chain; a brief provider outage never strips your role).

## 4. Stake for loyalty points

If the server runs a staking program, eligible NFTs show a **🔒 Stake** button in `/nft showcase`.

1. Open `/nft showcase` on an NFT from a staked collection.
2. Click **🔒 Stake**. The bot re-checks that you own it, records the stake, and the button flips to
   **🔓 Unstake**.

**Staking is non-custodial** — your NFT never leaves your wallet. "Staking" just records that you've
opted in, so you earn the program's **loyalty points daily**. You can **🔓 Unstake** anytime. If you
**transfer** a staked NFT out of your wallet, the stake ends automatically and a note posts to the
server's tracking channel.

Check your points:

```text
/rewards
```

shows your **total** balance, a **per-program** breakdown, and the **reward catalog** you can spend on.

> Points accrue once per **complete day** a stake is held (after any minimum-hold the program sets), so
> your first points appear a day or so after you stake.

## 5. Spend your points

```text
/redeem <id>
```

Use the reward id shown in `/rewards`. Redeeming **deducts the points immediately** and files a request
the **team fulfills manually** (e.g. granting a whitelist spot). Example:

```text
/rewards      → "🎁 Rewards you can redeem — #1 Whitelist spot — 50 pts"
/redeem 1     → "Redeemed Whitelist spot for 50 points. Your request (#4) is pending fulfillment."
```

If a reward is sold out or you're short on points, the bot tells you exactly why. Loyalty points are
**off-chain** — there are no token payouts or financial guarantees.
