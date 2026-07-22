# 08. Textures · Materials · Soft Physics & g1m Decode (g1t / kts / sid / SOFT)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

Besides cloth (NUNO), this covers the remaining formats involved in completing a character costume, and the measured corrections when decoding a g1m into something human-viewable (e.g. glTF). (These were not dug as deeply as cloth, so there are relatively more [HYPOTHESIS] items.)

---

## 1. g1t — texture bundle [VERIFIED]

- g1t is a container holding multiple textures (albedo/normal/…).
- **The header and format codes were pinned by public reference material + binary measurement.** Some public labels were wrong and corrected by measurement. [VERIFIED]
- **Compression format**: BC (block compression) family. **BC4/BC5/BC6H need post-processing** (channel reconstruction etc. on decode). [VERIFIED]
- **DOA6 swizzle [VERIFIED]**: DOA6 textures are observed as **no-swizzle** (format code 101/101, exSwizzle=0). i.e. texels are laid out linearly, so no extra deswizzle is needed.
- **To actually load/interpret textures you need the material (KTS)** (KTS defines which texture in which slot) — [02] §4.

---

## 2. kts — material [VERIFIED]

- **Magic `GSTK`.** [VERIFIED]
- Defines each texture slot's purpose via a **shader-category registry**. Observed: **1 = albedo (alb), 3 = normal/height (nmh)**, etc. [VERIFIED]
- **Slot order differs per material (shader type).** [VERIFIED] Transplanting one material's slot layout onto another breaks it.
- When replacing/adding textures, you must stay consistent with the material's slot definition for the game to load the right textures.

---

## 3. sid — mesh hash registry (Character.sid) [VERIFIED]

A per-character singleton. It decides, **per mesh hash, what is rendered / material / physics driving**.

- **`Character.sid` decides, per mesh hash, the render gate + material + soft/rigid driving.** [VERIFIED, confirmed in-game]
- That is, **even if you put a fully custom mesh into the g1m, if its mesh hash is not in the sid the game will not recognize/drive it.**
- **sid patching** enables registering a new mesh, disconnecting soft, etc. [VERIFIED]
- **Implication:** a "fully custom g1m" pipeline must necessarily include **sid patching**, not just g1m editing.
  - Envisioned as an integrated pipeline that also carries a fresh mesh hash + growth (registry expansion) and an asset-location (RDB) update. [HYPOTHESIS: the integrated pipeline is incomplete]

---

## 4. SOFT — soft-body physics [VERIFIED/HYPOTHESIS]

- Separate from NUNO (cloth), the **SOFT chunk holds soft-body physics (e.g. breast jiggle)**.
- Structure (observed): a node grid (pos/rot/influences) + physics parameters (Unk1) + parentID. [VERIFIED]
- **Byte-exact parse↔rebuild** was confirmed (capture → lossless reconstruction). [VERIFIED]
- A round-trip schema exposing the node grid · parameters for editing in Blender etc. was envisioned. [HYPOTHESIS: the authoring pipeline is partly incomplete]

---

## 5. g1m decode (e.g. glTF) measured corrections [VERIFIED]

Points repeatedly gotten wrong when viewing a g1m or moving it to another tool (e.g. glTF). Pinned by measurement:

- **Layout/VB selection**: you must **pick exactly** the VB (vb_ref) a submesh references and the layout corresponding to it, or vertex semantics break. Using an arbitrary layout breaks position/normal. [VERIFIED]
- **Skinning indices**: vertex blend indices are resolved to bones **through the joint palette**. **Do not use jil directly for skinning** (jil is for local↔global mapping). [VERIFIED]
- **Submesh offsets**: apply vb_start/ib_start/numv/numi (submesh fields) exactly, or triangles are wrong. [VERIFIED]

---

## 6. g1m edit risk tiers [VERIFIED/DERIVED]

Risk per chunk when editing a g1m (observed):

- **Low risk**: submesh fields (material · turning off numi), joint-palette add (at the end), NUNO entry add (at the end).
- **Medium risk**: G1MF count updates (exact offset required — [03] §3). VB/IB append.
- **High risk / pitfalls**:
  - **The G1MF count pitfall** — updating counts via a search scheme touches the wrong field on value collision. Use only exact offsets.
  - **No mid-insertion** — collapses index-based references (ext_id, palette index). Always insert at the end.
  - **Material triple-store** (0x0C/0x14/0x18) — update all three.
- **Material type field**: a value with the character of `nMtrID` (material type, e.g. 0x10003) is observed in the submesh/material family. [HYPOTHESIS: details unconfirmed]

---

## 7. The fully-independent g1m goal [HYPOTHESIS]

A long-term goal of "**remove original dependency — regenerate every chunk byte-identically**" was envisioned.

- If every chunk (G1MF/G1MS/G1MM/G1MG/NUNO/SOFT), when parsed→regenerated, is byte-identical to the original, it becomes the trust basis for arbitrary edits. [HYPOTHESIS: partly achieved]
- Envisioned as the foundation on which physics parameter (NUNO/SOFT) exposure to the authoring tool, arbitrary-resolution cloth, VB-layout synthesis, etc. sit. [HYPOTHESIS]

---

## 8. Where this doc sits

- The g1t/kts/sid/SOFT here are **not verified as deeply as cloth (NUNO).** The cloth material ([06] · [07]) is the settled core of this doc set; the writing here is relatively more circumstantial/partial.
- To fully customize a whole character costume, **g1m (model) + g1t (texture) + kts (material) + sid (registration) + RDB (asset location)** must all interlock (see [02]). Full automation of that is incomplete.
