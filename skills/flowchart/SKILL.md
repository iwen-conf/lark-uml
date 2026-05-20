---
name: lark-uml:flowchart
description: 飞书画板流程图：执行型 Skill。当用户要在飞书画板上绘制或修改流程图（开始 / 结束、处理步骤、判断分支、回退路径、异常处理）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 Mermaid / SVG 代码。
---

# `lark-uml:flowchart`

Specialist skill for **flowcharts** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the flowchart from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native flow nodes and native connectors in place, then write raw back. Step, decision, branch, rollback, and exception arrows are business relationships, so every arrow must bind to source and target node ids. **No Mermaid / PlantUML / SVG anywhere in the loop, not even as a private sketch** — reason directly about native node ids and native connectors.

**Default mode is modify-in-place.** Duplicate and adapt existing template elements rather than redrawing the board. Only redraw from scratch when the user explicitly asks for a clean rebuild, the existing diagram type is wrong, or the structure is too corrupted to edit safely.

## Diagram-specific rules

- **Anchored start and end.** Exactly one start node and one end node per primary flow. Start / end use the stadium or circle shape; processing steps use the rectangle; decisions use the diamond. Same role → same shape across the whole diagram.
- **Main flow first.** The happy path is unambiguous and laid out top-to-bottom (or left-to-right) in a single dominant direction. Side branches peel off and either rejoin the main flow or terminate cleanly.
- **Decision branches.** Every diamond has exactly **one** incoming edge and **two or more** outgoing edges. Every outgoing edge carries an explicit branch label written in business language (`是` / `否`, `通过` / `不通过`, `Yes` / `No`). Branches must be mutually exclusive and cover every possible outcome.
- **Merge points.** When multiple branches converge, route them through a merge node (or a clearly labeled junction). Do not drop several incoming arrows onto the same processing step without a visible merge.
- **Rollback and exception paths.** A failure / rollback branch must terminate at a real handler (end node, retry step, or back-into-main with a labeled re-entry). Never leave a branch dangling.
- **Direction discipline.** Arrows follow the dominant flow direction. Explicit backward jumps (retries, loops) must be labeled with the condition that triggers them.

## Forbidden mixings

- Swimlane responsibility partitions — those belong in `lark-uml:swimlane`.
- Use case ovals or system boundaries — those belong in `lark-uml:usecase`.
- Network devices — those belong in `lark-uml:network`.
- Sequence messages / lifelines — those belong in `lark-uml:sequence`.

## Native node composition

Pick every native `type` from the matrix in [`../../references/native-types.md`](../../references/native-types.md) (see the flowchart row). Start / end, process steps, decisions, and merges are all `composite_shape` variants (`stadium` / `round_rect` / `diamond` / `circle`) — never `cylinder` / `actor` / `table_uml`, those belong in other skills. Branch labels live in `connector.captions`, not as floating `text_shape`.

Build the flowchart out of these native whiteboard primitives. Do not express any part of the diagram as Mermaid, PlantUML, or SVG — these are descriptions of what to put into raw, not a syntax to write.

- **Start / End** — native rounded / stadium / circle shape, single instance each per primary flow.
- **Process step** — native rectangle. Same fill / border style across all steps; new steps clone an existing step.
- **Decision** — native diamond. Exactly one incoming connector, two or more outgoing connectors, each carrying a Chinese branch label (`是` / `否`, `通过` / `不通过`, `成功` / `失败`).
- **Merge / junction** — native small circle or labeled rectangle; do not drop multiple arrows onto the same step without one.
- **Connector** — native `type: "connector"` between two existing node ids, with `connector.from` / `connector.to` bound to ids, anchors consistent with relative position, and `polyline` or `rightAngle` routing by default.
- **Branch label** — `connector.label` (or the label child the connector style already uses on this board), written as business-language Chinese.

When extending the diagram, copy the closest existing shape, text, or connector and adapt content, position, and binding — do not invent a new visual vocabulary on the board.
