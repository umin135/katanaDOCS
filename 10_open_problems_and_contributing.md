# 10. Open Problems & Contributing

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

This document honestly lays out **what we never fully solved** and where someone picking this up should start. The original authors leave this before reaching the final stabilization of custom cloth.

---

## 1. The road we walked (summary)

In trying to put a single custom cloth into the game, across dozens of in-game tests, we successively pinned down:

1. **Spike (vertex explosion)** → cause: the display mesh was bound to a `physIdx=0` palette, so the sim vertices' CP indices were misread as palette slots. → needs a **cloth physics palette (physIdx=driver bone)**. ([06] §8)
2. **The two NUNO grids** → every cloth is a **NUNO1+NUNO3 pair** (CP·influences identical, only skip derived from rows/cols). ([06] §2, §5)
3. **Sim vertices in the wrong place** → cause: **parentID was written as local**. Per the forward model, parentID must be **global (l2g[local])** for the CP frame to match. ([06] §6)
4. **Final root cause via full comparison** → the user's authored cloth had rivets perfectly placed on the wrist bone (bone68), but **only the parentID was the donor default (spine)** → resolved by **driver-bone auto-detection (rivet dominant bone)**.

Through this, **our parsing (g2l/l2g) and forward model were empirically verified correct on the KOK sample** (error 0.6). We matched the static structure to KOK with no error.

---

## 2. The key unsolved: final in-game cloth stabilization [UNSOLVED]

- **Status**: we succeeded in matching the static structure (parentID=global, two palettes, NUNO1 pair, rivet placement) to a working sample (KOK) with no error. But **whether a cloth authored with this final structure is fully stable down to the sim position in-game was not finally confirmed.**
- We switched to documentation right after fixing the last-found "parentID=spine" bug with auto-detection, so **the in-game result of that fix is not yet verified.**
- **What the next person should do**: in-game test a g1m whose driver bone was auto-detected as the wrist bone. If the sim position is right, what remains is likely only the [04] tuning (below).

---

## 3. [04] (U_10) reference bone [UNSOLVED]

- The **reference/base bone** at NUNO3 header `+0x10` (and NUNO1 pad idx2). ([06] §7.2)
- **What we know**: it must be a **low SK bone** (a high RF bone crashes). Observed as "the SK root of the driving limb" (arm→SK_L_Shoulder, leg→SK_L_UpLeg, spine→SK_Spine01).
- **What we don't**: the exact selection rule, and **exactly how this field affects sim position/behavior**. There was circumstantial evidence of position mismatch when copying the donor value (spine driver vs leg position), but that observation was from when parentID was wrongly written as local, so it **may be contaminated**.
- **What the next person should do**: with parentID fixed to global, put [04] as (a) the donor value as-is and (b) the driving bone's SK root, and compare in-game positions. Tabulate the [04]↔driver relation across several working samples to derive the rule.

---

## 4. The jiggle constraint [VERIFIED · design limitation]

- **Cloth jiggle comes only when the driver bone actually moves.** A cloth attached to a stationary bone (a still, standing leg) does not jiggle. ([06] §9)
- This looks like an **engine behavior**, not a bug. In the real game, leg-type clothes are made with **waist driving + draping over the legs** (the RYU way).
- **What the next person should do**: if you want a leg cloth, establish an authoring path that replicates the waist/pelvis-driven + leg-collision ([04]=leg SK bone) structure directly from a working sample (a skirt).

---

## 5. The exact role of NUNO1 [UNSOLVED]

- Every cloth has a NUNO1+NUNO3 pair, with **fully identical CP·influences**, and the display attaches only to NUNO3. The in-game reason for **storing NUNO1 redundantly** is unknown. ([06] §2)
- A physics-sim-grid / legacy dual-solver hypothesis exists but is unconfirmed. We observed the sim runs with NUNO3 alone.
- **What the next person should do**: observe in-game a g1m with NUNO1 removed/altered to isolate NUNO1's actual contribution.

---

## 6. globalToFinal for an external skeleton [UNSOLVED]

- We measured `globalToFinal[l2g[X]]=X` in a self-contained g1m, but how globalToFinal is merged for **a real game costume referencing an external base skeleton** is unverified. ([04] §3)
- If the authoring target is a real costume (external skeleton), the parentID resolution could differ.

---

## 7. Other unsolved (asset-system layer)

- **Name hash**: the filename → FileKtid hash algorithm is uncracked (nonlinear). Worked around with a CSV table. ([01] §5)
- **objID → g1t**: depends on the game singleton DB, no offline auto-resolution. ([02] §5)
- **sid integrated pipeline**: an auto pipeline covering sid registration + asset location (RDB) update for a fully custom mesh is incomplete. ([08] §3)
- **Byte-identical regeneration of every chunk**: partly achieved, fully incomplete. ([08] §7)

---

## 8. Practical tips for contributors

### 8.1 Methodology — "understand the sample completely first"
The most valuable lesson: **analyze a working sample you have (e.g. the KOK wrist cloth) field by field, until you can reproduce it with no error, and compare it against your own work.** We long oscillated between "it's a limit / we're doing well," until a **full comparison of the sample** found the root cause (parentID=spine). Comparison beats guessing.

### 8.2 Useful checks
- **Forward-model check**: for any cloth (a working one or your own), compute whether the world center of `world[g2l(parentID)]·CP_local` matches that cloth's **display mesh center**. If it diverges, the frame (parentID local/global) is wrong.
- **Full field comparison**: print, side by side for your work vs a working sample, parentID / [04] / cpc / palette (physIdx·jointIdx) / rivet weighted bone / NUNO1 pair / G1MF counts, and diff.
- **Static parse tools**: dump the g1m to XML etc. to confirm chunks parse normally and physicsIndex (cloth marker) · parent match expectations. (In a tool's transform matrices, treating very small numbers as 0 makes reading easier.)

### 8.3 Safe habits
- Always back up the original (`.kmbak` etc.), no in-place edits, idempotent recomputation from a clean original.
- Insert entries at section ends, preserving index references.
- Update G1MF counts by **exact offset only** (no search).

---

## 9. Re-affirming the most important settled facts (for whoever picks this up)

- **parentID = global (l2g[driver-local-bone]).** CPs are in the world[driver-local-bone] frame. (forward model, verified)
- **The display mesh binds to a physIdx=driver-bone cloth palette.** Otherwise, a spike.
- **Non-identity bones need two palettes** (display local + physics global).
- **The driver bone = the rivet dominant bone** = must be a moving bone to jiggle.
- **Every cloth = a NUNO1+NUNO3 pair** (skip derived from rows/cols).
- **Our parsing (g2l/l2g) and forward model are correct** (verified on KOK). What remains is final in-game stabilization and pinning down [04].

Good luck. Just reaching this point has already cleared most of the walls.
