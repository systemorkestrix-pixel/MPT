# MoneyPrinterTurbo — Audit Register

> **Baseline snapshot:** 2026-10-15, commit `b819f7e`, tag `v1.3.8`
> **Documents prepared:** 2026-10-16
> **Verification status:** All findings verified by static code inspection.
> Runtime test execution was BLOCKED (no dependencies installed).

## Status Legend

| Status | Meaning |
|--------|---------|
| `CONFIRMED OPEN` | Defect verified present by evidence; needs repair |
| `HISTORICALLY FIXED` | Defect was present, fix is present in current code |
| `REGRESSION RISK` | Fix present but interaction or coverage gap remains |
| `UNVERIFIED` | Cannot confirm or deny; evidence insufficient |
| `NO DEFECT DEMONSTRATED` | Inspected; no defect found by static analysis |
| `NOT A DEFECT` | Expected behavior, model variability, or editorial preference |

---

## PA-1: Voice/Speech Redirect and Replay Prevention

| Field | Value |
|-------|-------|
| **ID** | PA-1 |
| **Description** | HTTP speech APIs may redirect or replay billed synthesis |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | Critical |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `elevenlabs_tts` (L2509), `siliconflow_tts` (L1656), `minimax_tts` (L2381), `_openai_compatible_tts` (L2653), `fish_audio_tts` (L2906), `voxcpm_tts` (L3250), `get_elevenlabs_voices` (L221) |
| **Evidence** | All 7 HTTP providers use `allow_redirects=False` and check `300 <= status_code < 400` to stop retries |
| **Regression tests** | `test_voice.py`: `test_elevenlabs_tts_does_not_follow_or_retry_redirect` (L1586), `test_get_elevenlabs_voices_does_not_forward_key_on_redirect` (L1498). Multiple tests assert `allow_redirects=False`. |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-2: Voice Response Validation

| Field | Value |
|-------|-------|
| **ID** | PA-2 |
| **Description** | Invalid or incomplete audio from remote TTS could replace successful narration |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | High |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `_validate_remote_tts_audio` (L2292), `_write_validated_minimax_audio` (L2305) |
| **Evidence** | `_validate_remote_tts_audio` calls `bgm.validate_audio_file` for full FFmpeg decode. Used by SiliconFlow (L1685), MiniMax (L2324), ElevenLabs (L2567), OpenAI-compatible (L2678), Fish Audio (L2953). Each writes to temp file, validates, then `os.replace`. |
| **Regression tests** | All voice tests mock `_validate_remote_tts_audio`. `test_azure_file_publication.py` does real FFmpeg validation. |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-3: Voice Speed Propagation

| Field | Value |
|-------|-------|
| **ID** | PA-3 |
| **Description** | Voice rate not applied to synthesis, producing default-speed narration |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | Medium |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `elevenlabs_tts` (L2470–2490), `siliconflow_tts` (L1638), `minimax_tts` (L2358), `_openai_compatible_tts` (L2641), `fish_audio_tts` (L2863), Azure SSML (L1780) |
| **Evidence** | Each provider propagates `voice_rate` to API. ElevenLabs validates model-specific ranges. |
| **Regression tests** | `test_elevenlabs_speech_speed.py`, `test_voice.py` speed payload checks |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-4: ElevenLabs Voice Catalog Pagination

| Field | Value |
|-------|-------|
| **ID** | PA-4 |
| **Description** | Only first page of ElevenLabs voice catalog loaded |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | Medium |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `get_elevenlabs_voices` (L198–260) |
| **Evidence** | Pagination loop with `next_page_token`, `has_more` check, `seen_voices` dedup, `seen_tokens` cycle detection, 30s deadline, 100-page cap |
| **Regression tests** | `test_elevenlabs_catalog_pagination.py`: 5 tests |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-5: Azure Speech Output Staging

