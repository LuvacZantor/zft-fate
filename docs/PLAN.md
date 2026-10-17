# ZFT Fate — Design and Implementation Plan

**Status:** proposed design; no implementation or live Foundry testing completed.  
**Target:** Foundry VTT V13 Build 351, D&D5e.  
**Initial strategy:** small, standalone `zft-fate` module, with Shared Dice holding balances, ZFT Critical Fumbles providing read-only statistics, and ZFT Tools hosting an optional launch button.

## 1. Objectives

1. Move from discretionary Fate Coin/Die distribution to GM-invoked, auditable random awards.
2. Offer GM-selectable, configurable distribution profiles.
3. Allow optional critical/fumble-statistics bonuses without changing the existing detector or duplicate-counting d20 results.
4. Update resource balances for player user accounts whether or not an actor/token is present on a scene.
5. Maintain GM review, confirmation, and manual override; do not force automatic award rules.
6. Preserve module separation and a stable V13 production environment.

## 2. Modules and data ownership

| Component | Owns | Integration strategy |
| --- | --- | --- |
| `zft-fate` | Profiles, award decisions, distribution window snapshots, audit trail, GM UI | Implement directly |
| Shared Dice | Player balances, spending, gifting, caps, resource icons | Existing `.getUserDice()` and `.add()` APIs; initial required integration |
| `zft-critical-fumbles` | Qualified, de-duplicated per-user `{ criticals, fumbles }` | Read via existing `game.zftCriticalFumbles.getUserStats(userId)`; do not mutate counters |
| `zft-tools` | Shared canvas controls | Optional registration of one GM-only `Fate Distribution` action |

ZFT Fate must never scrape or mutate another module's rendered player-list DOM. Use public APIs and native Foundry documents/settings.

### ZFT Tools registration

ZFT Tools `develop` v0.1.0-dev.7 exposes the `zft-tools.collectTools` hook and `registerTool` API. Select **one** registration mechanism; contribute a GM-only button that calls ZFT Fate's window opener. Do not place economy logic inside the toolbar. If ZFT Tools is absent, Fate remains usable independently.

### Critical/fumble semantics

`zft-critical-fumbles` v1.3.1 records **qualified** outcomes for a Foundry **User**. Natural 20s qualify as critical hits on attacks when critical handling is enabled; natural 1 fumbles qualify only for enabled roll categories. Already handled duplicate rolls do not increment twice. Its counters are **cumulative**, and manually resettable. `zft-fate` should snapshot each user's pair of values at a distribution checkpoint and compare at the next one, handling reductions/resets explicitly (e.g., show the issue and require GM review rather than silently extrapolating an unknown interval).

An optional future enhancement is a well-scoped **event hook** in Critical Fumbles if real-time rewards ever become necessary. No such change is needed for phase one.

### Shared Dice semantics

Find/configure existing Fate Coin and Fate Die dice-type IDs, allow the GM to map them once, then use the Shared Dice API. Do not assume exact type names. Display current balances and configured caps before confirmation. Shared Dice `.add()` returns success/failure, not necessarily a measured applied increment; after each action read back the balance and record the **actual delta**. On failure or cap, preserve other successful awards, explain partial application, and keep an idempotent transaction log to avoid double-crediting on retries.

## 3. Proposed user experience

1. GM clicks ZFT Tools → Fate, or opens the alternate ZFT Fate GM entry point.
2. GM selects a named distribution profile and participating player **accounts** (not tokens). Include offline users when selected; exclude GMs by default, with manual inclusion if needed.
3. UI displays each player's existing Fate Coin/Die balance and optional critical/fumble change since the last checkpoint.
4. GM clicks **Roll / Preview**; results show raw random result, profile-awarded coins/dice, rule-based bonuses, and projected balances. No balances change during preview.
5. GM clicks **Apply Awards**; validate GM permission and balances again, apply awards once, and record the results.
6. Offer a human-readable summary card/log entry and GM access to history/corrections.

No notification spam: prefer one consolidated distribution summary over one unsolicited message per award, subject to how Shared Dice's message option behaves in the installed version.

## 4. Configurable profiles

Initial candidates below are drafts, **not approved final balance**. Each participating player independently rolls 1d6.

| d6 | Fate's Whim | Merciful Fate | Harsh Fate |
| --- | --- | --- | --- |
| 1 | Nothing | Nothing | Nothing |
| 2 | Nothing | +1 Coin | Nothing |
| 3 | +1 Coin | +1 Coin | Nothing |
| 4 | +1 Coin | +1 Coin | Nothing |
| 5 | +1 Die | +1 Die | +1 Coin |
| 6 | +1 Coin, +1 Die | +1 Coin, +1 Die | +1 Die |

