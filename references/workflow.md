# Execution workflow

All `lark-uml:*` skills follow this loop. Diagram-specific quality rules live inside each `SKILL.md`.

**Two execution modes — pick based on intent:**

| Mode | When to use | Key tool |
|------|------------|----------|
| **PlantUML** | First-time creation, clean rebuild, most edits | `whiteboard +update --input_format plantuml --overwrite` |
| **Raw native** | Precise in-place tweak on existing native board, combined-fragment surgery | `whiteboard +query --output_as raw` → edit → `+update --input_format raw` |

**Default to PlantUML.** It is fast, predictable, and lets you focus on diagram content instead of node geometry.

> **⚠️ Do NOT** use `@larksuite/whiteboard-cli` or run `npx whiteboard-cli`. Do NOT create local `./diagrams/` directories or generate local files like `diagram.json`, `diagram.png`, or `diagram.svg`. Pipe directly to `lark-cli`.

## Pre-flight (both modes)

1. Confirm `lark-cli` is reachable: `lark-cli --version`.
2. Load [`lark-shared`](../../lark-shared/SKILL.md) for auth and `--as` identity.

## Mode A: PlantUML (primary)

### A1 — Resolve the board token

| What the user gave you | How to get the `board_token` |
| --- | --- |
| Whiteboard token (`wbcnXXX`) | Use directly. |
| Whiteboard URL | Extract the token from the URL. |
| Docx URL / `doc_id` with an existing whiteboard block | `lark-cli docs +fetch --api-version v2 --doc <URL> --as user`, read `<whiteboard token="..."/>` from the response. |
| Docx URL / `doc_id` without a whiteboard block | `lark-cli docs +update --api-version v2 --doc <doc_id> --command append --content '<whiteboard type="blank"></whiteboard>' --as user`, read `data.new_blocks[0].block_token` (the entry whose `block_type == "whiteboard"`). |

### A2 — Build the system fact inventory

Inspect the target project source and identify:
- For **sequence diagrams**: Browser entry action, Page/View component, Controller method + params, Service method + params, Mapper/Dao method, SQL/ORM operations, return types, branch conditions.
- For **flowcharts**: Business steps, decision points, branch outcomes, error paths.

Use real identifiers from code. Do not invent class names, method names, or table names.

### A3 — Write PlantUML and update

```bash
cat << 'PUML' | lark-cli whiteboard +update \
  --whiteboard-token <token> \
  --input_format plantuml --source - \
  --overwrite --as user
@startuml
...
@enduml
PUML
```

- Always `--overwrite`. One `@startuml` block per call.
- The Feishu parser generates native nodes from PlantUML — the board is fully editable after write.
- Apply the diagram-specific rules from the skill's `SKILL.md`.

### A4 — Verify

```bash
lark-cli whiteboard +query <token> --output_as image --as user
```

Confirm layout, alignment, message numbering, and branch coverage. Report done.

---

## Mode B: Raw native (advanced, in-place edit only)

Follow [`native-agent.md`](./native-agent.md): read raw, modify the existing template in place, write raw, verify native connector binding.

### B1 — Read current raw state

```bash
lark-cli whiteboard +query <board_token> --output_as raw --as user
```

### B2 — Build inventories

- **System fact inventory**: user request, source code identifiers, domain objects.
- **Template inventory**: reusable shapes, groups, labels, colors, layout rhythm, connector styles, existing node ids from the raw JSON.
- Compare: add what's missing, remove what's stale, correct what's wrong.

### B3 — Modify native raw

**Modify the template, do not redraw:**
1. Rename existing nodes when their role maps to the target.
2. Move, resize, reconnect, regroup, split, merge as needed.
3. Create new elements by duplicating the closest existing same-kind element.
4. Preserve unrelated regions and useful ids.
5. After any delete, repair connectors that referenced deleted ids.

### B4 — Validate before writing

- Business connectors must have string `connector.from` / `connector.to` bound to existing node ids.
- Coordinate-only endpoints only for annotations/decorations.
- Apply diagram-specific quality rules from the skill's `SKILL.md`.
- Scan for `elseif` / `else if` — rewrite as native decision diamonds.
- Sequence-specific: walk the validation checklist in `skills/sequence/SKILL.md` for every `combined_fragment`.

### B5 — Write and verify

```bash
lark-cli whiteboard +update <board_token> --source - --input_format raw --overwrite --as user
```

1. Re-query raw and verify connector bindings.
2. Query preview image and confirm layout.
3. If moving a connected node wouldn't keep the line attached → invalid.

## Document co-design (when multiple diagrams live in one doc)

1. Resolve or create the doc token.
2. Write the full document structure with `docs +update --command overwrite`, using `<whiteboard type="blank"></whiteboard>` for each diagram slot.
3. Parse `data.new_blocks` for whiteboard tokens.
4. Fill each whiteboard with separate PlantUML `+update` calls (parallel for independent boards).

Template:
```xml
<title>{Title}</title>
<callout emoji="✅">{Summary}</callout>
<table>...</table>
<hr/>
<h1>1. {Business}</h1>
<p>{Description — single focused action only}</p>
<h2>活动图</h2>
<whiteboard type="blank"></whiteboard>
<h2>时序图</h2>
<whiteboard type="blank"></whiteboard>
<hr/>
```

## Failure modes

- Permission denied → check `lark-shared` for scope guidance.
- `+update` succeeds but board empty → re-check stdin piping and PlantUML syntax.
- PlantUML parse error → check for `newpage`, multi-diagram blocks, or `elseif` token.
- Raw connector not bound to ids → repair as native connector before writing.
