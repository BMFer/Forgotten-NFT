# How-To Guides

Practical, example-driven usage for the FORGOTTEN NFT engine. Pick your role:

- **[Member Guide](member-guide.md)** — verify your wallet, flex your NFTs, stake, and redeem rewards.
- **[Admin Guide](admin-guide.md)** — approve collections, map roles, enable near-real-time updates,
  configure branding, and run the loyalty-staking + rewards program.

For how the system is built, see [`../ARCHITECTURE.md`](../ARCHITECTURE.md); to deploy it, see
[`../DEPLOYMENT.md`](../DEPLOYMENT.md).

> Throughout, the example collection address `grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA` is the
> **FORGOTTEN** collection — replace it with your own. All `/nft-admin …` commands are
> **Administrator-only**; member commands are available to everyone and reply privately (ephemeral).

---

## 60-second end-to-end (loyalty staking)

**Admin, once:**

```text
/nft-admin collection-add grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA FORGOTTEN
/nft-admin wallet-panel #verify-here   # posts a Connect-wallet button members can click
/nft-admin staking-configure  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA  rate-per-day: 10
/nft-admin staking-enable     collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA
/nft-admin reward-add  collection: grEcjrsDT3eqMxcvR4XbwFWLrPSNTa2wumzHjvCKtvA  name: "Whitelist spot"  cost: 50
```

…and set `NftEngine__EnableStaking=true` on the server (one global switch).

**Member:**

```text
/wallet connect          → open the link, sign (no transaction)
/nft showcase            → open an NFT, click 🔒 Stake
/rewards                 → watch points accrue daily; see the reward catalog
/redeem 1                → spend points on a reward (the team fulfills it)
```

**Admin fulfills:**

```text
/nft-admin redemptions           → see pending requests
/nft-admin reward-fulfill 1      → mark it delivered
/nft-admin rewards-report        → program totals: stakes, points issued/redeemed, earners
```
