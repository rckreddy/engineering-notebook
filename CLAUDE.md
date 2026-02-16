# Engineering Notebook

A living knowledge base of software engineering skills, patterns, and lessons learned across projects.

## Purpose

- Capture coding patterns and skills extracted from studying high-quality codebases (Rust stdlib, etc.)
- Document lessons learned from real project work
- Serve as a quick reference while coding on other projects

## Key File: rust/skills.md

**`rust/skills.md`** is the master index of all Rust skills (currently 124). It contains:
- Every skill numbered with a one-liner description
- Links to the detailed source documents for full code examples
- A TODO roadmap of remaining topics to cover

### Using skills.md in other projects

Add this to any Rust project's `CLAUDE.md`:

```markdown
## Coding Standards
Apply Rust patterns from ~/code/engineering-notebook/rust/skills.md when writing code.
Read the skills index first, then consult the detailed source document for relevant patterns.
Key areas: error handling (41-50), API design (51-62), testing (109-124).
```

This way Claude will read and apply the skills when helping you code.

## Structure

- `rust/skills.md` — **Master index** of all skills (start here)
- `rust/stdlib_skills.md` — Core types & traits (Skills 1-10)
- `rust/memory_ownership.md` — Memory & ownership (Skills 11-25)
- `rust/async_patterns.md` — Async patterns (Skills 26-40)
- `rust/error_handling.md` — Error handling (Skills 41-50)
- `rust/api_design.md` — API design (Skills 51-62)
- `rust/unsafe_patterns.md` — Unsafe Rust (Skills 63-76)
- `rust/embedded_patterns.md` — Embedded & no_std (Skills 77-92)
- `rust/concurrency_patterns.md` — Concurrency (Skills 93-108)
- `rust/testing_patterns.md` — Testing (Skills 109-124)
- `patterns/` — Language-agnostic design patterns (future)

## How to help

When the user asks to update this notebook:
- Read the existing content first to avoid duplicates
- Add new skills in the same format as existing ones
- Update `rust/skills.md` index when adding new skills
- Update the Change Log at the bottom of each file
- Keep entries concrete with code examples, not abstract theory

When the user is working on another project and discovers a pattern worth saving:
- They may say "add this to the notebook" — write it to the appropriate file here
- Update the skills.md index to include the new skill