| Field | Value |
|-------|-------|
| **ID** | PA-5 |
| **Description** | Azure SDK file handle prevents replacement on Windows |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | High |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `azure_tts_v2` (L1826–1894) |
| **Evidence** | Uses `tempfile.mkstemp` per attempt (L1829), releases SDK owners before `os.replace` (L1865–1868), cleans up in `finally` (L1884–1894) |
| **Regression tests** | `test_azure_file_publication.py`: 6 parametrized outcomes |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | `test_azure_file_publication.py` requires FFmpeg. If FFmpeg is unavailable at test time, this test will skip and PA-5 will not be runtime-verified by batch 1A. Record the skip and leave PA-5 open until a run with FFmpeg available confirms the behavior. |

---

## PA-6: VoxCPM Streaming Bounds

| Field | Value |
|-------|-------|
| **ID** | PA-6 |
| **Description** | Unbounded streaming data from VoxCPM SSE could exhaust memory |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | Medium |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `voice.py`: `voxcpm_tts` (L3270–3285), `_iter_voxcpm_sse_events` |
| **Evidence** | `_VOXCPM_TTS_MAX_AUDIO_BYTES = 50MB` (L95), `_VOXCPM_SSE_MAX_EVENT_BYTES` bounds transport (L96), per-chunk byte check (L3284) |
| **Regression tests** | `test_voxcpm_stream_bounds.py`: 8+ tests |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-7: Video Rendering Edge Cases

| Field | Value |
|-------|-------|
| **ID** | PA-7 |
| **Description** | Individual rendering defects (zoom, alpha masks, subtitle colors, background colors, concat paths, music fade) |
| **Status** | `HISTORICALLY FIXED` (individual fixes) |
| **Severity** | High |
| **Root-cause confidence** | Confirmed for individual fixes |
| **Affected modules** | `video.py`: `combine_videos`, `render_image_zoom_video`, zoom constants, alpha mask handling, subtitle color validation |
| **Evidence** | 8 fix commits present in current code (Oct 5–9): zoom alpha alignment, subtitle color preflight, background music fade, image intermediate isolation, narration volume, background colors, concat paths, short clip transitions |
| **Regression tests** | `test_video.py`: zoom cap, combine scenes, concat paths. `test_zoom_alpha_alignment.py`, `test_short_clip_transitions.py`, `test_subtitle_color_preflight.py`, `test_bgm_final_fade.py` |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | See PA-13 for scene-preservation interaction |

---

## PA-8: Concurrency and State Isolation

| Field | Value |
|-------|-------|
| **ID** | PA-8 |
| **Description** | Race conditions in Redis state, material downloads, artifact writes, LoomLoom polling |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | High |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `state.py`: `_PATCH_EXISTING_TASK_SCRIPT` Lua script (L13–28), `patch_task` (L241). `material.py`, `loomloom.py`, `task_artifacts.py` |
| **Evidence** | Redis `patch_task` uses Lua script with ownership check. `MemoryState` uses `threading.RLock`. |
| **Regression tests** | `test_state.py`: `test_patch_task_updates_only_existing_redis_task` (L385), `test_real_redis_patch_and_delete_are_atomic` (L405, requires `MPT_TEST_REDIS_HOST`). `test_redis_patch_ownership.py`. |
| **Runtime verified** | No (tests BLOCKED; real-Redis test requires env var) |
| **Remaining work** | Run batch 1C. If `test_real_redis_patch_and_delete_are_atomic` is skipped due to missing `MPT_TEST_REDIS_HOST`, record it as `SKIPPED` — a passing batch with this test skipped does not constitute runtime verification of Redis atomicity. The mock-based tests verify logic only. |

---

## PA-9: Video Concatenation Path Preservation

| Field | Value |
|-------|-------|
| **ID** | PA-9 |
| **Description** | Path normalization could break FFmpeg concat file references |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | Medium |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `video.py`: `_format_ffmpeg_concat_path` (L496) |
| **Evidence** | Commit `9887b45` fixed concat path handling |
| **Regression tests** | `test_video.py`: `test_format_ffmpeg_concat_path_normalizes_windows_path` (L899) |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-10: Batch Task Preservation on Failure

