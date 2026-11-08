# MoneyPrinterTurbo — Engineering Decisions Log

> **Prepared:** 2026-10-16
> **Baseline:** commit `b819f7e`, tag `v1.3.8`

This log records important engineering decisions so future work does not reopen
settled questions without new evidence. Each decision includes a rationale,
the evidence that established it, its current status, and conditions that
would justify revisiting it.

---

## DEC-001: Preserve the Original Production Workflow

**Date:** 2026-10-15 (first audit)
**Status:** Active
**Decision:** The standard MoneyPrinterTurbo production pipeline (script →
terms → audio → subtitle → materials → video) shall be preserved wherever no
defect is demonstrated. Historical fixes to voice, video, state, and material
handling shall not be reverted without evidence of regression.

**Rationale:** 42 commits since v1.3.8 addressed specific production defects.
Each fix was reactive to a real failure. The fixes are present in the current
code and have corresponding regression tests.

**Evidence:** Git history from `fafec0f` to `b819f7e`. Static inspection
confirmed all fixes present in working tree.

**Conditions for revisiting:** A regression test failure, a new production
failure, or evidence that a fix introduced an unintended side effect.

---

## DEC-002: Treat `openai_image` Scene Pipeline as Active Development

**Date:** 2026-10-15 (first audit)
**Status:** Active
**Decision:** The uncommitted `openai_image` scene-generation pipeline
(+1791 lines across 15 files) is an existing feature under development,
not a defect. It shall be treated as pending work that extends the
production pipeline for one specific video source.

**Rationale:** The feature adds per-paragraph image scene generation with
allocated durations, zoom capping, and WebUI editing. It has comprehensive
test coverage (1063 test lines). It does not modify the standard video
source paths.

**Evidence:** Working tree diff inspection. Test files cover scene
generation, timing allocation, material generation, video composition,
and WebUI interaction.

**Conditions for revisiting:** Test failures in Phase 2 of the repair
roadmap, or evidence that the feature breaks existing video source paths.

---

## DEC-003: Preserve Paragraph-to-Scene Alignment

**Date:** 2026-10-15 (first audit)
**Status:** Active
**Decision:** Each script paragraph maps to exactly one scene description
and one scene duration. The system shall not silently produce fewer or more
scenes than paragraphs.

**Rationale:** The scene pipeline enforces this contract at every stage:
`resolve_image_scene_descriptions` validates count,
`allocate_scene_durations` allocates one duration per paragraph,
`_generate_openai_image_scene_clips` generates one image per scene,
`combine_videos` validates processed count matches assigned count.

**Evidence:** `llm.py` L1097 (`if len(scene_descriptions) != amount`),
`task.py` L567 (`if len(paragraphs) != len(scene_descriptions)`),
`material.py` L1756 (`if len(scene_durations) != len(search_terms)`),
`video.py` L1280 (`if len(processed_clips) != len(image_scene_durations)`).

**Conditions for revisiting:** A product decision to allow flexible
scene counts (e.g., fewer scenes for short scripts). This would require
changes at all four validation points.

---

## DEC-004: Never Silently Truncate Mismatched Scene Descriptions

**Date:** 2026-10-15 (first audit)
**Status:** Active
**Decision:** When the LLM returns a different number of scene descriptions
than the paragraph count, the system shall retry (up to 5 times) and then
fail the task. It shall not truncate, pad, or silently adjust the list.

**Rationale:** Silent truncation would cause scene-duration allocation to
mismatch the actual scenes, producing videos with missing or duplicated
content. A controlled failure is safer than silent corruption.

**Evidence:** `llm.py` L1103: `scene_descriptions = []` on mismatch.
`task.py` L465: returns `(None, error)` on count mismatch.
`_mark_task_failed` records the error for the user.

**Conditions for revisiting:** A product decision to accept partial results
(e.g., use the first N scenes if the LLM returns more than needed). This
would be a feature change, not a bug fix.

