---
name: lark-uml:usecase
description: 飞书画板用例图：执行型 Skill。当用户要在飞书画板上绘制或修改用例图（actor 与系统边界、用例椭圆、include / extend / 泛化关系）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 PlantUML / SVG 代码。
---

# `lark-uml:usecase`

Specialist skill for **use case diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the use case diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native actors, use case ovals, boundaries, and native connectors in place, then write raw back. Actor associations, include / extend arrows, and generalization relationships are business relationships, so endpoints must bind to actor or use case node ids. **No PlantUML / Mermaid / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing actors, ovals, system boundary, and relationship connectors. Only redraw when the user explicitly asks, or the diagram is fundamentally the wrong type.

## Diagram-specific rules

- **System boundary is a rectangle.** All use case ovals **MUST** sit inside the rectangle. All actors **MUST** sit outside. Never put an actor inside the boundary; never put a use case outside.
- **Actor layout.** Actors form vertical columns on the left and / or right side of the boundary. Same-side actors share an exact `x` coordinate; their `y` spacing is uniform. No drifting, no skewed placement.
- **Use case layout.** Inside the boundary, ovals lay out in a tidy multi-column grid or a single vertical column. Row spacing, column spacing, and oval size are uniform — no oversized "hero" ovals, no random rotation.
- **Connection semantics.**
  - Actor ↔ use case: straight association line, no arrow or open arrow head only.
  - Use case ↔ use case: `<<include>>` and `<<extend>>` use dashed arrows with the relationship label written out explicitly.
  - Generalization: solid line with a hollow triangle arrow head.
- **Endpoint binding.** Every relationship endpoint must bind to a real actor `id` or a real use case `id`. No free-floating connector tips.
- **Routing discipline.** Place an actor and its related use cases on the same side of the boundary. Avoid lines that span the entire boundary. Avoid heavy crossings. Multiple lines from the same actor should fan out from a shared anchor instead of leaving the actor on four different sides.

## Forbidden mixings

- Process step lanes — those belong in `lark-uml:swimlane`.
- Deployment node hierarchy — that belongs in `lark-uml:architecture`.
- Network link topology — that belongs in `lark-uml:network`.
- Sequence messages or lifelines — those belong in `lark-uml:sequence`.

## Native node composition

Build the use case diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **System boundary** — native rectangle with the system name in the header. All use case ovals sit inside it.
- **Actor** — native stick-figure (or rectangle labeled with the actor name if the template uses that style). All actors sit outside the boundary, lined up on the left and / or right side.
- **Use case** — native oval / ellipse inside the boundary, labeled with a verb-phrase use case name in business Chinese.
- **Association connector** — native `type: "connector"`, straight line, no arrowhead or open arrowhead, from actor id to use case id. Uniform style across all actor → use case lines.
- **Include / extend connector** — native connector, dashed line with open arrow, `connector.label` of `«include»` / `«extend»` (or whichever marker the template already uses).
- **Generalization connector** — native connector, solid line, hollow triangle arrowhead, from specific to general actor / use case.

When adding actors or use cases, clone the closest existing one and adjust label, position, and connector bindings. Multiple lines from a single actor should fan out from one side, not surround the actor on four sides.
