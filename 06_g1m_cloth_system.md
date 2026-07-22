# 06. g1m Cloth (NUNO) System — Deep Dive

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].
> **The core of this doc set.** The most measurement and trial-and-error is here.

The **NUNO** chunk holds the **cloth simulation** of a g1m. This document lays out its binary structure, the forward model of "why cloth behaves as it does," and the causes of the **spike**, **wrong position**, and **no jiggle** issues that had us wandering so long.

---

## 1. NUNO chunk overview [VERIFIED]

The NUNO chunk is a bundle of sections:

```
NUNO inner:
  section_count (u32)
  repeat: (magic u32, chunk_size u32, entry_count u32) + entries
```

| magic | name | role |
|---|---|---|
| 0x00030001 | **NUNO1** | control-point grid (cloth). **No display (physics grid only).** |
| 0x00030003 | **NUNO3** | control-point grid (cloth). **A display mesh links to it (ext=20000+idx).** |
| 0x00030004 | **NUNO4** | driver-bone table (local↔global mapping, cloth driver chain). |
| 0x00010001 | **NUNV** | vertex-based cloth (a different kind). |
| 0x00030005 | NUNO5 | newer (Ryza/Yumia family). Absent in DOA6. |

---

## 2. ★Every cloth = a NUNO1 + NUNO3 pair [VERIFIED]

**Every real in-game cloth stores the same grid twice, in the NUNO1 and NUNO3 sections.**

- Measured: for one donor cloth, NUNO1[0] and NUNO3[0] are **fully identical in control points 98/98**, and **fully identical in influences (P1~P4 topology + P5/P6 rest-length) 98/98**. Same parentID.
- KOK wrist cloths: NUNO1[75,76] + NUNO3[75,76] (a pair per left/right wrist).
- RYU skirt: 2 NUNO1 + 2 NUNO3.

That is, **CP·influences are identical between the two sections, and the only difference is the "skip structure"**, which is **derived from the grid's rows/cols** (§5). So when authoring you can **generate both NUNO1 and NUNO3 from just one grid (rows/cols/CP)**.

- **A display mesh links only to NUNO3** (ext=20000+idx). NUNO1 exists as a physics grid with no display. [VERIFIED]
- **The exact in-game role of NUNO1 (why store it redundantly) was not fully explained** — presumed a physics sim grid / legacy dual-solver. [HYPOTHESIS] Still, **since real cloths always have the pair, making the pair when authoring is safe.** (We measured that the sim itself runs with NUNO3 alone, but recommend the pair to match the original structure.)

---

## 3. NUNO3 entry structure [VERIFIED]

Based on version `"9200"` (0x39323030).

```
NUNO3 entry:
  [header 32 bytes]
    +0x00  parentID  (u32)   # ★global bone ID (§8 forward model)
    +0x04  cpc       (u32)   # control point count (rows×cols)
    +0x08  unkcnt    (u32)   # anchor (unkSection) count = 2×cols (top 2 rows)
    +0x0C  skip1     (u32)
    +0x10  [04]/refBone (u32) # ★reference/base bone (see §7 U_10)
    +0x14  skip2     (u32)
    +0x18  skip3     (u32)   # tail link (tether) count = (rows-2)×cols
    +0x1C  skip4     (u32)
  [head params]  +0x20 ~ +0xD7  (0xA8+0x10 padding by version; §7)
  [control points]  cpc × 16 bytes (RichVec4: x,y,z,w)
  [influences]  cpc × 24 bytes (NunInfluence)
  [unkSection(anchors)]  unkcnt × 48 bytes
  [skips]  (4×skip1 + 8×skip2 + 12×skip3 + 8×skip4) bytes
```

- **CP start offset = 0xD8** (header 32 + head 0xA8 + 0x10). [VERIFIED]

---

## 4. Control points · influences · anchors [VERIFIED]

### 4.1 Control point (RichVec4)
- `(x, y, z, w)` 4 floats, 16 bytes.
- **The position is stored in the driver bone's local frame** (§8).
- `w` has a flag/inverse-mass character (observed often 1.0, unrelated to the rivet/sim distinction).

### 4.2 influences (NunInfluence, 24 bytes)
```
struct NunInfluence {
  int   P1;   # left  neighbor CP index (-1 = none)
  int   P2;   # right neighbor CP index
  int   P3;   # up (parent) neighbor CP index (-1 = topmost row = root)
  int   P4;   # down (child) neighbor CP index
  float P5;   # rest-length (distance to P2 neighbor)
  float P6;   # rest-length (distance to P4 neighbor)
}
```
- **P1~P4 = grid topology (4-way neighbors)**. This defines the cloth's constraint graph.
- **A CP with P3 = -1 = the topmost row (root)** — the row pinned to anchors. [VERIFIED]
- **P5/P6 = rest-length** (spring natural length). If you scale/deform the grid you must recompute them. [VERIFIED]

