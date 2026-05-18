# lark-prompt absorption into lark-uml

## Plan

- [x] Add native connector rules to `lark-uml` shared references.
- [x] Change shared workflow from external diagram source first to raw-first whiteboard edits.
- [x] Update README so `lark-uml` contains the prompt's constraints and no longer depends on `lark-prompt`.
- [x] Tighten all diagram skills to inherit raw/native connector execution and stop recommending Mermaid / PlantUML as the write path.
- [x] Verify there are no remaining runtime references that require `lark-prompt`.

## Review

- Added `references/connectors.md` as the shared hard-rule source for native connector binding, coordinate endpoint limits, waypoint limits, pre-write validation, and post-write verification.
- Updated `references/workflow.md` to require raw query, raw modification, raw update, fact/template inventories, and structural validation.
- Updated all diagram skills so Mermaid / PlantUML examples are private sketches only, not write paths.
