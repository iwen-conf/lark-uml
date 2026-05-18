---
name: lark-uml:swimlane
description: 飞书画板泳道图：执行型 Skill。当用户要在飞书画板上绘制或修改泳道图（按角色 / 团队 / 系统 / 阶段切分责任归属、跨泳道流转、异常回流）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 Mermaid / PlantUML / SVG 代码。
---

# `lark-uml:swimlane`

Specialist skill for **swimlane diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the swimlane from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native lane groups, action nodes, decision nodes, and native connectors in place, then write raw back. Cross-lane handoffs, branch arrows, and re-entry paths are business relationships, so every endpoint must bind to node ids. **No Mermaid / PlantUML / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt existing lane groups and existing node / connector elements rather than redrawing. Only redraw when the user explicitly asks, or the template is fundamentally wrong.

## Diagram-specific rules

- **Lane axis.** Organize lanes by role, team, system, or stage. Each lane is one `subgraph`. Lanes flow in one consistent direction (top-to-bottom or left-to-right) — pick one and keep it.
- **Node semantics.** Nodes inside a lane express actions, decisions, or cross-lane handoffs. Decisions are diamonds, actions are rectangles, start / end are stadium / circle shapes — keep the same shape for the same role across all lanes.
- **Responsibility binding.** Every action belongs to exactly one lane. Do not float a node between lanes or stretch a node across a lane boundary.
- **Cross-lane connections.** Cross-lane arrows must explicitly exit one lane and enter another — do not draw "lane-less" floating arrows. Label the arrow with the trigger (event, status, message) when the handoff is not trivially obvious.
- **Exception flow-back.** Failure / rejection paths must explicitly return to their handling lane with a labeled arrow (`失败回流` / `Rolled back` etc.). Do not leave a dangling failure branch.
- **Order discipline.** Stage order within a lane must match the upstream / downstream sequence implied by the cross-lane arrows. No backward jumps without an explicit re-entry label.

## Forbidden mixings

- Use case ovals or actor figures — those belong in `lark-uml:usecase`.
- Network device legends — those belong in `lark-uml:network`.
- Pure deployment layering (web tier / app tier / db tier) — that belongs in `lark-uml:architecture`.
- Class attributes / methods — those belong in `lark-uml:class`.

## Native node composition

Build the swimlane out of these native whiteboard primitives. Do not express any part of the diagram as Mermaid, PlantUML, or SVG.

- **Lane** — native group / container shape (one per role, team, system, or stage). Lane header text uses the role name in business language. New lanes are created by duplicating an existing lane and editing its header.
- **Action** — native rectangle, placed entirely inside one lane. Every action belongs to exactly one lane; never straddle a boundary.
- **Decision** — native diamond inside the lane that owns the decision. One incoming connector, two or more outgoing connectors, each labeled in business Chinese.
- **Start / End** — native stadium / circle shape inside the lane where the flow begins or terminates.
- **Cross-lane connector** — native `type: "connector"` with `connector.from` / `connector.to` bound to node ids in different lanes. Anchors must reflect lane geometry (e.g., right anchor of source lane → left anchor of target lane). Label the connector with the trigger when the handoff is not obvious.
- **Failure / re-entry connector** — same native connector primitive, with a Chinese label like `失败回流` / `驳回返工`. Endpoints must still bind to real node ids; never leave a dangling tail.

New lanes, nodes, and connectors are produced by duplicating the closest existing same-kind element and adapting it.
