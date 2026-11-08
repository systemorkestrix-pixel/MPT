# MoneyPrinterTurbo — Repair Roadmap and Verification Matrix

> **Baseline:** commit `b819f7e`, tag `v1.3.8`, snapshot 2026-10-15
> **Prepared:** 2026-10-16
> **Status:** No confirmed defects from static inspection. All work is
> verification or optional hardening.

## Roadmap Overview

```
Phase 0: Environment Setup (approval-gated) ───────────────────────────
    │
Phase 1: Runtime Verification of Historical Fixes (PA-1 through PA-10)
    │         [blocked by Phase 0]
Phase 2: Runtime Verification of Scene Pipeline (PA-11, PA-12, PA-15)
    │         [blocked by Phase 0]
Phase 3: Optional Hardening (PA-11 prompt, PA-12 assertion, PA-13 test)
    │         [H1, H3 blocked by Phase 2; H2 independent]
Phase 4: Closure
```

Phases 1 and 2 are runtime-verification phases blocked by the unavailable
test environment (Phase 0). Phase 3 contains optional hardening tasks with
different prerequisites: H1 and H3 require Phase 2 to be complete, while H2
(adding a comment and assertion linking FPS constants) can be authored
independently of environment setup, though verifying the assertion requires
a working test environment. No optional hardening has been approved or started.

---

## Phase 0 — Environment Setup

**Purpose:** Enable test execution.

**Approval required:** Yes — dependency installation is approval-gated.
Do not execute without explicit user authorization.

**Pre-installation checks:**

1. Inspect `requirements.txt`, `pyproject.toml`, and `.python-version` for
   the project's supported Python version range.
2. Verify that the available Python interpreter (currently 3.14.8) is
   compatible with the project's dependencies. Do not assume compatibility.
3. If the available Python version is not supported, identify a compatible
   version before proceeding.

**Action (when approved):**
```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pip install pytest
```

**Verification:** `python -m pytest test/services/test_config.py -x` passes.

**Blocker:** This phase blocks Phases 1 and 2. Phase 3 task H2 can be
authored without this phase, but verifying its assertion requires a working
test environment.

---

## Phase 1 — Runtime Verification of Historical Fixes

**Purpose:** Confirm that the 10 historically fixed issues remain fixed at
runtime. No code changes expected.

### Batch 1A — Voice (PA-1 through PA-6)

**Files:** `voice.py` and 7 test files

**Command:**
```
python -m pytest test/services/test_voice.py test/services/test_elevenlabs_catalog_pagination.py test/services/test_azure_file_publication.py test/services/test_voxcpm_stream_bounds.py test/services/test_elevenlabs_speech_speed.py test/services/test_compatible_speech_transport.py test/services/test_remote_speech_complete_decode.py -v
```

**Expected:** All tests pass. `test_azure_file_publication.py` may skip if
FFmpeg is not installed.

**Acceptance criteria:** Zero failures. Skips must be recorded individually
with the test name and skip reason. A batch-level pass does not override a
missing acceptance check: if `test_azure_file_publication.py` skips, PA-5
is not runtime-verified by that batch.

**Issues closed if passing:** PA-1, PA-2, PA-3, PA-4, PA-6. PA-5 may only
be closed if `test_azure_file_publication.py` actually ran and passed; if it
skipped, PA-5 remains open pending a future run with FFmpeg available.

### Batch 1B — Video Rendering (PA-7, PA-9)

**Files:** `video.py` and 5 test files

**Command:**
```
python -m pytest test/services/test_video.py test/services/test_zoom_alpha_alignment.py test/services/test_short_clip_transitions.py test/services/test_subtitle_color_preflight.py test/services/test_bgm_final_fade.py -v
```

**Expected:** All tests pass.

**Acceptance criteria:** Zero failures.

**Issues closed if passing:** PA-7, PA-9

### Batch 1C — State and Batch Preservation (PA-8, PA-10)

**Files:** `state.py`, `task.py`, and 5 test files

**Command:**
```
python -m pytest test/services/test_state.py test/services/test_state_string_values.py test/services/test_redis_patch_ownership.py test/services/test_partial_batch_results.py test/services/test_task_artifacts.py -v
```

**Expected:** Most tests pass. `test_real_redis_patch_and_delete_are_atomic`
skips unless `MPT_TEST_REDIS_HOST` is set.

**Acceptance criteria:** Zero failures. `test_real_redis_patch_and_delete_are_atomic`
is gated behind `MPT_TEST_REDIS_HOST`. If the env var is not set, this test
will be skipped — record it as `SKIPPED`. A passing batch with this test
skipped does not constitute runtime verification of Redis atomicity; only the
mock-based logic tests are verified in that case.

