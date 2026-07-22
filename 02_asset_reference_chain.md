# 02. Costume Asset Reference Chain

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

A character's costume is **not one asset but a bundle of several**: the model (g1m), textures (g1t), materials (kts), and the reference data linking them. This document covers how those references connect, and **how far they can be resolved offline**.

---

## 1. Big picture: references of a composite asset [DERIVED]

```
Costume
  └─ G1M (model: skeleton + mesh + physics)
       └─ (KTID/slot reference) ──▶ G1T (texture bundle)
       └─ (material reference)  ──▶ KTS (material / shader params)
  └─ kidsobjdb / object DB (object ID ↔ asset KTID reverse index)
```

- **G1M** holds model geometry but does not hold textures directly. Instead it references G1T through **texture slots**.
- **G1T** is a bundle of multiple textures (albedo/normal/…).
- **KTS** defines the material (which shader, which texture-slot layout). **To actually load a texture you need the KTS.** [VERIFIED]
- These references are connected through a reverse index like **kidsobjdb** (or the game's object DB).

---

## 2. G1M → texture reference chain [DERIVED]

Swapping just the model differs in difficulty from swapping the texture layout too:

- **Model-only swap**: just replace the G1M (reuse existing texture slots).
- **Swap the texture layout too**: you must also change the **whole reference chain** of **G1M → (slot mapping) → G1T → material (KTS)**.

This chain is connected through an object DB like kidsobjdb, as "this mesh's this slot = this texture." Slot order/count **differs per material (KTS)** (§4).

---

## 3. Costume string → asset map (hash-free resolution) [VERIFIED]

- Starting from a **costume string (identifier)** that names one costume, there exists a path to resolve the **full asset map** that costume uses **hash-free**. [VERIFIED, DOA6]
  - That is, bypassing the unsolved name-hash (filename→FileKtid), you can follow explicit mappings in the game data to get costume→asset set.
- This hash-free path is the default entry point when "understanding a whole costume."

---

## 4. Material (KTS) and texture loading [VERIFIED]

- **The KTS format magic is `GSTK`.** [VERIFIED]
- KTS determines the purpose of each texture slot via a **shader-category registry**. Observed categories, e.g.: **1 = albedo (alb), 3 = normal/height (nmh)**. [VERIFIED]
- **To actually load/interpret textures you need the KTS** — the material defines "which texture in which slot." [VERIFIED]
- **Slot order differs per material (shader type).** [VERIFIED] So transplanting one material's slot layout onto another material breaks it.

---

## 5. The singleton object-DB constraint [VERIFIED/UNSOLVED]

The fundamental constraint hit when trying to port a new bundle (assets of another costume/game) into ktmod:

- **Resolving object ID (objID) → g1t texture requires the game's singleton DB. You cannot pin a g1t from an objID alone offline (outside the game).** [VERIFIED]
  - That is, part of the references depend on a **runtime game DB (singleton)**. Static files alone cannot fully resolve them.
- **File lookup can be worked around via `HashByName` (finding a file by name hash).** [VERIFIED] — Without the DB you can find "the file of this name," but the semantic link objID↔asset needs the DB.

Because of this constraint, "automatically pulling in and assembling arbitrary external assets" is limited with pure-offline tools.

---

## 6. Mesh hash registry = Character.sid [VERIFIED]

Per character, the registry that decides **which mesh is rendered / which material it uses / which physics (soft·rigid) drives it** is **`Character.sid`**.

- **`Character.sid` decides, per mesh hash, the render gate + material + soft/rigid driving.** [VERIFIED, confirmed in-game]
- That is, **merely putting a new custom mesh into the g1m is not enough** — you must **register that mesh's hash into the sid** for the game to recognize and drive it.
- sid patching enables **registering a new mesh, disconnecting soft**, etc. [VERIFIED]
- This constraint is crucial for authoring "fully custom meshes." See [08_g1m_textures_materials_physics.md](08_g1m_textures_materials_physics.md) §sid for details.

---

## 7. Summary: what works / doesn't offline

| Task | Offline possible? | Basis |
|---|---|---|
| Replace an existing asset via RDB redirect | ✅ | [01] strategy B [VERIFIED] |
| Costume string → asset set resolution | ✅ | hash-free path [VERIFIED] |
| Find a file by filename | ✅ | HashByName [VERIFIED] |
| Semantic objID → g1t resolution | ❌ | needs the singleton game DB [VERIFIED] |
| New filename → new FileKtid generation | ❌/△ | name hash uncracked [UNSOLVED] |
| Game recognizing/driving a new mesh | △ | needs Character.sid registration [VERIFIED] |

From the next document on, we go deep into the central asset of this reference chain: **the g1m model format itself.**