---

## DEC-005: Separate Prompt Quality from Software Correctness

**Date:** 2026-10-15 (first audit)
**Status:** Active
**Decision:** Prompt engineering (script prompts, scene description prompts,
custom system prompts) is content-quality tuning, not a software defect.
Iterative prompt changes shall not be tracked as bugs.

**Rationale:** LLM output is inherently non-deterministic. The software's
job is to validate, retry, and fail gracefully when the LLM output does not
meet structural requirements. Improving output quality through prompts is a
separate concern from ensuring the software handles bad output correctly.

**Evidence:** `config.toml` prompt changes between backup (Oct 10, empty
prompts) and current (detailed multi-paragraph prompts). The retry loop in
`generate_image_scene_descriptions` handles count mismatches regardless of
prompt content.

**Conditions for revisiting:** If prompt changes cause structural failures
(e.g., the LLM stops returning valid JSON), that is a software defect in
the response parser, not a prompt quality issue.

---

## DEC-006: Do Not Claim Runtime Reliability from Static Inspection

**Date:** 2026-10-16 (Phase 1 verification)
**Status:** Active
**Decision:** A finding confirmed by reading code is weaker than one
confirmed by running a test. Static inspection shall be labeled as such
and not treated as runtime verification.

**Rationale:** The Phase 1 verification could not run any tests because
no project dependencies were installed. All conclusions were based on
static code inspection. While static inspection is valuable, it cannot
detect runtime issues like import errors, dependency conflicts, or
environment-specific behavior.

**Evidence:** Phase 1 verification report: all 5 test batches BLOCKED,
Python 3.14.8 available but no venv, no pytest, no project dependencies.

**Conditions for revisiting:** Once tests are actually executed, update
the audit register with runtime verification results and change statuses
from "static inspection" to "runtime verified."

---

## DEC-007: Require Documented Test and Closure Criterion for Every Repair

**Date:** 2026-10-15 (second audit)
**Status:** Active
**Decision:** Every repair must have: (1) a specific test that demonstrates
the failure, (2) a minimal code change, (3) a test that verifies the fix,
and (4) a documented acceptance criterion. No repair may be closed without
evidence.

**Rationale:** The audit found that some historical fixes lacked regression
tests. While the fixes are present in the code, their continued correctness
depends on manual verification or indirect test coverage. Explicit tests
and closure criteria prevent this gap from recurring.

**Evidence:** PA-1 through PA-6 each have dedicated regression tests.
PA-7 has tests for individual fixes but not for interactions.
PA-13 has no test for transitions with scene preservation.

**Conditions for revisiting:** If a repair is trivially verifiable by
existing tests, the additional test requirement may be waived with
documentation of which existing test provides coverage.

---

## DEC-008: FPS Constants Must Remain Aligned

**Date:** 2026-10-16 (Phase 1 verification)
**Status:** Active
**Decision:** `video.fps` (video.py L78, currently 30) and
`SCENE_DURATION_FPS` (task.py L484, currently 30) must remain equal.
The tolerance calculation in `combine_videos` depends on this alignment.

**Rationale:** `allocate_scene_durations` uses `SCENE_DURATION_FPS` for
frame alignment. `combine_videos` uses `fps` for tolerance:
`tolerance = max(1.0, len(image_scene_durations)) / fps`. If these
diverge, the tolerance would be miscalculated, potentially causing
spurious failures or accepting insufficient coverage.

**Evidence:** Both constants verified as 30 in the current source.
The tolerance formula at video.py L1279.

**Conditions for revisiting:** If the project changes its frame rate,
both constants must be updated together. An assertion test (Phase 3, H2)
would catch divergence automatically.

---

## Superseded Decisions

No decisions have been superseded as of 2026-10-16. Future decision changes
should be recorded here with the original decision ID, the date of change,
the new decision, and the reason for the change.