**Issues closed if passing:** PA-8, PA-10 (PA-8 Redis atomicity remains
unverified if the real-Redis test is skipped)

---

## Phase 2 — Runtime Verification of Scene Pipeline

**Purpose:** Confirm the uncommitted `openai_image` scene pipeline works
correctly at runtime.

### Batch 2A — Scene Generation and Timing (PA-11, PA-12)

**Files:** `llm.py`, `task.py`, and 2 test files

**Command:**
```
python -m pytest test/services/test_llm.py -k "image_scene" test/services/test_task.py -k "image_scene or scene_duration or allocate_scene" -v
```

**Expected:** All tests pass. No real LLM calls are made (tests mock
`_generate_response`).

**Acceptance criteria:** Zero failures. Scene count mismatch retry behavior
verified. Duration allocation verified.

**Issues closed if passing:** PA-11, PA-12

### Batch 2B — Scene Materials (PA-15 partial)

**Files:** `material.py` and 1 test file

**Command:**
```
python -m pytest test/services/test_material_openai_image.py -k "scene" -v
```

**Expected:** All tests pass.

**Acceptance criteria:** Scene clip generation, count mismatch rejection,
invalid duration rejection, and missing-scene stop behavior all verified.

### Batch 2C — Scene Video Composition (PA-13, PA-15 partial)

**Files:** `video.py` and 1 test file

**Command:**
```
python -m pytest test/services/test_video.py -k "image_scene or zoom_cap" -v
```

**Expected:** All tests pass.

**Acceptance criteria:** Scene combine preserves duration, order, and count.
Zoom cap bounds long scenes. Non-image sources still cap and loop.

### Batch 2D — Pipeline Integration (PA-15)

**Files:** `task.py` and 1 test file

**Command:**
```
python -m pytest test/services/test_task.py -k "openai_image_pipeline" -v
```

**Expected:** All tests pass.

**Acceptance criteria:** Full pipeline from script through scene descriptions,
timing, materials, composition, and task state verified with mocked stages.

### Batch 2E — WebUI (PA-15 partial)

**Files:** `webui/Main.py` and 2 test files

**Command:**
```
python -m pytest test/services/test_webui_generation_defaults.py test/services/test_webui_i18n.py -v
```

**Expected:** All tests pass. Requires Streamlit installed.

**Acceptance criteria:** Scene description editor, submission, persistence,
restoration, and language switching all verified.

**Blocker:** Requires Streamlit. If not installed, mark as BLOCKED.

---

## Phase 3 — Optional Hardening

**Purpose:** Improve resilience beyond current correctness. No defects
require these changes.

### H1 — Retry Prompt Count Emphasis (PA-11)

**Issue:** PA-11
**Status:** `NO DEFECT DEMONSTRATED`
**Rationale:** The retry loop uses the same prompt each time. Adding count
emphasis on retry could reduce LLM mismatch frequency for high paragraph
counts (6+).

**Scope:** `llm.py`: `generate_image_scene_descriptions` (L1058–1110)

**Change:** On retry (when `i > 0`), append to prompt:
```
IMPORTANT: You MUST return exactly {amount} items. Do not return more or fewer.
```

**Tests to add:** Verify prompt content on second retry attempt.

**Acceptance criteria:** Retry prompt includes count emphasis. Existing tests
still pass.

**Prerequisites:** Phase 2 complete.

### H2 — FPS Constant Linkage (PA-12)

**Issue:** PA-12
**Status:** `NO DEFECT DEMONSTRATED`
**Rationale:** `fps = 30` (video.py L78) and `SCENE_DURATION_FPS = 30`
(task.py L484) are separate constants. If one changes, the tolerance
calculation in `combine_videos` would be incorrect.

**Scope:** `task.py`: line 484

**Change:** Add comment:
```python
# Must match video.fps (currently 30). The combine_videos tolerance
# calculation uses the module-level fps constant.
SCENE_DURATION_FPS = 30
```

**Tests to add:** `assert SCENE_DURATION_FPS == video.fps`

**Acceptance criteria:** Comment present. Assertion passes.

**Prerequisites:** The comment and assertion can be authored independently
of environment setup. However, verifying that the assertion passes requires
a test environment capable of running the relevant test. No runtime result
can be claimed before the test actually runs.

### H3 — Transition + Scene Preservation Test (PA-13)

**Issue:** PA-13
**Status:** `NO DEFECT DEMONSTRATED`
**Rationale:** Static inspection confirms all transitions preserve clip
duration, but no test exercises transitions with `preserve_scene_clips=True`.

