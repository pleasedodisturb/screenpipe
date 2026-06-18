## description

`frames.accessibility_tree_json` — the full accessibility tree, holding the raw text of every node — was never redacted. The async PII worker scrubbed `frames.accessibility_text` and `frames.full_text`, but the same content survived in the tree JSON, which `/frames/:id/context` serves verbatim. So with "AI PII removal" on, `accessibility_text` read `[EMAIL]` while the same email stayed readable in the tree JSON for that frame.

This fixes the data **at rest** (issue suggestion (a)): the tree JSON is now a redaction surface. The fix reuses the existing per-frame "detect once, propagate" path (screenpipe/website#291) — the tree is a derived copy of the same screen content the worker already detects on `full_text`, so we scrub its node-text fields with the **same** `RedactionMap`, no extra model pass, and stamp an `accessibility_tree_redacted_at` watermark consistent with the destructive-overwrite model.

**Both redactor backends scrub the tree.** The worker's full_text pass has two arms: the span-aware backend yields a `RedactionMap` and the tree is scrubbed via map propagation (the path above); the **span-less enclave backend** (Tinfoil) can't expose detections as a map (`redact_with_map` returns `None`), so it takes a fallback arm. A second security audit found that fallback arm scrubbed `full_text` and stamped `full_text_redacted_at` but never touched the tree — and because the fetch query filters `WHERE full_text_redacted_at IS NULL`, the frame was never re-selected, so the raw tree (same PII) was served forever for enclave users. The fallback arm now drives the redactor directly over each allowlisted node-text field (`redact_tree_json_with_redactor`, one batched detection over all fields, reusing the same allowlist + tree walk), writes the scrubbed tree, and stamps `accessibility_tree_redacted_at` **before** stamping full_text — so a frame is never marked done while its tree still holds raw text. A malformed tree leaves the whole row pending for retry rather than half-stamping it.

**Scope decision (a over b):** scrub-on-serve (b) is lighter but leaves raw PII at rest — it would still leak through any other reader of the column (pipes, future endpoints, a DB copy). The worker already runs the detection for `full_text`/`accessibility_text` on every frame, so adding the tree to that same pass costs one extra `RedactionMap::apply` (string scan, microseconds) and one UPDATE — no extra inference. Fixing at rest closes the class of bug for good.

**What's covered (deliberately scoped, per the issue's STOP note):** only the node fields that hold free-form human-readable text — `text`, `value`, `help_text`, `placeholder`, `role_description`, `url`. Structural metadata (`role`, `depth`, `bounds`, `on_screen`, booleans, `automation_id`, `class_name`, `subrole`, `accelerator_key`, `access_key`) is left untouched — it isn't PII-bearing free text and redacting it would corrupt search / overlay rendering. This is the common node-text field set, **not** a universal JSON walker. New free-text *scalar-string* node fields just get added to `REDACTABLE_FIELDS` (an array-shaped free-text field would need walker support too — noted at the allowlist).

When a node's `text` is redacted its sibling `lines[]` array (char offsets into the *original* text) is dropped, since those offsets desync the moment the text length changes; the overlay consumer falls back to the node's paragraph bbox rather than mis-highlighting.

related issue: #4116

## before

`GET /frames/{id}/context` for a frame whose `full_text` was redacted returns `accessibility_text: "[EMAIL]"` but `accessibility_tree_json` still contains the raw email in node `text`. Repro: `select count(*) from frames where accessibility_tree_redacted_at is null and full_text_redacted_at is not null` → all already-served frames; `select accessibility_tree_json from frames where id=<redacted frame>` shows the raw value.

## after

The worker, in the same pass that redacts `full_text`, rewrites `accessibility_tree_json` so every node-text field carrying a detected PII value reads its `[LABEL]`/pseudonym placeholder; structure (roles, depths, bounds, booleans) is byte-for-byte preserved; `accessibility_tree_redacted_at` is stamped. `/frames/:id/context` then serves the redacted tree automatically (no endpoint change needed — the fix is at rest). Verified by the integration test `frame_fulltext_redaction_propagates_to_tree_json` (map path): after the worker runs, the raw secret is absent from the tree JSON, `[SECRET]` is present, the JSON still parses as the same 2-node array with roles/`on_screen` intact, and detection ran exactly **once** (no per-node model pass). The span-less enclave path is covered by `frame_fulltext_no_map_path_also_scrubs_tree_json`: driving the worker with a span-less redactor (`redact_with_map` => `None`), the tree's node text is scrubbed, `accessibility_tree_redacted_at` is stamped, structural fields and full_text are intact, and the stale `lines` array is cleared.

## how to test

1. `cargo test -p screenpipe-redact` — covers the new `tree_json` unit tests (redaction of all six fields, structure preservation, malformed-JSON-doesn't-panic, empty-map no-op, nested children) and the worker integration test that drives a real `frames` row end-to-end through the worker and asserts the tree JSON is scrubbed via propagation.
2. Optional manual: enable "AI PII removal", let the worker run, then `sqlite3 ~/.screenpipe/db.sqlite "select accessibility_tree_json from frames where accessibility_tree_redacted_at is not null limit 1"` — node text shows `[LABEL]` placeholders, structure intact.
3. `GET http://localhost:3030/frames/<id>/context` for a redacted frame — `accessibility_tree_json` no longer contains the PII that `accessibility_text` already redacts.

## desktop app checklist (if applicable)

Not applicable — no `#[tauri::command]` handlers or frontend-exported Rust types changed. This is a backend redaction + DB-migration change in `screenpipe-redact` / `screenpipe-db`.