### 4.3 Anchor (unkSection, 48 bytes) [VERIFIED]
Pins the top 2 rows (= 2×cols) of control points to a bone.
```
anchor 48 bytes:
  +0x00  direction vector / data (8 floats = 32 bytes)  # normal etc., rotated into the driver bone frame
  +0x20  count (u32)                                     # bone-weight count of this anchor
  +0x24  packedBoneIdx (u32)                             # ★3 × (palette slot) — same encoding as rivet i1
```
- **The anchor's `@0x24` (packedBoneIdx) is 3×palette-slot.** Writing a raw local bone gets //3-misread by the game → top↔body frame mismatch → tearing. [VERIFIED]
- The anchor direction vector (@0x00) is stored **rotated into the driver bone frame** (an axis-aligned bone gives clean values like (0,0,-1)(0,1,0); a rotated bone gives correspondingly tilted values). [VERIFIED]

---

## 5. NUNO1 entry structure & skip derivation [VERIFIED]

NUNO1 has CP·influences·anchors identical to NUNO3 but a different header/padding/skip.

```
NUNO1 entry (ver 9200):
  [header 24 bytes]
    +0x00 parentID, +0x04 cpc, +0x08 unkcnt,
    +0x0C skip1, +0x10 skip2, +0x14 skip3
  [pad 0x5C]   # 0x3C + 0x10(ver>0x30303233) + 0x10(ver>=0x30303235)
               # pad idx2 (entry@0x20) = reference bone (same as NUNO3's [04])
  [control points] cpc×16   [influences] cpc×24   [unkSection] 48×unkcnt
  [skips]  4×(skip1+skip2+skip3)
```

