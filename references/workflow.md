# Execution workflow

All `lark-uml:*` skills follow this loop. Diagram-specific quality rules live inside each `SKILL.md`.

## Pre-flight

1. Load [`lark-shared`](../../lark-shared/SKILL.md) (auth, scopes, `--as` identity).
2. Load [`lark-whiteboard`](../../lark-whiteboard/SKILL.md) (the actual raw `+query` / `+update` surface).
3. Confirm `lark-cli` is reachable: `lark-cli --version`.

## Step 1 — Resolve the board token

| What the user gave you | How to get the `board_token` |
| --- | --- |
| Whiteboard token (`wbcnXXX`) | Use directly. |
| Whiteboard URL | Extract the token from the URL. |
| Docx URL / `doc_id` with an existing whiteboard block | `lark-cli docs +fetch --api-version v2 --doc <URL> --as user`, read `<whiteboard token="..."/>` from the response. |
| Docx URL / `doc_id` without a whiteboard block | `lark-cli docs +update --api-version v2 --doc <doc_id> --command append --content '<whiteboard type="blank"></whiteboard>' --as user`, read `data.new_blocks[0].block_token` from the response (the entry whose `block_type == "whiteboard"`). |

## Step 2 — Read current raw state

```bash
lark-cli whiteboard +query <board_token> --output_as raw --as user
```

- Read raw before modifying an existing board, including rearrange, add-node, add-connector, fix-line, or partial-update tasks.
- For a first-time initialization, still use raw when the board already contains a visual template that should be reused.
- Query a preview image only as a visual aid; it is not the source of truth.
- Do not route existing complex boards through Mermaid / PlantUML / SVG / DSL reconstruction when the task requires continued editing, anchored connectors, or drag-follow behavior.

## Step 3 — Build the fact and template inventories

- Build a system fact inventory from the user's request, repository context, README, API docs, database schema, routes, service names, domain objects, process descriptions, and other available evidence.
- Build a whiteboard template inventory from the raw nodes and, when useful, the preview image: reusable shapes, groups, labels, colors, layout rhythm, connector styles, and existing node ids.
- Compare the two inventories:
  - Present in the system but missing on the board → add native nodes, groups, or connectors.
  - Present on the board but absent from the system → remove, merge, or mark as not applicable. Ask before destructive removal when evidence is weak.
  - Present in both but semantically wrong → correct names, direction, relationship, grouping, and granularity.
- Do not finish by only renaming template nodes when the structure and relationships disagree with the facts.
- Apply the diagram-specific quality rules from this skill's `SKILL.md`.
- Apply the language rules from [`language.md`](./language.md).
- Apply the core boundaries from [`boundaries.md`](./boundaries.md).
- Apply the native connector rules from [`connectors.md`](./connectors.md).

## Step 4 — Modify native raw

- Edit the native `WBDocument` / raw structure directly.
- Preserve useful existing node ids and styles. Avoid unnecessary id churn.
- When adding elements, copy the closest existing whiteboard shape / text / connector style from the template and then adjust content, position, and binding.
- When adding or repairing business relationships, create or modify native `connector` nodes whose endpoints bind to node ids.
- Use external diagram languages only as private reasoning drafts if helpful. Do not write them to the board for tasks that require editable, node-bound connectors.

Allowed write surface:

```bash
lark-cli whiteboard +update <board_token> \
  --source - \
  --input_format raw \
  --overwrite \
  --as user
```

Always pipe through stdin. Do not paste raw JSON into chat as the deliverable.

## Step 5 — Validate before writing

- Validate the final raw against [`connectors.md`](./connectors.md).
- Business relationship connectors must have string `connector.from` and `connector.to` values that refer to existing node ids.
- Coordinate endpoints are allowed only for non-business annotation, decoration, measurement, or helper lines.
- `waypoints` may shape a route but must not replace endpoint binding.
- If the validation fails, do not write back.

## Step 6 — Write and verify

1. Write raw through `+update --input_format raw`.
2. Re-run `+query --output_as raw`.
3. Verify key business connectors are still native `connector` nodes with `node id -> node id` endpoints.
4. Re-run `+query --output_as image` and confirm the layout matches the diagram-type rules.
5. Apply the drag semantics check: if moving a connected node would not keep the line attached, the result is invalid.
6. For layout-strict diagrams (sequence, usecase, class, ER), also verify alignment, lifelines, boundaries, row-level bindings, and relationship directions before reporting done.

## Failure modes to watch for

- Permission denied → check `lark-shared` for scope guidance.
- Token resolves but the block is not a whiteboard → re-confirm with the user.
- `+update` succeeds but the board renders empty → re-check the raw shape, `version`, `nodes`, and stdin piping.
- A line looks connected but does not bind to ids → repair it as a native connector before writing.
- A generated diagram language route would lose editability or anchor semantics → reject that route and continue with raw.
