# SDNA Review Skill

A Claude Code skill that assesses codebases, implementation plans, and data format specifications against the **SDNA methodology** — a set of five design principles distilled from Blender's self-describing `.blend` file format — and returns a scored report with prioritised, actionable alignment instructions.

---

## What is SDNA?

SDNA (Struct DNA) is the file-format design system at the heart of Blender's `.blend` format. Its central insight is deceptively simple:

> **Embed the schema in the data itself, then reconcile on load.**

Rather than assuming the reading software already knows the structure of saved data, every `.blend` file carries a compact binary block (`DNA1`) that describes every struct, field name, type, and array size used at the moment of saving. When a newer version of Blender opens an old file, it reads the embedded schema, maps fields by name (not position), runs a chain of incremental versioning patches, and upgrades the data on the fly. A file saved in Blender 1.0 (pre-2000) can be opened in Blender today.

This skill generalises those principles into a methodology applicable to any project that serialises structured data — game save files, network protocols, asset pipelines, database schemas, configuration formats, or any system where the data you write today will be read by a version of your software that does not yet exist.

---

## The Five Principles

| # | Principle | One-line statement |
|---|-----------|-------------------|
| 1 | **Self-Description** | The artifact carries its own schema; the reader discovers layout, it does not assume it |
| 2 | **Name-Based Mapping** | Fields are matched by name across versions, never by position or byte offset |
| 3 | **Incremental Versioning** | Migrations are a chain of small, ordered, permanent patch functions |
| 4 | **Graceful Degradation** | Unknown blocks and fields are skipped cleanly; partial reads are valid |
| 5 | **Two-Layer Architecture** | Storage schema and public interface schema are separated by a stable mapping layer |

---

## What the Skill Does

When triggered in Claude Code, the skill:

1. Reads the full SDNA methodology reference (bundled inside the `.skill` file)
2. Analyses the submitted code, plan, or specification against each of the five principles
3. Scores each principle on a 0–3 scale
4. Identifies the specific anti-patterns present in the submission — quoted and referenced, not generic
5. Produces a `README_SDNA.md` report containing:
   - A score table (total out of 15)
   - Principle-by-principle assessment with current approach, anti-pattern, and gap
   - Prioritised alignment instructions tailored to the actual language and ecosystem of the submission
   - An applicability note on whether full SDNA alignment is warranted for the project's scale and lifetime

---

## How to Use

### Installation

Download `SDNA-review.skill` from this repository and install it into Claude Code:

```bash
claude skill install SDNA-review.skill
```

### Triggering the Skill

The skill triggers automatically when you describe a problem involving data format versioning or compatibility. Example prompts:

```
SDNA review my save system — it breaks every time I change the data model
```

```
Assess this serialization plan against SDNA principles
```

```
My config files are brittle across versions — check SDNA alignment
```

```
How do I make this file format backward compatible?
```

You can also trigger it explicitly by including a file, codebase, or design document:

```
Here is my data layer. Give me an SDNA compliance assessment and tell me what to fix first.
```

### What to Submit

The skill accepts any of the following as input:

- Source code that reads or writes structured data (any language)
- File format specifications or design documents
- Database schema definitions or migration scripts
- Network protocol definitions
- Save-file systems, asset pipelines, or configuration formats
- Implementation plans that include serialisation or persistence

### Example Output

The skill produces a `README_SDNA.md` file structured as:

```
# SDNA Alignment Report — [Project Name]

## System Summary
## Score Overview          ← scored table across all five principles
## Principle-by-Principle Assessment
## Prioritised Alignment Instructions
## What Is Already Working
## Applicability Note
```

Each alignment instruction is concrete and ecosystem-specific — a Python project receives Python implementation patterns; a Rust project receives Rust patterns. Where an existing library (Protocol Buffers, Apache Avro, serde with versioning) already satisfies a principle, the skill recommends using it rather than building from scratch.

---

## Skill Format

This skill is distributed as a single `.skill` file — a zip archive containing:

```
SDNA-review.skill
├── SKILL.md                              ← review instructions and output template
└── references/
    └── SDNA-generalized-grounding.md    ← full methodology reference, loaded at review time
```

The methodology reference is loaded fresh on every invocation, ensuring scores are always grounded in the documented principles rather than model memory.

---

## Research Grounding

Two grounding documents are included in this repository for researchers and implementers who want to study the methodology in depth.

### `SDNA-grounding.md` — The Reference Implementation

A technical deep-dive into Blender's `.blend` file format and the original SDNA system. Covers:

- How the `makesdna` tool generates the `DNA1` schema block at save time
- The six-step load process: header parsing, SDNA extraction, schema diffing, name-based field mapping, incremental versioning patches, and pointer relocation
- Forward compatibility and graceful degradation in older Blender versions
- The DNA + RNA two-layer architecture and why it enables 25+ years of backward compatibility
- Source code locations in the Blender repository

This document is the empirical foundation — everything the methodology claims is demonstrated in working production code that has been maintained since the late 1990s.

### `SDNA-generalized-grounding.md` — The Generalised Methodology

A language-agnostic, implementation-agnostic restatement of the SDNA principles for application to new projects. Covers:

- The core problem: data written today will be read by software that does not exist yet
- All five principles stated as design rules with implementation patterns and explicit anti-patterns
- The canonical load sequence in pseudocode
- Schema block design specification (minimum required components)
- Versioning patch guidelines (one patch, one concern; patches are permanent; test with the oldest artifact you have)
- When to use SDNA-style self-description vs. when it is overkill
- Comparison table against Protocol Buffers, Apache Avro, Apache Thrift, Cap'n Proto, plain JSON, and raw binary structs

This is the document the skill loads at review time and the starting point for any new implementation.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{SDNA_skill,
  title = {SDNA Skill: Self-Describing Data Architecture Review Skill for Claude Code},
  author = {[Drift Johnson]},
  year = {2025},
  url = {https://github.com/MushroomFleet/SDNA-skill},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
