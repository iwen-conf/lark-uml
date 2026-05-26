# Add native Slides / PPT skill

## Plan

- [x] Inspect existing skill conventions and shared lark-cli rules.
- [x] Confirm current `lark-cli slides` command surface and schema shape.
- [x] Add `lark-uml:ppt` as an execution-oriented Feishu Slides skill.
- [x] Update README skill list, requirements, and repository layout.
- [x] Verify Markdown files, command references, and git diff.
- [x] Tighten PPT skill for 12-15 pages, numeric text, replaceable image slots, and native Slides-only delivery.

## Review

- Added `skills/ppt/SKILL.md` with an execution-oriented Feishu Slides workflow, content rules, visual direction, payload-only mode, and quality gates.
- Confirmed local `lark-cli slides` supports `+create`, `+replace-slide`, `+media-upload`, and `xml_presentation.slide.*`; checked create / replace schemas.
- Updated README and plugin metadata so the plugin describes both native whiteboard UML and native Slides skills.
- Updated the PPT skill defaults to 12-15 pages, require numeric expressions on content slides, require replaceable image slots when images are used, and explicitly forbid stopping before native Feishu Slides write operations.
