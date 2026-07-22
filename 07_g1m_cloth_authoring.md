# 07. g1m Custom Cloth Authoring Recipe (proven procedure)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].
> Format basis is in [06_g1m_cloth_system.md](06_g1m_cloth_system.md). This document is the "how to make it" procedure.

**Goal:** add an arbitrary cloth mesh (user-modeled in 3D) into a g1m as a new NUNO cloth, using a working original (donor) cloth as a template.

**Prerequisite:** you have one working donor cloth (e.g. the character's default costume cloth) and that g1m's skeleton (jil/oid).

---

## 1. Inputs

What you need to author:

- **Display mesh**: the cloth mesh to render (vertices/triangles/UV). Vertices have **bone weights** painted (the rivet = attachment area is weighted to the driver bone).
- **Proxy grid**: the sim control-point (CP) grid. `rows × cols`. Top 2 rows = the anchor (attachment) rows.
- **Rivet mask**: which display vertices are rivets (fixed).
- **Donor cloth**: reused as the template for head params · anchors · influences.

---

## 2. ★Step 1 — Decide the driver bone (the root of parentID)

**Decide the cloth's sim driver bone (local).** If this is wrong, everything goes wrong.

- **Auto-detection [VERIFIED, recommended]:** the **dominant bone of the rivet vertices (the bone with the max weight sum)** = the driver bone.
  - Rationale: rivet = the sim attachment area → that bone is the one the cloth actually hangs from. In measurement the rivets of a wrist cloth were weighted to `RF_L_ForeArm_01`, and that bone was the correct driver.
  - If the user specifies it explicitly, prefer that.
- **The pitfall [VERIFIED]:** if you do not specify the driver bone, the **donor default (e.g. spine bone10)** is used. Then the rivets are on the wrist but only the parentID is the spine → **the sim is spine-driven → does not follow the arm when it moves (wrong place).** Always pin it to the rivet bone.
- **The jiggle constraint [VERIFIED]:** the driver bone must be a **bone that moves even in idle** (wrist/forearm/waist) for it to jiggle. A stationary leg bone does not jiggle ([06] §9).

The decided **local driver bone = X**.

---

## 3. ★Step 2 — parentID = global [VERIFIED]

- **parentID = l2g[X]** (the global ID of the local driver bone). For both the NUNO3 and NUNO1 headers.
- Reason: the forward model ([06] §6). CPs are encoded in the `world[X]` frame and the game places them via `world[globalToFinal[parentID]]` → since `globalToFinal[l2g[X]]=X` they match.
- **Do not write it as local** — globalToFinal[local] = the wrong bone → sim vertices in the wrong place.
- Verified example: wrist bone68 → parentID = l2g[68] = **75** (identical to the real KOK value).

---

## 4. ★Step 3 — Encode CPs into the driver local frame [VERIFIED]

- Convert the proxy grid CPs (model space) to **`CP_local = inv(world[X]) · CP_model`** and store.
- Generate influences (P1~P4 topology) from the grid rows/cols, and **recompute P5/P6 rest-length from CP_local neighbor distances**.
- Also compute the anchor (top 2 rows) direction vectors in the **driver local frame** (a rotated bone tilts them accordingly).

---

## 5. ★Step 4 — Create two cloth palettes [VERIFIED]

Make the **cloth physics palette** the display mesh will bind to. (Reusing an existing physIdx=0 palette causes a spike — [06] §8.)

- **Display palette**: entry = `(G1MMIndex(X), physIdx=X, jointIdx=X)`. Size = donor palette size. → make the **submesh bmap point to this palette**.
- **Physics palette**: entry = `(G1MMIndex(X), physIdx=l2g[X], jointIdx=l2g[X])`. (Only when local≠global; an identity bone needs the display one alone.)
- **G1MMIndex(X)** is found by looking up the G1MMIndex of an entry with `jointIndex==X` in existing palettes, and reusing it.
- **Append the two palettes at the end of 0x10006** and update `npal`.
- **G1MF**: `num_bone_maps(inner 0x34) += palettes added`, `num_individual_bone_maps(0x38) += total entries added`. (No search, exact offset.)

> Because the palette is all-same-bone (X), whichever slot the rivet/anchor references, it skins to bone X. Sim vertices (cpW2≠0) are interpreted by CP index regardless of the palette, so it is safe.

---

## 6. ★Step 5 — Insert the NUNO3 entry [VERIFIED]

Using a donor NUNO3 entry as the template, replace the following and **insert at the end of the NUNO3 section**:

- Header: parentID = l2g[X], cpc/unkcnt = new grid, [04] = (keep donor value or an SK reference bone), skip4·skip3 tail links.
- head params: copy the donor's (stiffness etc.). `@0xBC` (the NUNO4-use flag) = 0 when from-scratch.
- CP·influences·anchors·skip3 tethers: generated from the new grid.
- Update the NUNO3 section chunk_size · entry_count.
- **G1MF NUNO3 counts**: `num_nuno3s(0xE4)+1`, `control_points(0xEC)+=cpc`, `unk1(0xF0)+=unkcnt`, etc.

The new NUNO3 index = the previous NUNO3 entry count.

---

## 7. ★Step 6 — Create the NUNO1 pair [VERIFIED]

Wrap the CP·influences·anchors of the NUNO3 you just made into the **NUNO1 skip structure and insert at the end of the NUNO1 section**:

- Header: parentID = l2g[X], same cpc/unkcnt, **skip1=cols, skip2=rows-2, skip3=1**.
- pad (0x5C): copy the donor NUNO1 pad, **set pad idx2 (entry@0x20) = reference bone** equal to the NUNO3 [04].
- CP·influences·anchors = the NUNO3's, as-is (identical).
- skip: `skip1=[0..cols-1]`, `skip2=[r*cols|r=2..rows-1]`, `skip3=[2]`.
- **G1MF NUNO1 counts**: `num_nuno1s(0x68)+1`, `unk4(0x6C)+1`, `control_points(0x70)+=cpc`, `unk1(0x74)+=unkcnt`, `unk2_and_unk3(0x78)+=(cols+rows-2)`.

---

## 8. ★Step 7 — Add & bind the display mesh [VERIFIED]

- Append the display mesh (vertices/IB) as a new VB/IB/submesh.
- **Submesh bmap = the display palette index** (Step 4).
- Vertex semantics:
  - **Rivet vertices (cpW2=0)**: `cpW1=rest(bone skinning frame)`, `i1 = 3×slot` (the driver bone slot).
  - **Sim vertices (cpW2≠0)**: bilinear-bound to the proxy grid, `i1 = CP index` (4 CPs).
- **Mesh group**: insert a new entry at the end of block1, `ext_id = 20000 + new NUNO3 index`.
- **G1MF**: num_vb/num_layouts/num_submeshes/num_meshes etc. +1.

---

## 9. ★Step 8 — Align the anchor bone reference [VERIFIED]

- Anchor (unkSection) `@0x24 = 3 × (the driver bone slot in the display palette)`.
- If the palette is all-same-bone, any slot is the driver bone, so as long as the rivet/anchor slots are within the palette range it is fine.

---

## 10. Post-build verification (static, before in-game)

Compare the built g1m **field by field** against a working sample (e.g. KOK):

- NUNO3/NUNO1 parentID = l2g[X] (global) ✓
- The display submesh → a physIdx=X(local) palette, and a separate physIdx=l2g[X](global) physics palette exists ✓
- Rivets skinned to the driver bone ✓
- The world center of `world[X]·CP` = the display mesh center (forward model, small error) ✓
- G1MF counts consistent ✓
- (Whether chunks parse normally via an XML dump tool, etc.)

**Measured result:** applying this procedure with the wrist bone (bone68) generates parentID=75 + a display palette (physIdx=68) + a physics palette (physIdx=75) + the NUNO1 pair — **matching the real KOK structure with no error**.

---

## 11. Remaining uncertainty [UNSOLVED]

We matched the static structure to KOK with no error, but **whether a cloth authored with this final structure is fully stable in the sim-vertex position in-game was not finally confirmed.** In particular:

- **The exact value and effect of [04] (the reference bone)** — circumstantial evidence of position mismatch when copying the donor value (spine driver vs leg position). Needs re-verification after the forward-model fix (parentID=global).
- **Jiggle of stationary-bone clothes** — fundamentally impossible (needs a moving bone).

For the full unsolved list and next steps, see [10_open_problems_and_contributing.md](10_open_problems_and_contributing.md).
