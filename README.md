# KatanaEngine Assets & g1m Research — Master Index

> **This document set is a standalone knowledge base.**
> Files inside `_docs/_2_research_docs/` **never reference any file outside this directory.**
> All formats and algorithms are described self-containedly here, in prose / tables / pseudocode.
> (We do not point to external tools or implementation files. We capture the knowledge itself.)

---

## 0. What this is

This document set records **facts discovered through reverse engineering + in-game measurement** about the asset formats of games built on Koei Tecmo's **KatanaEngine** (in particular **Dead or Alive 6: Last Round**), gathered while trying to build a custom mod layer (codenamed **ktmod**) on top of them.

The central subject is the **`g1m` 3D model format**, and within it the **NUNO cloth simulation system**. The most effort and empirical verification went into that part, and the center of gravity of these docs is there too.

This is left **so that someone can pick it up and contribute**. We (the original authors) leave this before fully stabilizing the last step of custom cloth authoring (getting the sim vertices to land in the right place in-game). What we learned and what is still unsolved is summarized in [10_open_problems_and_contributing.md](10_open_problems_and_contributing.md).

---

## 1. Important disclaimers (read first)

### 1.1 The ktmod plan is an "initial plan" and unstable
Everything written here about **ktmod (the mod package / editor / manager) design and operation is only an initial plan, and it is unstable.** It is not something implemented and verified — it is a direction, a "this should work" sketch. In particular:

- **The ktmod `Content/` mechanism (holding assets symbolically so the manager resolves them at install time) has been deferred indefinitely.** The related writing ([09_ktmod_design_plan.md](09_ktmod_design_plan.md)) is only a directional record, not a settled spec.
- Plan-related text can change at any time and must not be implemented as-is. Read carefully, distinguishing **changeable plans** from **proven format facts**.

### 1.2 Format knowledge is empirical, but only one game is verified
Format writing is backed by **binary analysis + in-game measurement**. However:

- **The only fully verified game is DOA6: Last Round.** Same-engine relatives (Wo Long / Nioh / Atelier, etc.) look similar in format but are unverified.
- The game engine internals (physics sim solver, exact globalToFinal computation) were understood **only by observation/inference** since there is no source. We measured the observable inputs/outputs, but some explanations of internal behavior are hypotheses.

---

## 2. Verification-grade tags

Throughout, reliability is marked with the tags below.

| Tag | Meaning |
|---|---|
| **[VERIFIED]** | Directly observed / reproduced in-game or in the binary |
| **[DERIVED]** | Logically derived from several verified facts, with consistency confirmed |
| **[HYPOTHESIS]** | An estimate not yet confirmed in-game / empirically. Falsifiable |
| **[UNSOLVED]** | A problem we never cracked |

---

## 3. Reading order / structure

Ordered from the big picture of the asset system → g1m format → cloth (the core) → plans → unsolved.

1. [01_katana_engine_asset_system.md](01_katana_engine_asset_system.md) — RDB/RDX/FDATA index/data layers, IDRK blocks, zlibext, KTID hashing, override strategies.
2. [02_asset_reference_chain.md](02_asset_reference_chain.md) — how one character costume connects across many assets (model/texture/material). kidsobjdb, the singleton DB, the unsolved name-hash problem.
3. [03_g1m_container_and_chunks.md](03_g1m_container_and_chunks.md) — g1m container structure and chunk list.
4. [04_g1m_skeleton_and_matrices.md](04_g1m_skeleton_and_matrices.md) — G1MS skeleton, jil (local↔global), globalToFinal, G1MM inverse bind matrices, oid bone names.
5. [05_g1m_geometry.md](05_g1m_geometry.md) — G1MG geometry sections, vertex buffer semantics, joint palettes, skinning, mesh groups.
6. [06_g1m_cloth_system.md](06_g1m_cloth_system.md) — **the core.** Deep dive into the NUNO cloth format: NUNO1/NUNO3 pairs, control points, influences, anchors, head params, the **forward model**, the **cloth palette (physIdx)**, the cause of spikes.
7. [07_g1m_cloth_authoring.md](07_g1m_cloth_authoring.md) — the proven step-by-step custom cloth authoring recipe.
8. [08_g1m_textures_materials_physics.md](08_g1m_textures_materials_physics.md) — g1t textures, kts materials, sid mesh registry, SOFT soft body.
9. [09_ktmod_design_plan.md](09_ktmod_design_plan.md) — ktmod plan (initial, unstable, Content deferred indefinitely).
10. [10_open_problems_and_contributing.md](10_open_problems_and_contributing.md) — unsolved problems, hypotheses, contributing guide.

