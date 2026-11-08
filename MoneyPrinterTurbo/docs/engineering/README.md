# MoneyPrinterTurbo — Engineering Knowledge Base

This directory contains the durable engineering reference documents for the
MoneyPrinterTurbo v1.3.8 local fork. These documents are the primary source of
truth for development, audit, testing, and repair work.

## Purpose

This knowledge base exists so that future development sessions can resume
without depending on previous conversation memory. Every document is
evidence-based and references specific source files, functions, and commits.

## Documents

| Document | Purpose | Authoritative For |
|----------|---------|-------------------|
| [system-map.md](system-map.md) | Architecture, production pipeline, modules, interfaces, and data flow | How the system works; module roles; call paths; contracts |
| [audit-register.md](audit-register.md) | Consolidated issue register from all audits | Issue status, severity, evidence, and remaining work |
| [repair-roadmap.md](repair-roadmap.md) | Ordered, dependency-aware repair and verification plan | What to fix next, in what order, and how to verify |
| [decisions-log.md](decisions-log.md) | Engineering decisions with rationale and conditions for revisiting | Why the system is designed as it is; settled questions |

## Current Baseline

### Original Source Snapshot (2026-10-15)

This snapshot was recorded during the first audit and remains unchanged:

- **Version:** MoneyPrinterTurbo v1.3.8 (local fork)
- **HEAD commit:** `b819f7e` — `fix(video): render background colors (#1596)`
- **Branch:** Detached HEAD at tag `v1.3.8`
- **Working tree:** 15 modified source/test files, 1 untracked config backup
  (`config.toml.backup-20261010-223608`)
- **Uncommitted feature:** `openai_image` scene-generation pipeline
  (+1791 / −40 lines across those 15 files)

### Current Repository State

The repository currently contains:

- **15 modified source/test files** — the same `openai_image` scene-generation
  pipeline changes described in the original snapshot. No source modifications
  have been added or removed since the snapshot.
- **1 untracked configuration backup** — `config.toml.backup-20261010-223608`,
  present since before the knowledge-base task.
- **1 new untracked directory** — `docs/engineering/` containing five Markdown
  files: `README.md`, `system-map.md`, `audit-register.md`, `repair-roadmap.md`,
  and `decisions-log.md`. This is the only addition made by the knowledge-base
  task.

## Reading Order Before Making Changes

1. Read this file for orientation.
2. Read `system-map.md` to understand the production pipeline and module roles.
3. Read `audit-register.md` to understand known issues and their current status.
4. Read `repair-roadmap.md` to understand what work is planned and what is blocked.
5. Read `decisions-log.md` to understand why the system is designed as it is.

## Key Rules

1. **Unresolved findings are not confirmed defects.** Do not treat an
   `UNVERIFIED` or `NO DEFECT DEMONSTRATED` status as permission to change
   code. Gather evidence first.

2. **Implementation changes must be documented.** When a change alters the
   production pipeline, data contracts, or module interfaces, update the
   system map and record the decision in the decisions log.

3. **Static inspection is not runtime verification.** A finding confirmed by
   reading code is weaker than one confirmed by running a test. Label
   evidence correctly.

4. **The source code is authoritative.** When documentation conflicts with the
   current source, the source wins. Update the documentation.

5. **Never expose secrets.** Configuration values such as API keys, tokens,
   and passwords must not appear in these documents.

## Test Environment Status

As of the most recent verification (2026-10-16):

- Python 3.14.8 is available on the system.
- No virtual environment exists (`.venv` is absent).
- No project dependencies are installed (pytest, Streamlit, MoviePy, Redis,
  edge_tts, loguru, requests are all missing).
- All test execution is **BLOCKED** pending dependency installation.
- Previous audit conclusions were based on static code inspection only.

### Environment Setup (Approval-Required)

The following instructions are a reference for when setup is approved. **Do
not execute them without explicit user authorization.**

Before installing dependencies:

1. Inspect `requirements.txt`, `pyproject.toml`, and `.python-version` for
   the project's supported Python version range.
2. Verify that the available Python interpreter (currently 3.14.8) is
   compatible with the project's dependencies. Do not assume compatibility
   simply because Python is installed.
3. If the available Python version is not supported, identify a compatible
   version before proceeding.

Setup reference (when approved):

```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pip install pytest
python -m pytest test/services/ -x -v
```

## How to Update This Knowledge Base

After any significant audit, fix, or test run:

1. Update `audit-register.md` with new evidence and status changes.
2. Update `repair-roadmap.md` if priorities or acceptance criteria change.
3. Update `system-map.md` if implementation contracts or data flow change.
4. Record consequential decisions in `decisions-log.md`.
5. Record the test command, environment, observed result, and limitations.
6. Preserve previous decisions as superseded rather than deleting history.
7. Update the baseline information in this file if the HEAD commit changes.

Every future task should read this index and the relevant canonical document
before proposing changes.

## Relationship to Existing Documentation

The `docs/` directory also contains:

- `docs/video-projects.md` — Video project documentation
- `docs/skill/SKILL.md` — MPT agent skill documentation
- `docs/voice-list.txt` — Voice provider catalog
- `docs/webui.jpg`, `docs/webui-en.jpg` — UI screenshots
- `docs/api.jpg` — API documentation

These are user-facing documents. The `docs/engineering/` directory contains
developer-facing engineering reference material.