# ZFT Fate

ZFT Fate is a proposed GM-controlled reward distribution module for Foundry VTT V13 Build 351 and D&D5e. It determines Fate Coin and Fate Die awards through configurable random profiles, optionally incorporating qualified critical/fumble statistics. This repository is currently in the **planning** stage; no executable module has been implemented or verified.

## Purpose

Replace ad-hoc Fate awards with transparent, configurable distribution while preserving the GM's ability to grant rewards manually. Awards are attached to **Foundry player users**, not scene tokens, so characters can be absent from the current scene or represented by a group token.

## Architecture

| Component | Responsibility | Required? |
| --- | --- | --- |
| `zft-fate` | Profiles, randomization, distribution UI, bonus calculation, award audit | Core |
| [Shared Dice](https://github.com/Belodri/shareddice) | Stores and displays expendable Fate Coin/Die balances, spending and gifting | Required for initial award backend |
| [ZFT Critical Fumbles](https://github.com/LuvacZantor/zft-critical-fumbles) | Read-only source of qualified critical and fumble totals | Optional for statistics bonuses |
| [ZFT Tools](https://github.com/LuvacZantor/zft-tools) | Optional GM canvas-toolbar launch action | Optional |

**Module boundaries:** ZFT Fate does not clone Shared Dice, write critical/fumble counters, or move gameplay logic into ZFT Tools. It should run via its own GM interface when ZFT Tools is disabled.

## Proposed first experience

1. GM opens **Fate Distribution** (from the ZFT Tools button when available, otherwise from ZFT Fate's own entry point).
2. GM selects a distribution profile and eligible player accounts; no scene tokens are required.
3. Module resolves current Shared Dice resource balances and optionally shows critical/fumble deltas since the saved distribution baseline.
4. GM rolls/preview-calculates each player's rewards and any configured bonuses.
5. GM confirms **Apply Awards** once; the module adjusts Shared Dice balances and saves the actual applied result plus a distribution audit entry.
6. GM can review the award history and manually correct amounts using supported APIs.

A distribution should be **GM-triggered**, not automatically applied on every critical/fumble event. Fate remains the authoritative award policy.

## Draft sample profiles (not approved game rules)

All examples use one independent **1d6 per participating player**. Actual profiles and bonus thresholds are still design choices.

| Roll | Fate's Whim | Merciful Fate | Harsh Fate |
| --- | --- | --- | --- |
| 1 | None | None | None |
| 2 | None | +1 Coin | None |
| 3 | +1 Coin | +1 Coin | None |
| 4 | +1 Coin | +1 Coin | None |
| 5 | +1 Die | +1 Die | +1 Coin |
| 6 | +1 Coin and +1 Die | +1 Coin and +1 Die | +1 Die |

The design should support configurable weighted outcomes, active profile selection, award limits, and future profiles without code changes.

## Draft bonus ideas (disabled until explicitly defined)

- 2+ **qualified** fumbles since previous distribution: possible +1 Fate Coin.
- 2+ **qualified** criticals since previous distribution: possible +1 Fate Die.
- GM recognition: manual award/override.

These are **examples**, not finalized mechanics. Critical Fumbles currently counts qualified natural 20s on **attacks** and eligible natural 1s on configured roll types. It does **not** count every natural 20 as a critical. Counters are per Foundry user and cumulative; `zft-fate` must establish distribution baselines to calculate intervals. Detect counter resets and manual corrections; never mistake a reset for negative activity. Because players roll different quantities of d20s, cumulative counts alone are not necessarily fair reward probabilities.

## Existing APIs to use

- Critical/Fumble read: `game.zftCriticalFumbles.getUserStats(userId)` returns `{ criticals, fumbles }`.
- Shared Dice: `game.modules.get('shareddice').api.getUserDice(userId, diceId)` and `.add(userId, diceId, { amount, chatMessage })`; configure resource **IDs**, not only display names.
- ZFT Tools V13 `develop` implementation: contribute a button through `Hooks.on('zft-tools.collectTools', tools => ...)` **or** `game.modules.get('zft-tools').api.registerTool(moduleId, tool)`, not both.

Handle missing or disabled modules safely; report inability to apply awards rather than manufacturing balances. Respect Shared Dice resource limits and only record **successfully applied amounts**.

## Status and scope

- **Primary target:** Foundry VTT V13 Build 351; D&D5e. V14 is explicitly deferred and not claimed supported.
- **Development:** Planning only. Validate APIs and Foundry-specific lifecycle in V13 before implementation.
- **Release:** No test build, stable tag or GitHub Release is claimed here.
- **Workflow:** `main` is stable; use `develop` for code and test builds. GitHub writes, branches, commits, tags and releases require explicit operation-scoped authorization. See the canonical [ZFT development workflow](https://github.com/LuvacZantor/zft-tools/blob/main/DEVELOPMENT.md).

See [docs/PLAN.md](docs/PLAN.md) for requirements, design decisions, stages, and test cases.
