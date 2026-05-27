# Add native Slides / PPT skill

## Plan

- [x] Inspect existing skill conventions and shared lark-cli rules.
- [x] Confirm current `lark-cli slides` command surface and schema shape.
- [x] Add `lark-uml:ppt` as an execution-oriented Feishu Slides skill.
- [x] Update README skill list, requirements, and repository layout.
- [x] Verify Markdown files, command references, and git diff.
- [x] Tighten PPT skill for 12-15 pages, numeric text, replaceable image slots, and native Slides-only delivery.
- [x] Add native SML 2.0 reference and prohibit PPTX/import fallback after real create failures.

## Review

- Added `skills/ppt/SKILL.md` with an execution-oriented Feishu Slides workflow, content rules, visual direction, payload-only mode, and quality gates.
- Confirmed local `lark-cli slides` supports `+create`, `+replace-slide`, `+media-upload`, and `xml_presentation.slide.*`; checked create / replace schemas.
- Updated README and plugin metadata so the plugin describes both native whiteboard UML and native Slides skills.
- Updated the PPT skill defaults to 12-15 pages, require numeric expressions on content slides, require replaceable image slots when images are used, and explicitly forbid stopping before native Feishu Slides write operations.
- Added `references/slides-native.md` with the SML 2.0 minimum visible slide structure, native two-step creation route, image-token rules, replace rules, partial-create recovery, and read-back validation requirements.
- Updated `skills/ppt/SKILL.md` so real decks default to blank native deck creation plus `xml_presentation.slide.create`, require schema/reference review before XML generation, and treat successful API responses with empty `<data/>` as failures.

# Fix PlantUML activity-branch constraints

## Plan

- [x] Inspect existing diagram constraints and search for `elseif` / activity diagram sources.
- [x] Tighten flowchart / activity diagram rules so native whiteboard drawing cannot fall back to PlantUML.
- [x] Add explicit PlantUML fallback syntax gate: no `elseif`; use nested `if` / `else` / `endif`.
- [x] Record the lesson so future diagram fixes check this before drawing.
- [x] Verify repository text no longer contains an allowed `elseif` path.

## Review

- Root cause: `lark-uml:flowchart` did not explicitly cover activity diagrams or the known PlantUML parser limitation, so a caller could still leak `elseif` / `else if` pseudo-code into activity drawing despite the native-whiteboard rule.
- Updated `skills/flowchart/SKILL.md` to cover flowchart / activity diagrams, reject PlantUML activity source, and require native multi-way decisions or nested `if` / `else` / `endif` only for forced external compatibility paths.
- Added shared hard gates in `references/boundaries.md`, `references/workflow.md`, and `references/native-agent.md` so `elseif` / `else if` is treated as leaked pseudo-code during flowchart / activity work.
- Removed the unrelated explanatory `else if` wording from `skills/sequence/SKILL.md` to avoid teaching the model that spelling.
- Added the recurrence rule to `tasks/lessons.md`.
- Verified with `rg --hidden -n "elseif|else if"` and `git diff --check`; remaining matches are all prohibition / validation / lesson text, not allowed syntax examples or diagram sources.
