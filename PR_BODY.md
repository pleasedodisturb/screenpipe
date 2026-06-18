## description

Webex is used by many orgs as **both** a messaging app and a meeting app, under one process. The meeting detector couldn't tell the two apart, so leaving Webex open for chat triggered phantom meetings (and unwanted audio transcription). This adds a per-window title discriminator so the detector fires for real Webex **meetings** but stays quiet during Webex **messaging** — no manual per-call action from the user.

related issue: #4145

### root cause

Webex profile (`crates/screenpipe-engine/src/meeting_detector.rs`):

```rust
call_signals: vec![
    CallSignal::AutomationIdContains("leave"),
    CallSignal::RoleWithName { role: "AXButton", name_contains: "leave" },
    CallSignal::RoleWithName { role: "AXButton", name_contains: "end meeting" },
],
min_signals_required: 1,
```

The bare `"leave"` substring matches Webex's **messaging** chrome — Spaces/Teams expose "Leave space" / "Leave team" — so a single match in chat mode is enough to declare a call active. Same class of false positive as the historical `"end"` matching "Send" (documented in the Zoom profile comments in the same file).

The empirical discriminator (from a user's own screenpipe DB, in #4145):
- Webex **messaging** window title is exactly `Webex`.
- Webex **meeting** window title is the meeting/space name (e.g. `Project Update Call`, `VHM Calendar Hold`), never `Webex`.

### change

Implements the issue's **primary (robust) approach**: a per-window title guard.

- New per-profile field `ignore_window_titles: &'static [&'static str]`, set to `&["webex"]` for Webex only (every other profile is `&[]`, a no-op).
- New helper `window_title_is_ignored_titles(window_title, ignore_titles)` — returns true when a window's title (trimmed, case-insensitive) is **exactly equal** to an ignored title. Exact-equality, not substring, on purpose: a meeting named "Webex Standup" must still be detected.
- Wired into **both** scan paths as a **per-window** guard (not per-process): the macOS AX scan reads each window's `AXTitle` before walking it and `continue`s past a guarded window; the Windows UIA scan does the same with `element.CurrentName()`. Because it's per-window, signals from a bare-`Webex` messaging window are dropped while a real meeting window of the *same process* is still scanned — so a meeting open alongside a chat window is still detected.

### desired behavior (from #4145)

| Webex state | Screen capture | Meeting + audio transcription |
| --- | --- | --- |
| Messaging (window title == `Webex`) | yes (unchanged) | **no** (was: phantom meeting) |
| Meeting (window title == meeting name) | yes (unchanged) | **yes** (unchanged) |

Screen/OCR capture is independent of meeting detection and stays on in both states. This only changes whether a meeting session (and its audio transcription) auto-starts.

## before

Webex open for chat → the messaging window's "Leave space" / "Leave team" button matches the bare `leave` signal → `min_signals_required: 1` met → phantom meeting starts → audio transcription begins, with no real call.

## after

The same chat window is titled exactly `Webex`, so the per-window guard drops its signals before the tree is walked → no phantom meeting. A real meeting window (titled with the meeting name) is unaffected → still detected within the normal interval. With both windows open at once, the meeting is still detected (per-window scope).

## how to test

The engine runs in-process; the proof here is the eval/unit tests (a true manual test requires proving the running binary contains the fix, which is out of scope for an automated change). Real-app AX/UIA capture on macOS + Windows is requested in #4145 and should accompany a follow-up if signal labels ever drift.

```
export CARGO_TARGET_DIR=<shared-target>
cargo test -p screenpipe-engine meeting_detector::tests
```

New tests (all passing, 7 of them, alongside the existing 105 in the module → 112 total):

1. `webex_profile_ignores_bare_webex_window_title` — the loaded Webex profile carries `ignore_window_titles: ["webex"]`.
2. `bare_webex_messaging_window_is_guarded` — `Webex` / `webex` / `  Webex  ` are all guarded.
3. `real_webex_meeting_window_is_not_guarded` — `Project Update Call`, `VHM Calendar Hold`, `Webex Standup` (substring), `Cisco Webex Meeting`, empty, and `None` are NOT guarded.
4. `webex_messaging_leave_chrome_does_not_start_meeting` — proves "Leave space"/"Leave team" DO match a call signal (the bug), then proves the window guard suppresses them → no phantom.
5. `real_webex_meeting_still_detected` — a meeting-titled window with a "Leave Meeting" button (and a `leave-call-button` automation id) is still detected.
6. `webex_messaging_open_during_meeting_still_detects_meeting` — both windows open; per-window scope still detects the meeting.
7. `window_title_guard_is_noop_for_profiles_without_ignore_list` — every other profile (empty ignore list) is never suppressed.

```
running 6 tests
test meeting_detector::tests::webex_profile_ignores_bare_webex_window_title ... ok
test meeting_detector::tests::bare_webex_messaging_window_is_guarded ... ok
test meeting_detector::tests::real_webex_meeting_window_is_not_guarded ... ok
test meeting_detector::tests::webex_messaging_open_during_meeting_still_detects_meeting ... ok
test meeting_detector::tests::webex_messaging_leave_chrome_does_not_start_meeting ... ok
test meeting_detector::tests::real_webex_meeting_still_detected ... ok
test result: ok. 6 passed; 0 failed; ...

running 112 tests
test result: ok. 112 passed; 0 failed; 0 ignored; ...
```

### scope / non-goals

- Webex-specific only. Broader "dual-use app" hardening (a generic title-shape gate) is a separate concern.
- Does not touch the existing `ignoredMeetingApps` opt-out (the all-or-nothing fix from #3847) — that still works for users who only ever use Webex for messaging.
- An eval-crate replay trace (`crates/screenpipe-meeting-eval`) and real-app AX/UIA evidence on both OSes are tracked in #4145's acceptance criteria; the unit tests give regression protection without a live app on every run.

## desktop app checklist (if applicable)

No `#[tauri::command]` handlers or exported Rust types changed.