| Field | Value |
|-------|-------|
| **ID** | PA-10 |
| **Description** | Batch failure could discard completed videos from earlier iterations |
| **Status** | `HISTORICALLY FIXED` |
| **Severity** | High |
| **Root-cause confidence** | Confirmed |
| **Affected modules** | `task.py`: `_mark_task_failed` (L290–298), `generate_final_videos` (L1170–1176) |
| **Evidence** | `_mark_task_failed` preserves `videos`, `combined_videos`, `warnings`, `failed_video_index`, `video_scene_descriptions`. `generate_final_videos` updates `partial_results` before each attempt. |
| **Regression tests** | `test_partial_batch_results.py`: 6 tests |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | None by static inspection |

---

## PA-11: Image Scene Description Count Mismatch

| Field | Value |
|-------|-------|
| **ID** | PA-11 |
| **Description** | LLM may return wrong number of scene descriptions for script paragraphs |
| **Status** | `NO DEFECT DEMONSTRATED` |
| **Severity** | Medium |
| **Root-cause confidence** | Probable (expected LLM variability) |
| **Affected modules** | `llm.py`: `generate_image_scene_descriptions` (L992–1118), `task.py`: `resolve_image_scene_descriptions` (L424) |
| **Evidence** | Retry loop bounded at `_max_retries = 5` (L20). Mismatched counts discarded, never truncated (L1103). Controlled failure on exhaustion (L1112–1116). Same prompt on each retry (no count emphasis). |
| **Regression tests** | `test_llm.py`: `test_image_scene_count_mismatch_is_retried_and_never_truncated`. `test_task.py`: `test_supplied_scene_count_mismatch_fails_before_scene_or_material_generation`. |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | Optional: add count emphasis on retry prompt. Not required for correctness. |

---

## PA-12: Scene Duration Allocation and FPS Constants

| Field | Value |
|-------|-------|
| **ID** | PA-12 |
| **Description** | Scene timing uses separate FPS constants that could diverge |
| **Status** | `NO DEFECT DEMONSTRATED` |
| **Severity** | Medium |
| **Root-cause confidence** | Confirmed (constants verified equal) |
| **Affected modules** | `task.py`: `SCENE_DURATION_FPS = 30` (L484), `video.py`: `fps = 30` (L78) |
| **Evidence** | Both constants are 30 as of snapshot. Tolerance in `combine_videos` (L1279) uses `fps`. Allocation uses `SCENE_DURATION_FPS`. |
| **Regression tests** | `test_task.py`: `test_allocate_scene_durations_*` (4 tests), `test_combine_image_scenes_fail_when_duration_cannot_cover_audio` |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | Optional: add comment or assertion linking the two constants |

---

## PA-13: Scene Preservation and Transition Interaction

| Field | Value |
|-------|-------|
| **ID** | PA-13 |
| **Description** | Transitions applied to scene-preservation clips could modify duration or alpha |
| **Status** | `NO DEFECT DEMONSTRATED` |
| **Severity** | Low |
| **Root-cause confidence** | Confirmed (transitions verified duration-preserving) |
| **Affected modules** | `video.py`: `combine_videos` (L1150–1174), `video_effects.py`: all 6 transition functions |
| **Evidence** | All transitions preserve clip duration: `FadeIn`/`FadeOut` are alpha effects; `SlideIn`/`SlideOut` use explicit `.with_duration(clip.duration)`; `ZoomIn`/`ZoomOut` use `clip.transform()`. Speed scaling (L1126) and duration cap (L1176) are correctly skipped for scene clips. |
| **Regression tests** | `test_video.py`: `test_combine_image_scenes_*` (5 tests). No test exercises transitions with `preserve_scene_clips=True`. |
| **Runtime verified** | No (tests BLOCKED; static inspection confirms duration preservation) |
| **Remaining work** | Static inspection confirms transitions preserve clip duration. Runtime verification of ordinary scene composition is covered by batch 2C. Runtime verification of the specific combination of transitions with `preserve_scene_clips=True` requires optional hardening test H3. Without H3, that specific interaction remains unverified at runtime — the static finding `NO DEFECT DEMONSTRATED` still holds but is weaker than a runtime result. |

