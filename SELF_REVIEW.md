# Self-review — Webex messaging-vs-meeting discriminator (#4145)

Adversarial correctness review of the per-window title guard.

## What changed

- `MeetingDetectionProfile` gains `ignore_window_titles: &'static [&'static str]`. All 13 production profiles + 1 test profile literal updated; only Webex is `&["webex"]`, the rest `&[]`.
- `window_title_is_ignored_titles(window_title, ignore_titles)` — exact, case-insensitive, trimmed equality. `window_title_is_ignored(window_title, profile)` is the profile-scoped wrapper (macOS/test).
- macOS scan loop reads each window's `AXTitle` and `continue`s past a guarded window before walking it.
- Windows scan loop does the same via `element.CurrentName()`; the ignore list is threaded into `windows_scan_process_uia`.

## False-negative risk (silencing a REAL meeting)

The only way the guard silences a real meeting is if a real meeting window's title is **exactly** `Webex` (case-insensitively, after trim). Per the empirical data in #4145, a real Webex meeting window is titled with the meeting/space name and is *never* bare `Webex`. Mitigations:

- **Exact-equality, not substring.** A meeting titled "Webex Standup" or "Cisco Webex Meeting" contains "webex" but is NOT guarded. Test: `real_webex_meeting_window_is_not_guarded`. A substring guard would have re-introduced the ambiguity.
- **Per-window, not per-process.** A messaging window open *alongside* a meeting window does not suppress the meeting window — each window is judged on its own title. Test: `webex_messaging_open_during_meeting_still_detects_meeting`.
- **Residual risk:** if a Webex version ever titled a meeting window literally `Webex` (e.g. a meeting with no name before the title resolves, or a transient state), that window's signals would be dropped for that scan. Bounded: the meeting still produces screen capture; the next scan after the title resolves to the meeting name would detect it; and audio batch transcription remains a backstop. This matches the empirical discriminator the maintainer endorsed; if it proves wrong in the field, the belt-and-suspenders fallback (require a *meeting-specific* signal like "leave meeting" in addition) is a one-line follow-up. Flagged, not silently assumed.

## False-positive risk (phantom meeting during chat)

The bug was: chat chrome ("Leave space"/"Leave team") matches the bare `leave` signal. The guard fires *before* the tree walk for any window titled exactly `Webex`, so those signals never accumulate. Test `webex_messaging_leave_chrome_does_not_start_meeting` first asserts the chrome DOES match a signal (pins the root cause) then asserts the guard suppresses it.

- **Title casing/whitespace:** AX/UIA may report `Webex`, `webex`, or padded. Guard trims and compares case-insensitively. Test `bare_webex_messaging_window_is_guarded`.
- **Missing title:** a window with no readable title (`None`) or empty title is **not** guarded — absence of a title can't prove it's the messaging window, and we'd rather risk detecting than wrongly silence. Covered in `real_webex_meeting_window_is_not_guarded`.

## Edge cases considered

- `None` / empty / whitespace-only title → not guarded (returns false early).
- Other profiles with `&[]` → guard is a no-op (early `is_empty()` check skips the AXTitle IPC entirely, so zero perf cost for non-Webex apps). Test `window_title_guard_is_noop_for_profiles_without_ignore_list`.
- Webex-in-browser (`webex.com`) → unaffected: the guard only runs in the native AX/UIA scan path (`scan_process`), not the browser-URL detection path. Browser meetings match by URL, not by native window title.
- Simultaneous messaging + meeting windows → per-window scope handles it (test above).

## Why the tests suffice (and what they don't cover)

The engine's AX/UIA tree-walk is `#[cfg(target_os = ...)]` FFI that can't run in a unit test. The tests therefore exercise the two **pure decision functions** the live scan composes per window: `window_title_is_ignored` (the guard) and `check_signal_match` (the existing per-node matcher, used on Windows + tests). `any_signal_matches` + the guard reproduce the exact per-window scan outcome the live loop computes. This is the same testing strategy the existing `ignoredMeetingApps` tests use.

Not covered by unit tests (and out of scope for this automated change, tracked in #4145):
- Real Webex AX/UIA labels on macOS + Windows (the issue requests captured evidence). If Webex ever renames its messaging window away from bare `Webex`, the guard simply stops firing (fails open to current behavior) — it cannot cause a new false-negative beyond the residual above.
- An eval-crate replay trace for `crates/screenpipe-meeting-eval`.

## Build / format

- `rustfmt --edition 2021` applied to the changed file.
- `cargo test -p screenpipe-engine meeting_detector::tests` → 112 passed, 0 failed (105 existing + 7 new). Cold build clean, no new warnings on the changed code.
