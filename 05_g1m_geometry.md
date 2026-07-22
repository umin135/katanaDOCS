# 05. g1m Geometry (G1MG · vertex buffer · joint palettes · mesh groups)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

**G1MG** is the chunk holding the actual render geometry, as a bundle of **sections**. Especially important for cloth authoring are the **submesh**, the **joint palette**, and the **mesh groups**.

---

## 1. G1MG section list [VERIFIED]

The G1MG inner has sections after a header. Each section is identified by a **section ID**.

| Section ID | Contents |
|---|---|
| **0x10004** | **VertexBuffer** — raw vertex data (multiple VBs) |
| **0x10005** | **Layout** — vertex semantic layout (which field at which offset/type) |
| **0x10006** | **JointPalettes (bone maps)** — the (G1MMIndex, physicsIndex, jointIndex) palettes a submesh uses |
| **0x10007** | **IndexBuffer** — triangle indices (multiple IBs) |
| **0x10008** | **Submesh** — submesh entries (stride 0x38) |
| **0x10009** | **MeshGroups** — submesh grouping + **ext_id (cloth/physics link)** |

---

## 2. Submesh (0x10008) [VERIFIED]

A submesh entry is **stride 0x38 bytes**; the section has a leading 4-byte count then the entries (`base=4`).

### 2.1 Measured entry fields (offset within the entry)
| off | field | meaning |
|---|---|---|
| 0x04 | vb_ref | which VertexBuffer index this uses |
| **0x08** | **bmap** | **which joint palette index** (which palette of 0x10006) |
| 0x0C | matID (palID) | material / palID |
| 0x14 | (mat dup) | material duplicate |
| 0x18 | (mat dup) | material duplicate |
| 0x1C | ib_ref | which IndexBuffer index this uses |
| 0x28 | vb_start | start vertex within the VB |
| 0x2C | numv | vertex count |
| 0x30 | ib_start | start index within the IB |
| 0x34 | numi | index count |

- **Material is stored in triplicate at 0x0C · 0x14 · 0x18.** To reflect it in rendering you must change **all three** (changing only 0x18 is ignored). [VERIFIED]
- **To turn a submesh off in rendering, set numi (0x34) = 0.** [VERIFIED]

---

## 3. Joint palette (0x10006) [VERIFIED]

Holds the **bone list used for skinning**, per palette.

### 3.1 Structure
```
0x10006 inner:
  npal (u32)                     # palette count
  repeat npal times:
    cnt (u32)                    # entry count of this palette
    repeat cnt times:
      (G1MMIndex u32, physicsIndex u32, jointIndex u32)   # 12 bytes
```

### 3.2 Meaning of the 3 fields [VERIFIED]
- **G1MMIndex** — index into the G1MM inverse bind matrices. **Determines the actual skinning matrix.**
- **physicsIndex** — the **physics (cloth) marker.** If `≠0`, that palette is a **cloth physics palette**, and the game recognizes "meshes using this palette are physics (cloth) meshes." If `=0`, an ordinary skinning palette. (This is the core of cloth authoring — [06] · [07].)
- **jointIndex** — the bone ID. Observed as **local bone in a display palette**, and **global bone (= parentID) in a cloth physics palette**. [VERIFIED]

### 3.3 Vertex → palette skinning [VERIFIED/DERIVED]
- A vertex's blend index (the `i1` field per layout) points at a palette slot.
- **Ordinary/rivet vertices**: encoded as `i1 = 3 × (palette slot index)` (the game interprets //3). slot → G1MMIndex → matrix. [VERIFIED]
  - i.e. a rivet vertex's `i1` first byte = `3 × slot`.
- (**Cloth sim vertices** are interpreted differently — see [06] §vertex semantics.)

---

## 4. MeshGroups (0x10009) and ext_id [VERIFIED]

Mesh groups group submeshes and hold the **cloth/physics link (ext_id)**.

### 4.1 Structure (observed)
- After the header (block counts) come the entries. Each entry (variable length):
  - `@0x10` : flag (1 = cloth family)
  - **`@0x14` : ext_id** (external/physics link ID)
  - `@0x18` : count of following submesh indices (idxcnt)
  - then `idxcnt` submesh indices (u32)

### 4.2 ext_id encoding [VERIFIED]
The ext_id expresses which physics (cloth) entry a display mesh links to:

| ext_id range | link target | formula |
|---|---|---|
| 0 ~ 9999 | NUNO1 entry | ext = 0·10000 + idx |
| 10000 ~ 19999 | NUNV entry | ext = 1·10000 + idx |
| **20000 ~ 29999** | **NUNO3 entry** | ext = 2·10000 + idx |
| 0xFFFFFFFF | (no link, ordinary mesh) | — |

- **A cloth display mesh links to NUNO3 (ext=20000+idx).** [VERIFIED] NUNO1 has no display mesh (it exists only as a physics grid — [06]).

### 4.3 Insertion rule [VERIFIED]
- A new cloth display entry must be inserted at the **end of block1 (the same tier as sub0)** to render. Putting it in block2/another tier gives a different render context and it is **invisible.** [VERIFIED]
- ext_id = **20000 + NUNO3 index** (no shift).

---

## 5. VertexBuffer (0x10004) / Layout (0x10005) [VERIFIED]

- **0x10004**: after `num_vb`, each VB comes as a `(0, stride, numv, 0)` header + raw vertex data.
- **0x10005**: the semantic layout corresponding to each VB (which vertex attribute at which offset/data type). You need the layout to interpret a vertex buffer.
- A vertex's concrete semantics (position/normal/UV/blend/cloth fields) are defined by the layout; the **special semantics of cloth vertices** (cpW1/cpW2/i1 etc.) are covered in [06] §vertex semantics.

---

## 6. Where this doc meets cloth

Concepts that recur in cloth authoring, summarized up front:

- **Submesh (0x10008)**: the cloth display mesh. Its `bmap` (palette index) — which palette it uses — is decisive.
- **Joint palette (0x10006)**: `physicsIndex` must be the driver bone for the game to recognize a physics mesh. Otherwise, a spike.
- **Mesh group (0x10009)**: links display↔physics via `ext_id = 20000 + NUNO3 index`.

How these three interlock to drive a cloth is covered fully in the next document → [06_g1m_cloth_system.md](06_g1m_cloth_system.md).