### 5.1 skip values = derived from rows/cols [VERIFIED]
When the grid is `rows × cols` (cpc = rows×cols, cols = unkcnt//2):

| skip | count | contents (index list) |
|---|---|---|
| skip1 | cols | first row: `[0, 1, …, cols-1]` |
| skip2 | rows-2 | col0 of rows 2+: `[r*cols for r in 2..rows-1]` |
| skip3 | 1 | `[2]` |

- Measured example: cpc=98, unkcnt=14 → cols=7, rows=14. skip1=7([0..6]), skip2=12([14,21,…,91]), skip3=1([2]).
- **So one grid (rows/cols) lets you derive all NUNO1 skips** — no separate data needed.

---

## 6. ★Forward Model — how the sim position is determined [VERIFIED]

The part that had us wandering longest. **A CP's world position is determined like this:**

```
CP_world = world[ globalToFinal[parentID] ] · CP_local
```

- **On authoring (encoding)**: store CP as `CP_local = inv(world[driver-local-bone]) · CP_model`. That is, CPs are held in the **driver local bone's frame**.
- **On the game side (decoding)**: place with `world[globalToFinal[parentID]] · CP_local`.
- **For the two frames to match**: `globalToFinal[parentID] = driver-local-bone`.
- In a self-contained g1m, `globalToFinal[l2g[X]] = X` ([04] §3), so **parentID = l2g[driver-local-bone] = a global ID**.

### 6.1 Empirical verification (KOK wrist cloth) [VERIFIED]
- NUNO3 parentID = 75. g2l[75] = local bone68 (RF_L_ForeArm_01).
- The world center of `world[bone68] · CP_local` = **(57.8, 6.4, -2.6)**.
- That cloth's **display mesh center = (57.9, 6.9, -3.0)**. **Distance 0.6** → effectively identical.
- So the forward model holds exactly. This also confirms our parsing (g2l/l2g) is correct.

### 6.2 The bug of writing parentID as "local" [VERIFIED]
- Writing parentID = local bone (e.g. 68) → `globalToFinal[68]` = the **wrong bone** that is global 68 (e.g. a thigh) → CP frame mismatch → **sim vertices tremble in the wrong place**.
- (This symptom was long misread as a "limitation," but the cause was the parentID local/global confusion.)

---

## 7. head params and [04] (U_10) [VERIFIED/UNSOLVED]

### 7.1 head params (NUNO3 +0x20~0xD7)
- Stiffness · damping · other sim parameters are stored as floats/flags.
- Observed: comparing two clothes of the same character (left/right), **most head fields are identical** and **only a few fields differ per cloth** (e.g. two stiffness values, plus [04]).
- When authoring, **copying the donor (a working original) head wholesale** is safe (reproduces stiffness etc.).

### 7.2 [04] = U_10 = reference/base bone (NUNO3 +0x10) [VERIFIED/UNSOLVED]
- This field holds a **low SK bone index**. Observed examples:
  - Spine cloth (driven by SK_Spine02) → [04]=SK_Spine01
  - Wrist cloth (driven by RF_L_ForeArm) → [04]=SK_L_Shoulder (the arm's SK root)
  - Skirt (driven by SK_Spine02) → [04]=SK_L_Leg (the leg)
- **[04] must be a "low SK bone."** Putting a high RF bone (twist) **crashes the game.** [VERIFIED]
- **The exact rule for [04] and its effect on the sim are unsolved.** [UNSOLVED] There was circumstantial evidence that copying the donor value misplaces the cloth (spine driver vs leg position), but that observation was from the era of wrongly writing parentID as local, so **it may be contaminated**. It must be tuned by in-game testing after authoring.

---

## 8. ★Cloth palette (physicsIndex) and the spike [VERIFIED]

**For a cloth to simulate, the physicsIndex of the joint palette the display mesh binds to must be the driver bone.**

### 8.1 Vertex semantics: rivet vs sim [VERIFIED]
A cloth display mesh's vertices are of two kinds, **distinguished by the `cpW2` flag** (the 4 floats at vertex-buffer offset 32):

| vertex kind | cpW2 | meaning of i1 (blend index) |
|---|---|---|
| **rivet (fixed)** | = 0 | **palette slot** (3×slot). Skinned directly to a bone. |
| **sim (free)** | ≠ 0 | **control point (CP) / driver joint index**. Driven by NUNO physics. |

- That is, a sim vertex's i1 is a "CP index," and the game interprets it as a **NUNO driver joint** (not a palette slot).

### 8.2 The cause of the spike [VERIFIED]
- The game must recognize "this mesh is a physics (cloth) mesh" to correctly interpret a sim vertex's (cpW2≠0) i1 as a CP index.
- **This recognition is done by the physicsIndex of the palette the display mesh binds to.** If `physicsIndex = driver bone`, it is recognized as a physics mesh.
- **If bound to a `physicsIndex=0` (ordinary) palette**, the game mistakes it for ordinary skinning → misreads the sim vertex's i1 (=CP index, e.g. 7~11) as a **palette bone slot** → skins to the wrong bone → **vertex explosion (spike).** [VERIFIED, confirmed with an XML dump tool]
- (An early authoring tool reusing an existing 26-bone `physIdx=0` palette was the root cause of the spike.)

### 8.3 Display palette vs physics palette (two needed) [VERIFIED]
For non-identity bones (local≠global), **two palettes** are needed (the real KOK structure):

| palette | physicsIndex | jointIndex | purpose |
|---|---|---|---|
| **display** | local driver bone | local bones | **the submesh binds here**. Rivet/anchor slots reference this. |
| **physics** | **global driver bone (= parentID)** | global bones | game physics link. Not bound by a submesh. |

- **The G1MMIndex is the driver bone's for both palettes** (the skinning matrix is determined by G1MMIndex, so the same bone matrix regardless of local/global).
- An identity bone (local=global, e.g. bone10) needs only one (physIdx=10 is both local and global).

---

## 9. ★The jiggle constraint — the driver bone must move [VERIFIED]

- **The visible jiggle of a cloth sim comes from the actual movement of the driver bone (parentID).**
- Measured:
  - bone10=SK_Spine02 (always moving slightly with breathing) driver → jiggles even in idle.
  - RF_ForeArm (the arm sways) driver → jiggles.
  - **RF_UpLeg (a still, standing leg, stationary) driver → rigid even when the leg moves** (no jiggle).
- That is, **a cloth attached directly to a stationary bone does not jiggle.** [VERIFIED]
- Real game leg/skirt clothes are made with **the waist (moving SK_Spine02) as the driver + draping over the legs** (the RYU way). The leg itself is not used as the driver.
- **Implication:** if you want a jiggling cloth, take as driver **a limb that moves even in idle (wrist/forearm) or the waist**.

---

## 10. NUNO4 — the driver-bone table [VERIFIED/HYPOTHESIS]

- magic 0x00030004. A **local↔global mapping table** for cloth driver bones (a continuous cloth driver chain).
- Structure (observed): head[1] = bone count n, tail holds n (local, global) pairs.
- Observed as converting the anchor's local bone indices into global skeleton bones. [HYPOTHESIS: exact role unconfirmed]
- Our authored clothes (NUNO3 alone/pair) were observed to work without a new NUNO4, so NUNO4 is presumed donor-cloth-only or for a specific feature. [HYPOTHESIS]

---

## 11. Summary checklist (what a working cloth must have)

- ☐ A NUNO3 entry (+ paired NUNO1) — CP·influences·anchors·skip.
- ☐ **parentID = global (l2g[driver-local-bone])**.
- ☐ CPs encoded in the **world[driver-local-bone] frame**.
- ☐ **A display palette (physIdx=local bone)** + **a physics palette (physIdx=global bone)**.
- ☐ The display mesh **bound to the display palette** (bmap).
- ☐ Rivet i1 = 3×slot, anchor @0x24 = 3×slot (the display palette slot).
- ☐ Mesh group ext_id = 20000 + NUNO3 index, inserted in block1.
- ☐ All G1MF counts updated (palette/submesh/NUNO1/NUNO3).
- ☐ The driver bone must be a **moving bone** for jiggle to be visible.
- ☐ [04] = a low SK bone (the exact value is [UNSOLVED], tune in-game).

The step-by-step authoring procedure is in [07_g1m_cloth_authoring.md](07_g1m_cloth_authoring.md).
