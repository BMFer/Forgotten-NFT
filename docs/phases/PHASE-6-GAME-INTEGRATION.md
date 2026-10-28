# Phase 6 — External Game Integration (POW)

**Spec refs:** §14 (all), §4.2 Level 7, §10.3 (`NftEligibilityChangedEvent`),
§20 (`EnablePowIntegration`), §22.6, §26
**Status:** not started
**Deliverable (spec §22):** An external game (POW) can use verified NFT ownership
and eligibility **without depending directly on blockchain APIs** — through
clean, reusable application interfaces.

---

## 1. Objective

Expose the NFT engine's trust (ownership + eligibility) as stable interfaces an
external game consumes, and provide a POW adapter + NFT→fighter mapping seam. The
governing rule (spec §26, §14): **the core engine stays collection- and
game-agnostic**; POW / Dead Mint Carnival rules live outside it.

## 2. Scope

**In scope**

- `INftOwnershipService` + richer ownership result (spec §14.1).
- `INftEligibilityService` + `NftEligibilityResult` (spec §14.2) — Verification
  **Level 7** (spec §4.2).
- POW adapter interface + NFT→fighter mapping seam (spec §14.3).
- Ownership-change notifications surfaced to integrations (spec §14.3, §10.3).
- `NftEligibilityChangedEvent` emission (defined in Phase 5).

**Out of scope** — must **not** be built in the core engine (spec §14, §26)

- Combat mechanics / battle engine.
- Fighter persistence, levels, XP, magic attacks, win/loss records — those belong
  to the game integration, not `Forgotten.Nft.*`.

## 3. Prerequisites

- [ ] Phase 5 Exit Gate passed (fresh ownership + events available).
- [ ] `EnablePowIntegration` config flag (spec §20, default `false`).

## 4. Tasks

### 4.1 Ownership interface (`Forgotten.Nft.Application/Interfaces`)
- [ ] `INftOwnershipService.IsOwnedByWalletAsync(asset, wallet, ct)` (spec §14.1).
- [ ] ⚠ For security-sensitive callers, provide a **richer result type** that
      distinguishes ownership mismatch from provider failure (spec §14.1, §4.3).
      A boolean `false` must not conflate "not owned" with "couldn't check".

### 4.2 Eligibility interface (Level 7)
- [ ] `INftEligibilityService.CheckEligibilityAsync(discordUserId, asset,
      applicationId, ct)` → `NftEligibilityResult` (spec §14.2).
- [ ] Compose prior layers: member ownership (Phase 3) + approved collection
      (Phase 2) + any application-specific requirement for `applicationId`
      (spec §4.2 L7).
- [ ] ⚠ Fail closed: if authorization cannot be verified, deny (spec §16.15, §16.19).
- [ ] ⚠ Verify NFT ownership **before** any game authorization; never use cached
      metadata as proof of current ownership (spec §16.19, §16.20).

### 4.3 POW adapter (`Forgotten.Nft.Integrations/Pow`)
- [ ] Adapter interface mapping: Discord member → verified wallet → owned NFT →
      NFT↔fighter mapping → (external) fighter profile (spec §14.3 flow).
- [ ] ⚠ The engine provides ownership/eligibility only; it must **not** implement
      combat (spec §14.3, §26).
- [ ] Define the NFT→fighter mapping *seam* (interface) — the game owns the
      fighter records (`FighterId`, Level, XP, Wins, etc. — spec §14.4 belongs to
      the game, not here).
- [ ] ⚠ Ownership must be **rechecked before granting control** of a fighter
      (spec §14.4).
- [ ] Document the §14.4 guidance: on owner change, fighter progression stays with
      the NFT unless the game's explicit rules say otherwise — but that policy
      lives in the game, not the engine.

### 4.4 Notifications & events
- [ ] Surface `NftOwnershipChangedEvent` → integrations (spec §14.3, §10.3).
- [ ] Emit `NftEligibilityChangedEvent` when a member's eligibility for an
      application changes (spec §10.3).

### 4.5 Config & guardrails
- [ ] All POW wiring gated behind `EnablePowIntegration` (spec §20).
- [ ] ⚠ Architecture test: no `Forgotten.Nft.*` core project references POW /
      game-specific types (enforces spec §26 decoupling).

## 5. Deliverables

- Stable `INftOwnershipService` + `INftEligibilityService` with provider-failure-
  aware results.
- POW adapter + NFT→fighter mapping interfaces (no combat, no fighter storage).
- Eligibility-change notifications flowing to integrations.

## 6. Tests required

- [ ] `IsOwnedByWalletAsync`: owned → true; not owned → false; provider failure →
      **distinct** (not false) via the richer result.
- [ ] ⚠ Eligibility denies when ownership unverifiable (fail closed).
- [ ] ⚠ Eligibility requires member ownership AND approved collection AND app rule.
- [ ] ⚠ Ownership rechecked before fighter control is granted (stale cache rejected).
- [ ] `NftEligibilityChangedEvent` fires on eligibility transitions.
- [ ] Architecture test: core engine has no POW/game references.

## 7. Exit Gate — project acceptance (spec §23)

- [ ] External game can resolve ownership + eligibility with **zero** direct
      blockchain-API access.
- [ ] Ownership-vs-provider-failure is always distinguishable to callers.
- [ ] Core engine contains no POW / Dead Mint Carnival rules (verified by test).
- [ ] Ownership rechecked before game authorization; fails closed.
- [ ] Build clean; Phase 6 tests green; `POW_INTEGRATION.md` written.
- [ ] Revisit spec §23 acceptance criteria 1–20 across all phases; all satisfied.
