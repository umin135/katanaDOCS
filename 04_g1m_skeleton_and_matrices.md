# 04. g1m Skeleton & Matrices (local/global IDs · G1MM · oid)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

This document covers the **bone coordinate system** of a g1m. The **local ID ↔ global ID** concept in particular is essential to understanding cloth parentID. One of the root causes of repeated cloth-authoring failures was exactly this ID confusion (see [06] · [07]).

---

## 1. G1MS — the skeleton chunk [VERIFIED]

G1MS holds the bone hierarchy (each bone's parent, local transform) and the **jil (Joint Index List)**.

### 1.1 Bone data
- Each bone has a **parent index** and a **local transform (Translation / RotationQuaternion / Scale)**.
- A bone's **world transform** is computed by accumulating local transforms up the parent chain (`world[b] = world[parent] · local[b]`). [DERIVED]

### 1.2 Local ID vs global ID (★core)
g1m uses two bone-numbering systems:

- **local ID** — index into this g1m's own bone array (0..N-1). "Which bone within this file."
- **global ID** — the bone ID in the global (merged/base) skeleton. The character-skeleton numbering shared by multiple g1ms (costumes).

What connects the two is **jil**.

---

## 2. jil — local↔global mapping [VERIFIED]

jil is an integer array defining the following relations:

```
localIDToGlobalID[jil[i]] = i          # (standard definition)
globalIDToLocalID[i]      = jil[i]
```

In practice we handle it with two functions:

- **l2g(local) = local_to_gid(local)** : local bone → global ID
- **g2l(global) = gid_to_local(global)** : global ID → local bone

### 2.1 jil parse location [VERIFIED]
- The jil count (jic) is at **offset 0x0A (u16)** of the G1MS inner.
- The jil array comes from **offset 0x10** of the G1MS inner, `jic` u16 values.
  - (This offset was mis-located several times before being pinned by measurement. We verified it by whether original clothes' parentID resolution matched.)
- Invalid entries are marked `0xFFFF` (that local slot maps to no global).

### 2.2 Measured example (one costume skeleton)
| local bone | name (oid) | l2g (global) |
|---|---|---|
| bone10 | SK_Spine02 | 10 (**identity**) |
| bone68 | RF_L_ForeArm_01 | **75** |
| bone72 | RF_L_UpLeg_01 | **79** |

- **Low SK (main skeleton) bones are often local=global (identity).** (e.g. bone10→10)
- **RF (twist/rig) bones, etc., have local≠global.** (e.g. bone68 local → global 75)
- So **mistaking "global 75" for a local number points at the wrong bone** (a thigh, etc.). Since parentID is global, you must always **apply g2l** to get the local driver bone. [VERIFIED] (This confusion is one reason cloth debugging wandered so long.)

---

## 3. globalToFinal — runtime joint resolution [DERIVED/HYPOTHESIS]

The game runtime builds a **globalToFinal** table mapping a global ID to a **position in the final joint array (final joint index)**. The cloth parentID (global) passes through this table to become the actual driver joint.

### 3.1 Computation (observed/inferred) [HYPOTHESIS]
- The game (and the reference viewer) iterate the skeleton joints, filling `globalToFinal[localIDToGlobalID[idx]] = jointIndex`, where jointIndex is the order joints are added (0,1,2,…).
- That is, **globalToFinal[global] = the position that bone has in the final joint array**.

### 3.2 Measured relation in a self-contained g1m [VERIFIED]
- In the (nearly) self-contained g1ms that are custom-authoring targets, the relation **`globalToFinal[l2g[X]] = X` (returns to local bone X)** was confirmed by measurement.
  - Verified: in one working cloth, `parentID=75=l2g[68]` → game driver bone = local bone68. The position of `world[bone68]·CP` matched that cloth's display-mesh position within **0.6 units**. (Details in [06] forward model.)
- This relation is the basis for **cloth parentID must be global (l2g[local])**. (Full derivation in [06] · [07].)

> **[UNSOLVED/caution]:** How globalToFinal is exactly merged for "real game costumes" that reference an external base skeleton was not fully verified. For our authoring-target g1m, the relation above held.

---

## 4. G1MM — inverse bind matrices [VERIFIED]

- G1MM is an array of 4x4 **inverse bind matrices**.
- Indexed by **G1MMIndex**. The G1MMIndex is a **permutation of local bone order** (1:1 with local index but possibly reordered).
- Relation: **`inv(G1MM[G1MMIndex]) = world[bone]`** (the inverse of the inverse-bind = that bone's bind-pose world matrix). [VERIFIED]
  - With this relation you can recover a bone's world bind matrix from G1MM.
- Skinning (vertex deformation) obtains bone matrices through this G1MMIndex. That is, **the joint palette's G1MMIndex field determines the actual skinning matrix** (→ [05]).

### 4.1 Two paths to a bone's world matrix [DERIVED]
1. Compute directly from the G1MS bone hierarchy by parent accumulation (`world[b]`).
2. From G1MM as `inv(G1MM[G1MMIndex(b)])`.
   The two paths must agree (in the bind pose). Useful for verification.

---

## 5. oid — the bone-name table [VERIFIED]

The `oid` file/data gives **global bone ID → name** (essential for debugging/analysis).

### 5.1 Structure (type2) [VERIFIED]
```
oid:
  maxJointIndex (u32)                       # max global index
  repeat: (global_idx u32, name_hash u32, test u32)   # 12-byte triples
```
- Each triple gives a global bone index and a **name hash**. A separate hash→string table yields the actual name.
- Example names: `SK_Spine02` (spine), `RF_L_ForeArm_01` (left forearm), `RF_L_UpLeg_01` (left thigh), `SK_L_Shoulder`, `SK_Hips`, `SK_L_Leg`, etc.
  - **`SK_` prefix = main skeleton bone**, **`RF_` prefix = rig/twist (follow) bone**. [DERIVED]

### 5.2 oid ↔ jil consistency [VERIFIED]
- **The set of valid globals in oid == the set of valid globals in jil** was confirmed (same skeleton). [VERIFIED]
- So oid lets you confirm "global 79 = RF_L_UpLeg_01," and that agrees with jil's l2g/g2l.

---

## 6. Practical summary (from the cloth-authoring view)

- The cloth **parentID is a global ID**. **local driver bone = g2l(parentID)**.
- When making a new cloth, for the **local driver bone X** the user picked, you must write **parentID = l2g(X)**.
- **CPs are encoded in the `world[local bone X]` frame**, and the game places them via `world[globalToFinal[parentID]] = world[X]` → **frames match**. (→ [06] forward model verifies the whole thing.)

The next document covers mesh, vertices, and joint palettes (skinning) → [05](05_g1m_geometry.md).
