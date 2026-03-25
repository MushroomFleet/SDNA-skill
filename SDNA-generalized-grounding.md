# SDNA — Generalized Methodology Grounding

## What This Document Is

This document extracts the **design principles** behind Blender's SDNA system and restates them as a **language-agnostic, implementation-agnostic methodology** that can be applied to any project that serializes structured data and needs to survive across versions.

The reference implementation is Blender's `.blend` format (documented in `SDNA-grounding.md`), but nothing here is Blender-specific. The patterns apply equally to game save files, database records, network protocols, configuration formats, asset pipelines, and any other system where structured data must remain readable across time.

---

## The Core Problem SDNA Solves

Any system that persists structured data faces a fundamental tension:

> **The data you write today will be read by a version of your software that does not exist yet.**

Traditional approaches to this problem include:

| Approach | Weakness |
|----------|----------|
| Fixed binary schemas | Break silently when structs change |
| Version numbers only | Require exhaustive migration code per version pair |
| External schema registries | Create external dependencies; rot over time |
| Human-readable formats (JSON, XML) | Verbose, slow, and still break when field names change |
| "Just never change the schema" | Impossible at any meaningful scale |

SDNA's answer is different: **embed the schema in the data itself, then reconcile on load.**

---

## The Five Core Principles

### Principle 1 — Self-Description

> *Every persisted data artifact carries a complete description of its own structure.*

The artifact does not assume the reader already knows its layout. It declares:
- What named fields exist
- What type each field is
- What the size and shape of arrays and references are

This description is written at **save time**, not at read time. The reader's job is to discover the layout from the artifact itself, not from a hardcoded expectation.

**Implementation pattern:** Embed a compact schema block alongside your data — before, after, or interleaved with the data blocks. The schema block is always written first on save and always parsed first on load.

**Anti-pattern:** Assuming the reader shares the writer's compiled struct definitions. This is only safe within a single binary release.

---

### Principle 2 — Name-Based Field Mapping

> *Fields are matched by name across versions, never by byte offset or positional index.*

When the reader reconciles its current schema against the artifact's embedded schema, it resolves fields by their **symbolic name**. This means:

- Fields can be reordered freely between versions.
- New fields added in the current version are initialised to safe defaults.
- Fields that no longer exist in the current version are skipped without error.
- Fields that exist in both are mapped directly, even if their position in memory has changed.

**Implementation pattern:** Your schema block stores field names as strings (or interned string IDs). Your loader iterates the artifact's field list and resolves each one by name lookup against the current schema, not by assuming positional alignment.

**Anti-pattern:** Positional/index-based deserialization (e.g. "field 3 is always the Y coordinate"). A single insertion anywhere breaks everything downstream.

---

### Principle 3 — Incremental Versioning Patches

> *Structural migrations are expressed as a chain of small, ordered, cumulative transformations — not as a single monolithic upgrade.*

Name-based mapping handles the easy cases (added fields, removed fields, reordered fields). But some changes are deeper: a field that was a scalar becomes an array; a nested struct is flattened; a value that was stored in degrees is now stored in radians. These require **active data transformation**, not just schema reconciliation.

The SDNA approach is to maintain a sequence of versioning functions, each responsible for exactly one migration step, applied in order:

```
patch_v1_to_v2(data)
patch_v2_to_v3(data)
patch_v3_to_v4(data)
...
```

The loader reads the artifact's version, then runs only the patches needed to bring it up to the current version. Patches accumulate — they are never collapsed or rewritten retroactively.

**Implementation pattern:** Version your artifacts with a monotonically increasing integer. Maintain a registry of patch functions indexed by version. On load, slice the registry from `artifact_version` to `current_version` and apply each patch in sequence.

**Anti-pattern:** A single `migrate(from_version, to_version, data)` function with a large switch statement. This becomes unmaintainable and discards the incremental audit trail.

**Anti-pattern:** Collapsing old patches into a single "legacy" path. Every patch that ever existed must remain in the chain — removing one breaks any artifact older than the removed step.

---

### Principle 4 — Graceful Degradation for Forward Compatibility

> *A reader encountering data it does not understand must skip it cleanly, not crash.*

When an older reader encounters a newer artifact (one written by a version it predates), it will find structs, fields, or blocks it has no knowledge of. The correct behaviour is:

- **Skip unknown blocks entirely** — log them if useful, but do not halt.
- **Skip unknown fields within known structs** — use the embedded schema to calculate the correct byte offset to advance past them.
- **Load what is understood** — the artifact is partially useful even if newer features are invisible.

This property is not automatic. It must be **designed in** from the start. A reader that uses positional offsets cannot skip unknown fields because it does not know their size. A reader that uses embedded schema sizes can always skip forward correctly.

**Implementation pattern:** Every block or record in the artifact stores its own byte length in the header. Unknown blocks can be skipped in O(1) regardless of content. Field sizes come from the embedded schema, so unknown fields can also be skipped safely.

**Anti-pattern:** Treating an unknown block type as a fatal error. This makes every new writer a breaking change for every old reader.

---

### Principle 5 — The Two-Layer Architecture (DNA + RNA)

> *Separate the storage schema (what is persisted) from the public interface schema (what code and users interact with). Bridge them with a stable mapping layer.*

Blender calls these DNA (storage) and RNA (interface). The names are domain-specific; the principle is universal.

