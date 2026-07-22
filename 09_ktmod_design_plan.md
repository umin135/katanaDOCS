# 09. ktmod Design Plan (initial · unstable — for reference)

> Standalone document. References no external file. Verification tags: [VERIFIED]/[DERIVED]/[HYPOTHESIS]/[UNSOLVED].

> ## ⚠️ This entire document is an "initial plan" and is unstable
> The ktmod package format · editor · manager · role split described below are **all an initial design intent, not a settled spec, and can change at any time.** Do not implement as-is. In particular:
> - **The ktmod `Content/` mechanism (holding assets symbolically) has been deferred indefinitely.** The writing in §3 is only a directional record, not a plan being pursued.
> - Clearly distinguish the **format facts ([VERIFIED])** of the earlier docs (01~08) from the **plan ([HYPOTHESIS])** in this one.

---

## 1. What ktmod is (intent) [HYPOTHESIS]

- **ktmod** = the custom mod package / authoring toolchain this project envisioned.
- Goal: let a mod maker build a mod with just **assets (model/texture/material) and a simple manifest**, without needing to know the engine internals ([01]~[08]).
- Target: on top of [01]'s RDB/RDX/FDATA · strategy B (redirect), layer assets without destroying the original.

---

## 2. Package format (intent) [HYPOTHESIS]

- **`.ktmod`** — a ZIP-compatible container.
- Composition (draft):
  ```
  mod.json          # manifest (mod name/version/target game/asset list)
  assets/           # actual assets (or symbolic references — §3)
  ```
- The manager reads the `.ktmod` and applies it to the game via strategy B (RDB entry redirect).

---

## 3. Content/ symbolic design (deferred indefinitely) [HYPOTHESIS · deferred]

> **The mechanism in this section has been deferred indefinitely.** Left only as a record.

- Idea: hash-referenced assets (those pointing at other assets by KTID/objID) are held in the package **always as symbolic JSON (`@name.ext` form)**, and **the manager resolves them to objID/FileKtid at install time**.
- `Content_Legacy` = raw redirect without symbols (simple replacement).
- Deferral reason (circumstantial): objID→asset resolution **depends on the game singleton DB** ([02] §5), making offline auto-resolution hard, and the name hash ([01] §5) is unsolved, so the symbolic-resolution scheme is unstable.

---

## 4. Three-layer role split (intent) [HYPOTHESIS]

An idea to divide authoring into three layers:

| layer | role (intent) |
|---|---|
| **Addon (3D tool, e.g. Blender)** | Author g1m (mesh/skeleton/physics) + geometry sidecar (grp) + material slots (mtl). **Blender handles only slot count/index**, mirroring meshType→sid. |
| **Editor (dedicated GUI)** | Edit material instances (= shader type + texture layout). Manage the project (folder-based, `project.ktproj`) + Build→`.ktmod`. |
| **Manager** | Glue sid + kts/mpr/g1t and apply to the game via RDB redirect. |

- The split has the addon do only "slot count/index," while actual material/texture gluing is the editor's/manager's job. [HYPOTHESIS]

---

## 5. Editor / project / launcher (intent) [HYPOTHESIS]

- **Folder-based project** (`project.ktproj`) + **Build → `.ktmod`**.
- **Launcher**: New / Open / Recent.
- UX direction (draft): UE-style docking, a unified Content browser, textures as TGA/DDS.
- Authoring scope (draft): **full authoring for DOA6LR only**, other games via Legacy (simple replacement) only.

---

## 6. Implementation stack (intent) [HYPOTHESIS]

- **C# / .NET (LTS 8+) + Avalonia (11)**, cross-platform (Win/Linux), single executable.
- Structure (draft): a UI-agnostic core + CLI + GUI + tests. GUI in MVVM.
- Binary handling: little-endian span-based parsing, memory-map for large .fdata.

---

## 7. What is settled and what is not

| item | status |
|---|---|
| RDB/RDX/FDATA format · strategy B | **[VERIFIED] settled** ([01]) |
| g1m format · cloth authoring | **[VERIFIED] mostly settled** ([03]~[07]) |
| **ktmod package/editor/role split** | **[HYPOTHESIS] initial plan, unstable** |
| **Content/ symbolic** | **deferred indefinitely** |
| Full auto-assembly (sid+kts+g1t+rdb) | **incomplete** |

**In short:** the contents of this doc (09) are a record of "we intended to go this way." What to actually trust is the format facts of [01]~[08]; what tools to layer on top is open.