**Scope:** `test_video.py`

**Change:** Add test that calls `combine_videos` with `image_scene_durations`
set and `video_transition_mode=fade`. Verify scene count and durations are
preserved.

**Tests to add:** 1 new test in `test_video.py`.

**Acceptance criteria:** Test passes. Scene clips maintain assigned duration
after fade transition.

**Prerequisites:** Phase 2 complete.

**Residual gap if not run:** The transition-plus-scene-preservation interaction
remains verified only by static inspection. Static inspection confirms all 6
transitions preserve clip duration, which is sufficient for the `NO DEFECT
DEMONSTRATED` finding, but is weaker than a runtime test result.

---

## Phase 4 — Closure

### Closure Criteria

| Issue | Can Close When |
|-------|---------------|
| PA-1 through PA-4, PA-6 | Batch 1A passes with zero failures |
| PA-5 | Batch 1A passes with zero failures **and** `test_azure_file_publication.py` ran (did not skip). If it skipped due to missing FFmpeg, PA-5 remains open. |
| PA-7, PA-9 | Batch 1B passes with zero failures |
| PA-10 | Batch 1C passes with zero failures |
| PA-8 | Batch 1C passes with zero failures **and** `test_real_redis_patch_and_delete_are_atomic` passes (not skipped). If that test is skipped because `MPT_TEST_REDIS_HOST` is unavailable, record `SKIPPED`, retain the runtime-verification gap, and leave this criterion open. Mock-based logic tests alone do not verify Redis atomicity. |
| PA-11, PA-12 | Batch 2A passes with zero failures |
| PA-13 | Batch 2C passes (verifies ordinary scene composition). The transition-plus-scene-preservation interaction is verified by static inspection (`NO DEFECT DEMONSTRATED`). Runtime verification of that specific interaction requires optional test H3. If H3 is not run, the residual verification gap must be documented: static inspection confirms transitions preserve clip duration, but no runtime test exercises that path. |
| PA-14 | No action needed (NOT A DEFECT) |
| PA-15 | Batches 2A–2E all pass |

### Closure Checklist

- [ ] All Phase 1 batches pass
- [ ] All Phase 2 batches pass
- [ ] PA-8 Redis atomicity: `test_real_redis_patch_and_delete_are_atomic` passes with `MPT_TEST_REDIS_HOST` set, or recorded as `SKIPPED` with the runtime-verification gap explicitly retained and the criterion left open
- [ ] All test results recorded with command, environment, and date
- [ ] Audit register updated with runtime verification status
- [ ] System map updated if any contracts changed
- [ ] No `CONFIRMED OPEN` defects remain

### Stop Conditions

Stop and request review if:

1. Any Phase 1 batch produces failures — a historical fix has regressed.
2. Any Phase 2 batch produces failures — the scene pipeline has a defect.
3. The Redis atomicity test fails — concurrency protection is broken.
4. A test reveals a previously unknown interaction between stages.

---

## Verification Matrix

| Phase | Batch | Issues | Test Files | Requires | Status |
|-------|-------|--------|------------|----------|--------|
| 0 | Setup | — | — | Approval, compatible Python | BLOCKED |
| 1 | 1A Voice | PA-1–6 | 7 | Phase 0 | BLOCKED |
| 1 | 1B Video | PA-7,9 | 5 | Phase 0 | BLOCKED |
| 1 | 1C State | PA-8,10 | 5 | Phase 0 | BLOCKED |
| 2 | 2A Scene Gen | PA-11,12 | 2 | Phase 0 | BLOCKED |
| 2 | 2B Scene Mat | PA-15 | 1 | Phase 0 | BLOCKED |
| 2 | 2C Scene Vid | PA-13,15 | 1 | Phase 0 | BLOCKED |
| 2 | 2D Pipeline | PA-15 | 1 | Phase 0 | BLOCKED |
| 2 | 2E WebUI | PA-15 | 2 | Phase 0 + Streamlit | BLOCKED |
| 3 | H1 Retry | PA-11 | 1 | Phase 2 | NOT STARTED |
| 3 | H2 FPS | PA-12 | 1 | Authoring: none; verification: Phase 0 | NOT STARTED |
| 3 | H3 Trans | PA-13 | 1 | Phase 2 | NOT STARTED |

Phases 1 and 2 are BLOCKED pending environment setup (Phase 0). Phase 3
tasks are NOT STARTED; H1 and H3 are blocked by Phase 2, while H2 can be
authored independently but its verification still requires Phase 0. No
optional hardening has been approved.