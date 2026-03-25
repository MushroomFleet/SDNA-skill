# SDNA Methodology — Technical Grounding

## Overview

Blender's `.blend` file format is one of the most elegant file-format designs in open-source software. Its secret is a system called **SDNA** (Struct DNA) — a self-describing binary schema embedded directly inside every `.blend` file. This document covers how SDNA works, how it enables backward and forward compatibility, and how it relates to Blender's higher-level RNA layer.

---

## How DNA Works

### The Core Design Trick

Blender's `.blend` files are **not** a traditional serialized format with fixed schemas or versioned chunks. Instead, every data block in a `.blend` file — meshes, scenes, objects, materials, and so on — is essentially a **raw memory dump** of Blender's internal C structs as they existed in RAM at save time.

### The `makesdna` Tool

Before saving, Blender runs a tool called **`makesdna`** that:

1. Scans all DNA-defined structs in the Blender source code (`source/blender/makesdna/`).
2. Generates a compact **binary SDNA block** (the `DNA1` block, found near the end of the file).
3. Embeds this block directly into the saved `.blend` file.

The SDNA block describes:
- Every struct used in the file
- Every field name within each struct
- Every type name
- Every array and pointer size

### Result: The File Carries Its Own Blueprint

The `.blend` file is **self-descriptive**. It does not rely on the loading version of Blender already knowing the exact memory layout from years ago. The file literally contains its own genetic code — hence the name **DNA**.

---

## Backward Compatibility — Opening Old Files in New Blender

When a newer version of Blender opens an older `.blend` file, it follows a precise process:

### Step 1 — Read the File Header
Parse pointer size, endianness, and the version of Blender that wrote the file.

### Step 2 — Parse the Embedded SDNA
Extract the `DNA1` block from the old file. This gives Blender the **exact struct layout** that existed when the file was saved.

### Step 3 — Compare Old DNA to Current DNA
Blender compares the file's embedded SDNA against its own current compiled DNA (its in-memory struct definitions).

### Step 4 — Name-Based Field Mapping
For each data block, Blender maps fields **by name**, not by byte offset. This means:
- Reordered fields still load correctly.
- Renamed fields can still be matched (with RNA bridging).
- Brand-new fields (added in newer Blender) are initialised with sensible defaults.
- Removed fields are skipped cleanly.

### Step 5 — Incremental Versioning Patches
Blender runs **versioning functions** that patch data step-by-step from the file's original version up to the current one. These functions (e.g. `blo_do_versions_400()` in the source) handle:
- Deep structural changes
- Deprecated features
- New default values
- Migrated data formats

This is found in: `source/blender/blenloader/`

### Step 6 — Pointer Relocation
The file stores raw RAM addresses from the time it was saved. Blender remaps all old memory pointers to valid new ones.

### Outcome
Because the old struct definitions are embedded in the file itself, even a `.blend` file from **Blender 1.00 (pre-2000)** can be opened in modern Blender. The loader reads the ancient DNA, runs all accumulated versioning patches, and upgrades the data on the fly — with no data loss for core features unless something was deliberately deprecated over many years.

---

## Forward Compatibility — Opening New Files in Old Blender

Forward compatibility is more limited, but more resilient than most formats:

- Old Blender reads the embedded SDNA and **skips unknown structs and fields** introduced in newer versions.
- The file opens without crashing; only new features are lost.
- Major breakage typically only occurs across large release jumps (e.g. `4.x → 5.0`).
- In those cases, the previous **LTS (Long-Term Support) release** often acts as a bridge converter.

---

## DNA + RNA — The Full Picture

Blender's compatibility architecture is a two-layer system:

| Layer | Role |
|-------|------|
| **DNA** | Low-level file storage — the self-describing binary structs embedded in `.blend` files |
| **RNA** | High-level reflection layer built on top of DNA — provides a stable API for Python add-ons, the UI system, and external tooling |

### Why RNA Matters

RNA gives Blender a **stable public API** even when the underlying DNA structs change. For example:
- A struct field can be renamed internally in C without breaking Python scripts.
- RNA maps the old API name to the new struct field transparently.

This combination is why Blender can completely rewrite core systems — the dependency graph, the mesh format, the rendering pipeline — while still opening files from **over 20 years ago**.

---

## Source Code Locations

| Path | Contents |
|------|----------|
| `source/blender/makesdna/` | DNA struct definitions scanned by `makesdna` |
| `source/blender/blenloader/` | File loading, SDNA parsing, pointer relocation, versioning patches |
| `blo_do_versions_*()` functions | Incremental versioning logic per release |

---

## Key Properties of the SDNA System

- **Self-describing** — the file carries its own schema; no external registry needed
- **Name-based mapping** — resilient to field reordering and struct evolution
- **Incremental versioning** — step-by-step upgrade patches accumulate over releases
- **Graceful degradation** — unknown blocks in forward compatibility are skipped, not fatal
- **Pointer-size aware** — handles 32-bit vs 64-bit and endianness differences at the header level

---

## Further Reading

- [Blender Developer Docs — DNA](https://developer.blender.org/) — short, technical overview
- **Blend File Compatibility Guidelines** — official policy on versioning and compatibility commitments
- **"Understanding Blender's .blend File Format and DNA/RNA"** — in-depth breakdown with SDNA block layout and code snippets

---

> SDNA is one of the most elegant file-format designs in open-source software. The file carries its own DNA, Blender performs name-based mapping and versioning on load, and new engine rewrites don't break old saves.