---

## 4. Glossary

Minimum terms needed to follow the formats. Details are in each doc.

| Term | Meaning |
|---|---|
| **RDB / RDX** | Asset index. RDB = file location metadata (file_ktid→location), RDX = filename-hash mapping. |
| **FDATA** | The actual asset data container. A collection of IDRK blocks, zlibext-compressed. |
| **IDRK** | An asset-unit block inside FDATA (`IDRK` magic). |
| **KTID** | 32-bit asset hash. **FileKtid** = per-asset unique (from filename), **TypeKtid** = type class. |
| **g1m** | KatanaEngine 3D model container (skeleton + geometry + physics). |
| **chunk** | A sub-block inside a g1m (G1MF/G1MS/G1MM/G1MG/NUNO/SOFT etc.). |
| **G1MS** | The g1m skeleton chunk (bone hierarchy + jil). |
| **jil** | Joint Index List. The local-bone ↔ global-bone ID mapping table. |
| **local ID** | Index into this g1m's own bone array (0..N). |
| **global ID** | Bone ID in the global (merged) skeleton. jil converts local↔global. |
| **globalToFinal** | The runtime table the game uses to map a global ID to a final joint-array position. |
| **G1MM** | The inverse-bind-matrix array chunk. Indexed by **G1MMIndex**. |
| **G1MG** | The geometry chunk. Contains vertex-buffer / layout / index / submesh / joint-palette / mesh-group sections. |
| **joint palette (bone map)** | The list of (G1MMIndex, physicsIndex, jointIndex) entries a submesh references. |
| **physicsIndex (cloth)** | A joint-palette entry field. **If ≠0, that palette is a cloth physics palette** (the game recognizes it as a physics mesh). |
| **NUNO** | The cloth simulation chunk. NUNO1/NUNO3 (control-point grids), NUNO4 (driver table), NUNV (vertex cloth). |
| **control point (CP)** | A node of the cloth sim grid (RichVec4 xyz+w). |
| **influences (NunInfluence)** | Per-CP topology/constraints (P1~P4 = neighbor CP indices, P5/P6 = rest-lengths). |
| **anchor (unkSection)** | A 48-byte block that pins control points to a bone (direction vector + bone reference). |
| **parentID** | The sim driver bone of a NUNO entry. **Stored as a global ID** (→ resolve to local via jil/globalToFinal). |
| **rivet** | A fixed display-mesh vertex skinned directly to a bone (cpW2=0 flag). |
| **sim vertex** | A free vertex driven by control points (driver joints) (cpW2≠0). |
| **oid** | The bone-name table (global ID → name hash → string). |
| **sid** | Character.sid. A per-character singleton registry that gates render/material/physics per mesh hash. |
| **kts / g1t / kidsobjdb** | Material (KTS), texture (g1t), object-reference DB. See [02](02_asset_reference_chain.md) · [08](08_g1m_textures_materials_physics.md). |
| **ktmod** | This project's custom mod package/editor/manager (initial plan, unstable). |

---

## 5. One-line summary (the most valuable findings)

For the busy reader, the most valuable things we proved:

- **[VERIFIED]** g1m cloth **encodes CPs in the `world[driver-local-bone]` frame**, and the game **places them via `world[globalToFinal[parentID]]`**. Therefore **parentID must be a global ID (= l2g[local-bone])**. (Measured on the KOK sample: within 0.6 units.)
- **[VERIFIED]** Every real cloth is stored as a **NUNO1 + NUNO3 pair** (control points and influences fully identical, only the skip structure derived from rows/cols differs).
- **[VERIFIED]** The cloth display mesh must be bound to a **palette whose physicsIndex = the driver bone**. Otherwise (physIdx=0) the game misreads the sim vertices' indices as palette slots and the **vertices explode (spike)**.
- **[VERIFIED]** The visible jiggle of a cloth sim comes **only when the driver bone actually moves**. A cloth attached to a stationary bone (a still, standing leg) does not jiggle.

Why these hold and how we verified them is covered in detail in [06](06_g1m_cloth_system.md) · [07](07_g1m_cloth_authoring.md).
