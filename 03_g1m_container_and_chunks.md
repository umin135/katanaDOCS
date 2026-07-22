# 03. g1m Container & Chunk Structure

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

`g1m` is a **chunk container** holding one 3D model. Skeleton, inverse-bind matrices, geometry, and physics (cloth/soft) each go in as a chunk.

---

## 1. File structure [VERIFIED]

```
g1m file
 ├─ file header (magic / version / chunk count etc.)
 └─ N chunks  (each: signature + version + size + inner data)
```

- All integers are **little-endian** (except that the chunk version field is sometimes read as a big-endian 4-byte value, handled specially by the parser). [VERIFIED]
- Each chunk consists of a **signature (4-byte ASCII)**, a **version**, a **size**, and the **inner (actual payload)**.
- **Version example: `0x39323030`** = ASCII `"9200"` (observed in the DOA6 family). [VERIFIED] The version governs the internal padding size of some chunks (NUNO especially).

---

## 2. Major chunk list [VERIFIED/DERIVED]

| Signature | magic (u32 BE) | Role | Detail doc |
|---|---|---|---|
| **G1MF** | 0x47314D46 | File info = **various count tables** (vertex count, submesh count, bone-map count, NUNO counts, etc.). Adding data to another chunk requires updating counts here. | this doc §3 |
| **G1MS** | 0x47314D53 | **Skeleton**: bone hierarchy + **jil** (local↔global ID mapping). | [04](04_g1m_skeleton_and_matrices.md) |
| **G1MM** | 0x47314D4D | **Inverse bind matrix** array. Used for skinning. Indexed by G1MMIndex. | [04](04_g1m_skeleton_and_matrices.md) |
| **G1MG** | 0x47314D47 | **Geometry**: a bundle of sections (vertex buffer / layout / index / submesh / joint palette / mesh group). | [05](05_g1m_geometry.md) |
| **NUNO** | 0x4E554E4F | **Cloth simulation**: NUNO1/NUNO3 (control-point grids), NUNO4 (driver table), NUNV (vertex cloth). | [06](06_g1m_cloth_system.md) |
| **SOFT** | (soft) | **Soft-body physics** (e.g. breast jiggle). Node grid + parameters. | [08](08_g1m_textures_materials_physics.md) |
| (others) | — | EXTR (external references), COLL (collision), etc. [HYPOTHESIS: details unverified] | — |

Note: the magics above are the 4-byte ASCII read as a big-endian u32. On disk they appear as ASCII like `G1MG`.

---

## 3. G1MF — the count table [VERIFIED]

G1MF is a flat structure holding many **counts** for the whole g1m. **When you add entries/data to a chunk, you must +update the corresponding G1MF count** or the game will not load correctly.

### 3.1 Offset convention [VERIFIED]
- When discussing G1MF field offsets, the **"struct offset" and the "inner offset" differ by 0x0C.**
  - `inner_offset = struct_offset − 0x0C`
  - (The struct includes signature@0x00 · version@0x04 · size@0x08 · unk@0x0C, but the inner starts past that 12-byte header.)
- The table below is by **inner offset** (the value actually used in code).

### 3.2 Measured major G1MF fields (inner offsets)

| inner off | field | meaning | update on add |
|---|---|---|---|
| 0x28 | num_vb | vertex buffer count | +1 per VB added |
| 0x2C | num_layouts | layout count | +1 |
| 0x30 | num_layout_refs | layout ref count | +1 |
| **0x34** | **num_bone_maps** | **joint palette (bone map) count = npal of section 0x10006** | +count per palette added |
| **0x38** | **num_individual_bone_maps** | **total entry count across all palettes** | += total entries added |
| 0x3C | (layout_refs family) | | |
| 0x40 | num_submeshes | submesh count | +1 |
| 0x44 | num_submeshes2 | submesh count (2) | +1 |
| 0x4C | num_meshes | mesh-group entry count | +1 |
| 0x50 | num_submeshes_in_meshes | | +1 |
| **0x68** | **num_nuno1s** | NUNO1 entry count | +1 per NUNO1 added |
| 0x6C | num_nuno1s_unk4 | (per entry) | +1 |
| 0x70 | num_nuno1s_control_points | total NUNO1 control points | += cpc |
| 0x74 | num_nuno1s_unk1 | total NUNO1 unkcnt (anchors) | += unkcnt |
| 0x78 | num_nuno1s_unk2_and_unk3 | total NUNO1 skip1+skip2 | += (skip1+skip2) |
| **0xE4** | **num_nuno3s** | NUNO3 entry count | +1 per NUNO3 added |
| 0xE8 | num_nuno3s_unk4 | (per entry) | +1 |
| 0xEC | num_nuno3s_control_points | total NUNO3 control points | += cpc |
| 0xF0 | num_nuno3s_unk1 | total NUNO3 unkSection | += unkcnt |

> **Warning [VERIFIED]:** A scheme like "search for a field equal to npal and +1" is dangerous. For instance the num_bone_maps (0x34) value **can coincidentally equal** num_vb, num_submeshes, etc., so the search touches the wrong field. **Always write the exact offset directly.**

---

## 4. Safety rules for chunk edits [DERIVED]

- **Insert entries at the "end" of a section/chunk.** Mid-insertion breaks index-based references (ext_id, palette index, etc.). [VERIFIED]
  - Index-based references point to "the n-th entry," so appending at the end preserves existing indices.
- When inserting data into a chunk, update **that chunk's size field and entry count**, and also update the **corresponding G1MF count**.
- Do not edit originals in place; write to a separate output + back up (`.kmbak`); always **idempotent recomputation from a clean original**.

---

## 5. Next

- Bones and coordinate frames (skeleton, inverse bind matrices, local/global IDs) → [04](04_g1m_skeleton_and_matrices.md)
- Mesh, vertices, skinning, joint palettes → [05](05_g1m_geometry.md)
- Cloth simulation — the core of this doc set → [06](06_g1m_cloth_system.md)
