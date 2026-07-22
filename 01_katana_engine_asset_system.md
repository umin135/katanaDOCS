# 01. KatanaEngine Asset System (RDB / RDX / FDATA)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

KatanaEngine (Koei Tecmo) manages game assets in two layers: an **index layer** and a **data layer**. This document covers the binary formats of those two layers and the flow by which a single asset is loaded.

---

## 1. Two-layer overview [VERIFIED]

```
[Index layer]
  root.rdb   : file_ktid -> (fdata_id, offset, size) etc. location metadata
  root.rdx   : fdata_id  -> filename hash (for filename resolution)

[Data layer]
  0x????????.fdata : a container of IDRK blocks (zlibext-compressed asset payloads)
```

- The **index (RDB/RDX)** knows "which asset lives in which .fdata, and where."
- The **data (FDATA)** holds the actual asset bytes (models/textures/materials) as compressed blocks.

Multiple .fdata files exist, each identified by a 16-bit ID (`fdata_id`, 0..65535). [VERIFIED]

---

## 2. Asset load flow [DERIVED]

The logical flow by which the game loads one asset:

```
LoadAsset(file_ktid):
  1) Look up file_ktid in RDB -> obtain (fdata_id, offset, size)
  2) Resolve fdata_id -> filename (hash) via RDX -> determine the 0x????????.fdata file
  3) Read the IDRK block at offset inside that .fdata
  4) zlibext-decompress the IDRK payload -> original asset bytes
```

Because of this flow, you can swap an asset **without touching the original .fdata** — just redirect an RDB entry's location metadata to a different .fdata (→ §6 override strategies).

---

## 3. RDB / RDX format [VERIFIED]

### 3.1 Common
- All integers are **little-endian**.
- RDB entries are **4-byte aligned** (stride is a multiple of 4).

### 3.2 RDB (root.rdb)
- The header has a **`file_count` (entry count)** field. Adding an entry requires updating it. [VERIFIED]
- Each entry holds the location metadata for one asset. Two observed location encodings exist:
  - **Location32** — a 32-bit location-field family
  - **Location40** — a 40-bit location-field family
  (Determine which encoding by game/version, then parse accordingly.)
- An entry roughly holds: **file_ktid** (per-asset unique hash), **type_ktid** (type-class hash), **fdata_id** (u16, which .fdata), **offset/size** (location/size within that .fdata), plus flags.
- **Binary-safety rule [DERIVED]:** When **editing** an entry, do not change `entry_size` (the entry-size field). Overwrite only interior fields. Only when **adding** an entry do you update the header `file_count`.

### 3.3 RDX (root.rdx)
- **RDX has no separate header.** [VERIFIED] (Unlike RDB, the mapping data comes immediately.)
- It holds the `fdata_id` → filename-hash mapping. That hash determines the actual `0x????????.fdata` filename.

---

## 4. FDATA / IDRK format [VERIFIED]

A `.fdata` file is a sequence of **IDRK blocks**.

### 4.1 IDRK block structure
- Each block starts with the magic `IDRK`.
- The block header has size fields and a `param` field.
- **★The most important measured fact:** **the payload (compressed data) location must be computed as "compressed_size back from the block end."**
  - Computing the payload offset from `param` is **wrong** (param does not point at the payload start). [VERIFIED]
  - That is, `payload_start = block_end - compressed_size`.

### 4.2 zlibext compression
- The IDRK payload is a **full zlib stream** (zlibext). It decompresses with standard zlib inflate. [VERIFIED]

### 4.3 No integrity check
- **The engine does not verify chunk hashes/CRC of IDRK blocks.** [VERIFIED, DOA6 LR]
  - So you do not need to recompute checksums after changing asset bytes.
  - This fact greatly simplifies modding.

---

## 5. KTID (asset hash) [VERIFIED/UNSOLVED]

- **FileKtid** — the per-asset unique 32-bit hash. Made from the original **filename**.
- **TypeKtid** — the type-class hash. TypeKtid determines the extension / asset kind (a TypeKtid→extension table exists).

### 5.1 The name-hash problem [UNSOLVED]
- **The filename → FileKtid hash algorithm was never fully cracked.** Observation suggests it is not a simple linear/CRC form but nonlinear. [VERIFIED]
- Workaround: keep a known **filename↔FileKtid table (CSV etc.)** and look it up by table, rather than cracking new hashes. [DERIVED]
- You need this hash to add a genuinely "new-filename" asset, but not to redirect/replace an existing asset (reuse the existing FileKtid).

---

## 6. Override (replacement) strategies [VERIFIED/HYPOTHESIS]

Four approaches to replacing an asset were considered. The key is **avoiding destruction of the original**.

| Strategy | Method | Status |
|---|---|---|
| **A** | In-place edit of the original .fdata | Destructive → avoid |
| **B** | **RDB entry redirect** — leave the original untouched, redirect the RDB location metadata to a custom .fdata | **[VERIFIED] confirmed in DOA6 LR** |
| **C** | last-wins duplicate append — add the same-KTID entry again at the end so the later one wins | [HYPOTHESIS] unverified (unneeded once B is settled) |
| **D** | other | unexplored |

### 6.1 Strategy B (recommended) procedure [VERIFIED]
1. zlibext-compress the custom asset bytes into an **IDRK block**.
2. Place that block in a **new .fdata** (or mods.fdata).
3. Assign an **unused fdata_id (u16)** to the new .fdata and register it in RDX/system.
4. **Edit the RDB entry** of the asset to replace to point (fdata_id, offset, size) at the new block. (Do not touch `entry_size`.)
5. Back up the original .fdata/RDB (e.g., `.kmbak`) and always patch by **idempotent recomputation from the clean original**.

- **Result [VERIFIED]:** In DOA6 LR, redirecting a g1t texture to a custom mods.fdata → confirmed reflected in-game. Measured with a magenta test texture.

---

## 7. Safety-rule summary [DERIVED]

- Integers are all **little-endian**, RDB entries are **4-byte aligned**.
- Do not edit originals (rdb/rdx/fdata) in place. Write patches to a separate output, back up the original.
- Mod application is always **idempotent recomputation from a clean original**.
- On **editing** an RDB entry keep entry_size unchanged; on **adding** update file_count.
- `fdata_id` is u16 → assign an **unused ID** when registering a new .fdata.
- No IDRK integrity check, so no checksum recomputation needed.

---

## 8. How this layer relates to g1m

RDB/RDX/FDATA are **the vessel for holding and finding assets**. The representative assets held inside are **g1m (3D model)**, **g1t (texture)**, **kts (material)**, etc. How one character costume cross-references those assets is covered in [02_asset_reference_chain.md](02_asset_reference_chain.md); the internal structure of g1m itself starts at [03_g1m_container_and_chunks.md](03_g1m_container_and_chunks.md).
