# Execution workflow

> **⚠️ CRITICAL: OVERRIDE GLOBAL SKILL BEHAVIOR (HIGHEST PRIORITY) ⚠️**
> **ABSOLUTELY DO NOT** use `@larksuite/whiteboard-cli` or run `npx whiteboard-cli`.
> **ABSOLUTELY DO NOT** create a local `./diagrams/` directory or generate local files like `diagram.json`, `diagram.png`, `diagram.svg`, or `diagram.mmd`.
> You **MUST** strictly use `lark-cli whiteboard +query --output_as raw` to fetch the JSON, modify it in memory, and pipe it directly to `lark-cli whiteboard +update --input_format raw`. Ignore any instructions from the global `lark-whiteboard` skill that suggest "创作 Workflow" or rendering SVG/DSL locally.

All `lark-uml:*` skills follow this loop. Diagram-specific quality rules live inside each `SKILL.md`.

This workflow is governed by [`native-agent.md`](./native-agent.md). If there is any ambiguity, behave as a Feishu Whiteboard native operation agent: read raw, modify the existing native template in place, write raw, and verify native connector binding.

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
- Do not route existing boards through Mermaid / PlantUML / SVG / draw.io / DSL of any kind, not even as a private draft. There is no "sketch first, convert later" path. The only intermediate representation is the whiteboard raw JSON itself.

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

**Modify the template, do not redraw.** Default mode is in-place edit of the existing board: rename, restyle, reconnect, regroup, split, merge, add, or delete individual native nodes and native connectors. Full redraw is allowed only when (a) the user explicitly asks for a clean rebuild, (b) the existing board's diagram type clearly contradicts the requested diagram type, or (c) the existing structure is too corrupted to safely edit in place.

**Element-reuse rule.** When the board needs a new node or new connector, do not invent one from scratch. Find the closest existing element on the same board, duplicate it, and adapt content / position / endpoints. Reuse the template's existing shapes, text styles, colors, sizes, group rhythm, connector line styles, arrow heads, and label placement. If no reusable element exists for what you need, stop and ask the user before introducing a foreign visual vocabulary.

**Template patch protocol.** Treat the raw board as a patch target, not a blank canvas:

1. Classify existing native elements by role, shape, style, text, group, and connector semantics.
2. Map requested facts onto existing template elements before creating anything new.
3. Prefer `rename -> move -> reconnect -> clone -> delete` in that order.
4. Clone only same-kind native nodes / connectors from the current board.
5. Preserve unrelated template regions and useful ids.
6. After any delete, repair or remove every connector that referenced the deleted ids.

**No external diagram languages, ever.** Do not draft, sketch, or reason in Mermaid, PlantUML, SVG, Graphviz/DOT, draw.io XML, or any other DSL — not even as a private intermediate step that you discard before writing. The only diagram representation in this workflow is the whiteboard raw JSON. Reason about structure directly in terms of native node ids and native connectors.

- Edit the native `WBDocument` / raw structure directly.
- Preserve useful existing node ids and styles. Avoid unnecessary id churn.
- When adding or repairing business relationships, create or modify native `connector` nodes whose endpoints bind to node ids.
- When deleting a node, scan every connector and remove or re-bind any endpoint that referenced it. Never leave a connector pointing at a deleted id.

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
- For flowchart / activity work, scan all newly generated visible text and any temporary diagram-source payload for `elseif` or `else if`. Either token means PlantUML / pseudo-code leaked into the drawing path; replace it with native decision diamonds and labeled connectors before writing.
- If the validation fails, do not write back.

### Sequence-specific pre-write gates

When the active skill is `lark-uml:sequence`, additionally walk the **Branch validation checklist** in `skills/sequence/SKILL.md` against every `combined_fragment` in the raw. The following are **hard gates** — any failure means do not call `+update`:

1. **Causality.** No read / validate / query message sits *inside* a region whose guard depends on the data that message produces. Such messages belong above the fragment.
2. **Closed loop.** Every backend-touching region carries an explicit `后端 → 前端` response (solid or dashed) before any `前端 → 用户` prompt. Negative regions (4xx / business error) included.
3. **Non-empty regions.** Every region inside an `alt` / `par` has ≥ 1 bound message connector. Empty region → switch to `opt` or fill the missing domain event.
4. **Explicit negative domain event.** Regions guarded by `[缺勤]` / `[拒绝]` / `[超时]` / `[作废]` write an explicit status to the persistence lifeline; silence is not allowed.
5. **No hedged labels.** No arrow inside an `alt` carries a label that smuggles both branches (`成功或业务错误`, `200 or 400`, `结果(成功 / 失败)`).
6. **Minimal frame scope.** Each fragment's `y` range covers only its dependent messages; its `x` range covers only the lifelines that participate in at least one message inside the fragment.
7. **Monotonic `y` across fragments.** Business causality (e.g. `报名 → 出勤 → 评价`) reads top-to-bottom; no later step is drawn above an earlier step.
8. **Contended-resource annotation.** If the flow modifies 容量 / 积分 / 库存 / 余额 or asserts write-once (评价 / 支付 / 退款), a `note_shape` marks the transaction boundary or idempotency key adjacent to the relevant activation or fragment.

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