The storage schema evolves freely to match implementation needs. The public interface schema evolves conservatively to preserve compatibility for consumers (scripts, plugins, external tools, users). A mapping layer translates between them.

This separation means:
- Internal refactors (renaming a field, splitting a struct, changing a type) do not break the public API.
- The public API can be stabilised and documented independently of the internal implementation.
- The storage format can be optimised (packed, compressed, reordered) without affecting consumers.

**Implementation pattern:**

```
[Persisted artifact]
        ↓  (SDNA load + versioning patches)
[Internal representation]  ← storage schema (evolves freely)
        ↓  (stable mapping layer)
[Public API / user-facing model]  ← interface schema (evolves conservatively)
```

**Anti-pattern:** Exposing internal storage structs directly as the public API. Any internal change immediately becomes a breaking change for every consumer.

---

## The Load Sequence

Any SDNA-pattern implementation should follow this load sequence:

```
1. Read artifact header
   → Extract: format version, pointer/reference size, endianness, encoding

2. Parse embedded schema block
   → Build: map of { struct_name → { field_name → (type, size, offset) } }

3. Diff embedded schema against current schema
   → Identify: new fields (need defaults), removed fields (skip), changed types (need coercion)

4. For each data block in the artifact:
   a. Identify block type from embedded schema
   b. Map fields by name from artifact layout → current layout
   c. Initialise missing fields with safe defaults
   d. Skip unrecognised fields (advance by size from embedded schema)

5. Apply incremental versioning patches
   → Run patch chain from artifact_version → current_version in order

6. Relocate references
   → Remap any stored addresses/IDs to valid current references

7. Hand off to public interface layer (RNA equivalent)
   → Expose via stable API, abstracting storage details
```

---

## Schema Block Design

The embedded schema block should be compact and self-contained. A minimal schema block contains:

| Component | Purpose |
|-----------|---------|
| Magic bytes / identifier | Allows the loader to locate the schema block within the artifact |
| Schema format version | Allows the schema block format itself to evolve |
| Type table | Maps type names to sizes (e.g. `int32 = 4 bytes`) |
| Struct table | Maps struct names to ordered lists of (field\_name, type, array\_length) |
| String table | Interned field and type name strings, referenced by index |

The schema block is written **once per artifact** at save time and parsed **once per load** before any data blocks are processed.

---

## Versioning Patch Guidelines

When writing a versioning patch function:

- **One patch, one concern.** Each patch function should address exactly one structural change. If two things changed in the same release, write two patch functions.
- **Patches are permanent.** Never delete a patch from the chain. An artifact written at version N must always be upgradeable by running patches N→N+1→...→current.
- **Patches operate on raw loaded data**, not on the public API model. They transform the internal representation before it is handed to the interface layer.
- **Document the reason**, not just the transformation. Future maintainers need to understand *why* the data changed, not only *how*.
- **Test with the oldest artifact you have.** The full patch chain must be exercised regularly or patches will silently rot.

---

## When to Use This Pattern

SDNA-style self-description is worth the implementation cost when:

- Your data format will be written and read by **different versions** of your software.
- Your data format will be read by **third-party tools** you do not control.
- Your **internal data structures are expected to evolve** over the lifetime of the project.
- You need **long-term archival integrity** (files must remain readable years or decades later).
- Your format will cross **platform, endianness, or pointer-size boundaries**.

It may be overkill when:
- Data is ephemeral (never persisted beyond a single session).
- The format is entirely internal and the codebase is small enough that migrations are trivial.
- You are using an existing format (Protobuf, Avro, Cap'n Proto) that already provides these guarantees.

---

## Relationship to Existing Technologies

SDNA's principles are not unique to Blender — several established technologies implement subsets of them:

| Technology | Self-describing | Name-based mapping | Incremental versioning | Graceful degradation |
|------------|:-:|:-:|:-:|:-:|
| **SDNA (Blender)** | ✓ | ✓ | ✓ | ✓ |
| Protocol Buffers | ✓ | ✓ (field numbers) | Partial | ✓ |
| Apache Avro | ✓ | ✓ | Partial | ✓ |
| Apache Thrift | ✓ | ✓ (field IDs) | Partial | ✓ |
| Cap'n Proto | ✓ | ✓ (field IDs) | Partial | ✓ |
| JSON (plain) | Partial | ✓ | ✗ | Partial |
| Raw binary structs | ✗ | ✗ | ✗ | ✗ |

SDNA's distinguishing feature is the **incremental versioning patch chain** applied at load time, which allows arbitrarily deep structural migrations that schema-mapping alone cannot handle. Most binary schema systems leave this as a problem for the application layer; SDNA treats it as a first-class concern of the format itself.

---

## Summary of Principles

| Principle | One-line statement |
|-----------|-------------------|
| **Self-Description** | The artifact carries its own schema; the reader discovers layout, it does not assume it |
| **Name-Based Mapping** | Fields are matched by name across versions, never by position or offset |
| **Incremental Versioning** | Migrations are a chain of small, ordered, permanent patch functions |
| **Graceful Degradation** | Unknown blocks and fields are skipped cleanly; partial reads are valid |
| **Two-Layer Architecture** | Storage schema and public interface schema are separated by a stable mapping layer |

---

> The file carries its own DNA. The reader reconciles, patches, and translates. New implementations do not break old data. Old readers do not crash on new data. The system evolves without a flag day.
