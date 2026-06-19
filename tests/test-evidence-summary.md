# Test Evidence Summary — DOM Node Recycling PR

## Test counts

| Suite | Count | Status |
|-------|-------|--------|
| Recycling + scrollbar drag (pytest) | 48 | 48/48 pass |
| Existing virtualization regression (pytest) | 29 | 29/29 pass |
| CDP behavioral (live browser) | 8 | 8/8 pass |
| Native scrollbar drag (SendInput + CDP) | 4 | 4/4 pass |
| **Total** | **89** | **89 pass, 0 skip** |

## Maintainer must-fix items (PR #4474 CHANGES_REQUESTED)

### MF-1: data-msg-idx on .assistant-turn corrupts measurement

**Finding:** `querySelector('[data-msg-idx="N"]')` returns the `.assistant-turn` container (height 560px) instead of the `.assistant-segment` child (height 120px), inflating virtual window padding by 440px per turn.

**Fix:** Use `data-recycle-key` on `.assistant-turn`, leaving `data-msg-idx` exclusively on `.assistant-segment`.

**Evidence (test_mf1_multi_segment_turn_raw_heights):**
- 3-segment turn: segments at 120px, 90px, 150px with tool cards at 60px, 80+60px
- Measured heights: seg5=180px (120+60), seg6=230px (90+80+60), seg7=150px
- Total measured: 560px (correct sum of segments + tools)
- Buggy path would return: 560px for the FIRST SEGMENT ALONE (4.67x inflation)

**Evidence (test_mf1_buggy_path_inflates_height):**
- Buggy (container match): 560px
- Fixed (segment match): 120px
- Inflation: 440px per turn, 4.67x factor

### MF-2: un-typed stash lookups allow cross-type recycling

**Finding:** Without type guards, index shift between renders could hand a user row to the assistant branch or vice versa, causing blank-chat via null dereference.

**Fix:** `classList.contains('msg-row')` guard on user branch, `classList.contains('assistant-turn')` guard on assistant branch.

**Evidence (test_mf2_user_row_in_assistant_slot_without_guard):**
- Without guard: accepts user row (3 mismatched fields: className, dataset.role, id)
- With guard: rejects → falls back to fresh build

**Evidence (test_mf2_assistant_turn_in_user_slot_without_guard):**
- Without guard: accepts assistant turn (3 mismatched fields: className, dataset.role, dataset.msgIdx)
- With guard: rejects → falls back to fresh build

**Evidence (test_mf2_stash_collision_rate_in_shifted_indices):**
- 5 nodes stashed, indices shift by +2 (prepend scenario)
- Without guards: cross-type collisions occur at shifted boundary
- With guards: all collisions rejected, fresh nodes built instead

### MF-3: source-text greps pass on broken code

**Finding:** Original tests were `assertIn(pattern, source_text)` which pass regardless of whether the code path works.

**Fix:** Replaced with behavioral tests that extract and execute functions against mock DOM, producing concrete pixel values and node counts.

**Evidence (test_mf3_measurement_test_fails_on_buggy_layout):**
- Buggy layout: returns 560px (test would FAIL the 120px assertion)
- Fixed layout: returns 120px (test PASSES)
- Test sensitivity confirmed: different outputs on buggy vs fixed

**Evidence (test_mf3_type_check_test_fails_on_unguarded_code):**
- Guarded: rejects wrong-type node (null)
- Unguarded: accepts wrong-type node (non-null)
- Test sensitivity confirmed: opposite results with/without guard

## Scrollbar drag handling

**Problem:** Virtual scroll re-renders during native scrollbar drag need to keep `scrollHeight` stable, otherwise the thumb jumps on release.

**Approach:** Detect scrollbar pointerdown via `offsetX >= clientWidth`. During drag, run full renders (with DOM recycling) using `_compensateScrollForMeasurementDelta` instead of the normal path. This keeps the rendered DOM in sync with the scroll position throughout the drag, so `scrollHeight` stays stable and the thumb doesn't jump when the user releases.

**Why not spacer-only updates:** An earlier iteration used spacer-only updates during drag (updating spacer heights without rebuilding DOM). This caused `scrollHeight` to drift by 10,000+ px because the stale rendered content didn't match the new spacer heights. On release, the full render would correct the content, but the `scrollHeight` change shifted the thumb position by ~40% of the viewport.

**Why full renders work:** The scroll container (`#messages`) is never destroyed; only its inner container (`#msgInner`) gets `innerHTML = ''`. The browser maintains the native pointer grab on the scrollbar because the scrollbar belongs to `#messages`, not `#msgInner`. DOM recycling further reduces the churn by reattaching surviving nodes after the wipe.

**Evidence (native scrollbar drag, Windows SendInput + CDP):**
- 12-step drag from near-top to near-bottom of scrollbar track
- Total scroll delta: 13,522px
- All 12 steps monotonically increasing, 0 reversals
- `_scrollbarDragActive` flag: `true` during drag, `false` after release
- 53 content nodes, 0 blanks after release
- `scrollHeight` delta on release: 49px (0.06% of total, vs 10,710px / 14.8% with spacer-only)

**Evidence (CDP behavioral, synthetic PointerEvents):**
- Scrollbar click (offsetX >= clientWidth): flag activates
- Content click (offsetX < clientWidth): flag does NOT activate
- pointerup: flag clears
- pointercancel: flag clears
- DOM children count preserved during drag (65 → 65)

## Research-informed design

Virtual scroll libraries surveyed: react-virtuoso, TanStack Virtual, react-window, react-virtualized.

**Key finding:** None implement DOM node recycling. This pattern is novel to this codebase.

**Adopted from research:**
- TanStack Virtual's spatial threshold (1.5px) vs temporal (150ms) for programmatic scroll detection — we use a 150ms safety valve
- Debounced clear pattern from react-virtuoso's scroll-to-index implementation — our `_deferClearProgrammaticScroll`
- No library tests scrollbar drag interaction; our SendInput-based test is novel

**Test architecture:**
- Tier 1: Structural (flag/stash existence, lifecycle) — 6 tests
- Tier 2: Bug-specific behavioral (measurement corruption, type guards, stuck flag) — 9 tests
- Tier 3: Integration (full stash→wipe→lookup cycles) — 8 tests
- Tier 4: Optimization correctness (content-skip, key coercion) — 3 tests
- Tier 5: Scrollbar drag (detection, full-render-during-drag, release) — 13 tests
- Tier 6: Maintainer regression tests (raw numerical evidence for each must-fix) — 9 tests
- CDP behavioral: live browser flag lifecycle and render suppression — 8 tests
- Native scrollbar: real OS mouse events via Windows SendInput + CDP — 4 tests
