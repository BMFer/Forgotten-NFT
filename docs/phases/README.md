# FORGOTTEN NFT Engine — Phase Guides

This directory breaks the [technical specification](../specs.md) into sequenced,
self-contained build phases. Each phase guide is a working checklist: scope,
tasks, deliverables, tests, and an explicit **exit gate** that must pass before
the next phase begins.

## Why these guides exist

`specs.md` §22 already lists six implementation phases. These guides expand each
one into concrete, verifiable work, and add a **Phase 0** that the spec implies
but does not break out: the spec is written as if extending an *existing*
FORGOTTEN bot, but this repository is currently empty (only `docs/` exists).
Per spec §2.2 — *"If no existing implementation exists, create the architecture
specified below"* — we treat this as a greenfield build. Phase 0 stands up the
solution, host, DI, and a live Discord connection so that Phase 1 has something
real to extend.

> **If a pre-existing FORGOTTEN bot is later located**, do not run Phase 0's
> scaffolding blindly. Instead perform the architectural assessment described in
> spec §2.2 / §25 Step 1 and fold the NFT engine into the existing solution.

## Phase map

| Phase | Guide | Spec ref | Deliverable (one line) |
|------|-------|----------|------------------------|
| 0 | [PHASE-0-BOOTSTRAP.md](PHASE-0-BOOTSTRAP.md) | §21, §25 | Solution builds; bot logs into Discord; DI + config + logging + SQLite host all wired. |
| 1 | [PHASE-1-FOUNDATION.md](PHASE-1-FOUNDATION.md) | §22.1, §4.1, §5, §8 | `/nft inspect <addr>` works against a configured DAS provider. |
| 2 | [PHASE-2-VERIFICATION.md](PHASE-2-VERIFICATION.md) | §22.2, §4.2–4.3, §13 | `/nft verify` separates NFT validity, collection authenticity, and admin approval. |
| 3 | [PHASE-3-WALLET.md](PHASE-3-WALLET.md) | §22.3, §7 | Members cryptographically prove Solana wallet ownership. |
| 4 | [PHASE-4-GALLERY.md](PHASE-4-GALLERY.md) | §22.4, §6.2, §12 | Members browse owned NFTs in interactive Discord galleries. |
| 5 | [PHASE-5-MONITORING.md](PHASE-5-MONITORING.md) | §22.5, §10, §11 | Bot safely detects ownership changes and reconciles Discord roles. |
| 6 | [PHASE-6-GAME-INTEGRATION.md](PHASE-6-GAME-INTEGRATION.md) | §22.6, §14 | External games (POW) consume verified ownership/eligibility without touching chain APIs. |

## Sequencing rules (from spec §25, §22)

1. **One phase at a time.** Proceed to phase _N+1_ only after phase _N_ builds
   and passes its required tests (spec final directive).
2. **Vertical slices over broad stubs.** Prefer a complete, tested command over
   many half-finished classes (spec §25 Step 3).
3. **No fabricated results.** If provider credentials / domain / deploy access
   are missing, implement and test everything possible locally, document the
   missing prerequisite, and do not claim live verification succeeded
   (spec §25, final directive).
4. **Security-critical settings fail closed.** Invalid security config stops
   startup; it is never silently ignored (spec §16, §20).
5. **Keep the core engine collection-agnostic.** No Dead Mint Carnival or POW
   rules inside `Forgotten.Nft.*`; those live behind the integration interfaces
   (spec §26).

## Definition of Done (applies to every phase)

A phase is "done" only when **all** of these hold:

- [ ] Solution builds with no new errors; warnings reviewed.
- [ ] All unit tests for the phase pass; tests use mocked providers, no live chain (spec §18).
- [ ] EF Core migrations added this phase apply cleanly to a fresh DB **and** are additive (never drop/recreate an existing DB — spec §9).
- [ ] New Discord commands register and do not break existing commands.
- [ ] New configuration is validated at startup and documented.
- [ ] Relevant docs (spec §24) updated.
- [ ] The phase's **Exit Gate** checklist (bottom of each guide) is fully checked.

## Conventions used in the guides

- **Project names** follow spec §21 (`Forgotten.Bot`, `Forgotten.Nft.Domain`,
  `Forgotten.Nft.Application`, `Forgotten.Nft.Infrastructure`,
  `Forgotten.Nft.WalletWeb`, `Forgotten.Nft.Integrations`, plus the three test
  projects).
- **Tech stack** is fixed by spec §2.1 (.NET 10 LTS, Discord.Net, EF Core +
  SQLite, Metaplex DAS, Ed25519, xUnit). Do not substitute without a compat note.
- Tasks are checkboxes so progress is trackable. `⚠` marks a security-critical
  item that must fail closed.