---

## PA-14: Prompt Configuration Iteration

| Field | Value |
|-------|-------|
| **ID** | PA-14 |
| **Description** | Config prompts iterated heavily; LLM output quality varies |
| **Status** | `NOT A DEFECT` |
| **Severity** | Low |
| **Root-cause confidence** | N/A |
| **Affected modules** | `config.toml`: `video_script_prompt`, `custom_system_prompt` |
| **Evidence** | Backup (Oct 10) has empty prompts; current has detailed prompts. Iterative content-quality tuning, not a software defect. |
| **Regression tests** | N/A |
| **Runtime verified** | N/A |
| **Remaining work** | None. LLM variability is expected. |

---

## PA-15: End-to-End `openai_image` Pipeline Contract

| Field | Value |
|-------|-------|
| **ID** | PA-15 |
| **Description** | Scene pipeline spans 6 modules; contracts between stages must be consistent |
| **Status** | `NO DEFECT DEMONSTRATED` |
| **Severity** | Medium |
| **Root-cause confidence** | Probable |
| **Affected modules** | `llm.py` → `schema.py` → `task.py` → `material.py` → `video.py` → `webui/Main.py` |
| **Evidence** | `test_task.py`: `test_openai_image_pipeline_allocates_durations_before_materials` (L1138) covers full pipeline with mocks. `test_openai_image_pipeline_fails_before_materials_on_scene_mismatch` (L1192) verifies fail-fast. `save_script_data` condition (L601–605) traced correct for all cases. |
| **Regression tests** | `test_task.py` (3 pipeline tests), `test_material_openai_image.py` (4 scene-clip tests), `test_llm.py` (6 scene-generation tests), `test_video.py` (5 scene-combine tests), `test_webui_generation_defaults.py` (1 comprehensive UI test) |
| **Runtime verified** | No (tests BLOCKED) |
| **Remaining work** | Run full test suite when dependencies available |

---

## Summary Table

| ID | Description | Status | Severity | Runtime Verified |
|----|-------------|--------|----------|-----------------|
| PA-1 | Voice redirect/replay prevention | `HISTORICALLY FIXED` | Critical | No |
| PA-2 | Voice response validation | `HISTORICALLY FIXED` | High | No |
| PA-3 | Voice speed propagation | `HISTORICALLY FIXED` | Medium | No |
| PA-4 | ElevenLabs catalog pagination | `HISTORICALLY FIXED` | Medium | No |
| PA-5 | Azure output staging | `HISTORICALLY FIXED` | High | No |
| PA-6 | VoxCPM streaming bounds | `HISTORICALLY FIXED` | Medium | No |
| PA-7 | Video rendering edge cases | `HISTORICALLY FIXED` | High | No |
| PA-8 | Concurrency/state isolation | `HISTORICALLY FIXED` | High | No |
| PA-9 | Concat path preservation | `HISTORICALLY FIXED` | Medium | No |
| PA-10 | Batch task preservation | `HISTORICALLY FIXED` | High | No |
| PA-11 | Scene count mismatch | `NO DEFECT DEMONSTRATED` | Medium | No |
| PA-12 | FPS constants alignment | `NO DEFECT DEMONSTRATED` | Medium | No |
| PA-13 | Scene + transition interaction | `NO DEFECT DEMONSTRATED` | Low | No |
| PA-14 | Prompt iteration | `NOT A DEFECT` | Low | N/A |
| PA-15 | Pipeline integration | `NO DEFECT DEMONSTRATED` | Medium | No |

**No `CONFIRMED OPEN` defects remain from static inspection. All
`HISTORICALLY FIXED` items require runtime test execution to close.
PA-8 has an additional caveat: the Redis atomicity integration test
(`test_real_redis_patch_and_delete_are_atomic`) requires `MPT_TEST_REDIS_HOST`
and will be skipped in environments without Redis; a passing batch with that
test skipped does not verify Redis atomicity at runtime.**