Profiles should eventually define outcomes by weights, not hard-coded d6 results, and include name, description, enabled state, coin/die grants, maximum per-distribution limits, and bonus policy reference. A default profile can be selected in settings, but every distribution should show the chosen configuration before rolling.

## 5. Bonuses from critical/fumble stats

**Not decided:** qualification window (since last distribution vs per session), thresholds, reward ratios, and whether to normalize by number of dice rolled.

Draft examples for discussion only:

- 2+ qualified natural 1 outcomes since previous checkpoint → potential +1 Coin.
- 2+ qualified natural 20 attack outcomes since previous checkpoint → potential +1 Die.
- GM special recognition → manual credit.

Make bonus rules independently enabled/disabled per profile. Keep the raw random outcome separate from each bonus in previews and history. Do not attempt to infer total rolled d20 counts from the existing counter API.

## 6. Persistence and safety

- User IDs are award identities; world settings or module-owned records store profiles, checkpoints, and immutable distribution history records.
- Persist a unique distribution ID, date, GM user ID, profile snapshot/version, selected player IDs, raw rolls, bonuses, requested awards, before/after balances, per-resource success, and errors.
- Preview is read-only. Applying awards is GM-authorized, rechecks current state and resource IDs, and guards against duplicate/repeated actions.
- Recovery should distinguish already-applied awards from unapplied awards. Never blindly rerun the entire distribution after a partial failure.
- No blanket resets or writes to Critical Fumbles; reset detection must not silently erase earned rewards.
- If Shared Dice is inactive/unavailable, explain that live granting is unavailable. Do not create a parallel balance store in MVP.

## 7. Implementation stages

### Phase 0 — Validate integration contract

- Check installed V13 module versions and exported APIs at runtime.
- Verify Shared Dice type ID mapping, cap handling, `add()` behavior, and balances for offline users.
- Verify ZFT Critical Fumbles `getUserStats()` and baseline/reset edge cases.
- Verify ZFT Tools GM-only button registration and safe alternate entry point.

### Phase 1 — Small GM macro/prototype

- One profile, selected users, preview, GM confirmation, Shared Dice awards, no automatic triggers.
- Exercise on a test world; do not publish as a stable module.

### Phase 2 — `zft-fate` development module

- Profiles/settings UI and reliable award engine.
- ZFT Tools optional integration plus own entry point.
- Baseline tracking, optional bonuses, audit history, and GM override.
- Build using project conventions (`0.1.0-dev.N` on `develop`), with V13 live verification before promotion.

### Phase 3 — Refinement after V13 tests

- Tune probabilities/bonus fairness based on live usage.
- Add optional import/export of configurations if needed.
- Consider V14 compatibility only in a separate test cycle.

## 8. V13 validation checklist

- [ ] UI appears for GM, not ordinary players, and privileged actions are rejected if directly invoked by players.
- [ ] Standalone Fate entry point works with ZFT Tools disabled.
- [ ] ZFT Tools button opens Fate and switching canvas tools remains safe.
- [ ] Players without scene tokens and offline accounts can be selected/updated as intended.
- [ ] Fate Coins and Dice use configured Shared Dice IDs; existing spending/gifting remains unchanged.
- [ ] Preview makes no writes; confirmed distribution makes exactly one award transaction per resource/recipient.
- [ ] Roll/profile outcomes are independently random and auditable.
- [ ] Shared Dice maximum caps and failed writes are shown accurately without duplicate retry awards.
- [ ] Missing Shared Dice fails safely; missing Critical Fumbles disables only statistics bonuses.
- [ ] Stats delta/checkpoint logic handles no activity, normal accumulation, resets and manual edits.
- [ ] Critical/Fumbles counters are never modified by Fate.
- [ ] Browser refresh and Foundry restart preserve profiles, checkpoints and history.
- [ ] GM and player clients both see the resulting balances correctly.
- [ ] Foundry V13 Build 351 live test passed before release claim.

## 9. Decisions still open

1. Define canonical Fate Coin vs Fate Die uses and maximum holdings (these may already exist in Shared Dice).
2. Choose initial profile as an *actual* campaign rule; example percentages are placeholders.
3. Choose eligibility window, bonus thresholds, and whether rewards should compensate low rollers or reward high rollers.
4. Decide whether baseline resets happen with every distribution or only at session boundaries.
5. Decide whether award history should be GM-private or publicly visible in a consolidated summary.

## 10. Repository workflow

- `main` is the stable/default branch; do not land untested code there.
- `develop` is intended for iterative design/testing after separately authorized branch creation.
- Documentation can be proposed now, but GitHub writes require explicit task-scoped approval; documentation changes are not exempt.
- Follow [canonical ZFT development workflow](https://github.com/LuvacZantor/zft-tools/blob/main/DEVELOPMENT.md).
- No version, ZIP, tag, release or live verification is implied by this plan.
