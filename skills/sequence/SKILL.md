---
name: lark-uml:sequence
description: 飞书画板时序图：执行型 Skill。当用户要在飞书画板上绘制或修改时序图 / 序列图（参与者、生命线、消息调用、返回、激活区）时使用。严格要求生命线垂直、消息水平、按时间从上到下单调推进。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 PlantUML / SVG 代码。
---

# `lark-uml:sequence`

Specialist skill for **sequence diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

Sequence diagrams are the most layout-sensitive diagram type in this plugin. The rules below are not stylistic — they are correctness gates.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the sequence diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native participants, lifelines, activations, and native message connectors in place, then write raw back. Sequence messages are business relationships, so endpoints must bind to participant, lifeline, or activation node ids and remain horizontally aligned. **No PlantUML / Mermaid / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing participants, lifelines, activations, message connectors, and combined fragment frames. Only redraw when the user explicitly asks, or the diagram is the wrong type.

## Diagram-specific rules

- **Participant head alignment.** Every participant header (`actor` / `participant` / `object` rectangle) sits on the **same top horizontal baseline**. Identical `y`. Left-to-right ordering. Uniform horizontal spacing. Uniform size. No drifting heads.
- **Lifeline discipline.** Every participant has a vertical lifeline directly below its header. Lifelines are **parallel, strictly vertical, identical length, bottom-`y` aligned**. Each lifeline's `x` is **exactly** the center `x` of its participant header.
- **Messages are horizontal.** Every message connector endpoint is bound to a lifeline (or its participant head). The two endpoints share the **same `y`** (within rendering tolerance). Messages stack top-to-bottom; their `y` coordinates increase monotonically. **No diagonal lines. No freeform polylines. No "point a to point b" geometry that bypasses lifelines.**
- **Self-call.** A participant calling itself draws as a small right-angle loop on the right side of its own lifeline: horizontal out → short drop → horizontal back. Never a long diagonal. Never routed through another lifeline.
- **Activation bar.** The activation rectangle is a narrow strip **centered on the lifeline**, uniform width. It must not overlap the neighbor lifeline. It does not replace the lifeline.
- **Crossings.** A message that visually crosses an intermediate lifeline does **not** change its `y` to dodge — it stays a straight horizontal arrow. Routing tools may render a small jump, but the source `y` is identical at both endpoints.
- **Combined fragments.** Use the whiteboard's **native combined-fragment shape** (`type: "combined_fragment"`, 飞书画板"组合片段"图形) when sequence logic needs conditional, optional, looping, or parallel behavior. **Do not** substitute with plain rectangles, grouped rectangles, or freeform frames — they render without the operator pentagon and the row separators, and break the diagram's semantics. The operator label sits in the top-left pentagon corner, branch guards are written in square brackets inside each region's header, and all messages inside each region still obey the horizontal-message and monotonic-`y` rules. Multi-region operators (`alt`, `par`) must use the fragment's native multi-row layout (`meta.row_num` ≥ 2 with matching `row_sizes`), not stacked separate fragments. Do not flatten branches into a single sequential list of messages.
  - `alt` means alternative / conditional branches: exactly one region runs based on its guard, like `if / else if / else`. When the source facts contain mutually exclusive outcomes, the diagram **must** include an `alt` frame covering those branches. Required triggers include success / failure, approved / rejected, authorized / unauthorized, exists / not found, enough quota / insufficient quota, in stock / out of stock, paid / unpaid, valid / invalid, and any user-described "如果...否则..." logic.
  - `opt` means an optional region: the region runs only when its guard is true, like a single `if` without `else`.
  - `loop` means repeated execution while the guard or count condition holds, like `for` / `while`.
  - `par` means parallel regions that may execute concurrently. Keep each region visually separated and do not imply an ordering between parallel branches unless messages make that ordering explicit.
- **Branch validation.** Before reporting a sequence diagram done, scan the requested flow for mutually exclusive outcomes. If any exist, verify that a visible native `combined_fragment` with `alt` operator is present (not a plain rectangle frame), has at least two guarded regions inside `meta.row_sizes`, and contains the branch-specific messages in their matching regions.
- **Forbidden constructions.**
  - Drawing a single diagonal connector between two participant heads with no lifelines.
  - Replacing "lifeline + horizontal message" with one point-to-point polyline.
  - Letting an arrow's `y` shift while it crosses other lifelines in the source.
  - Floating activation bars not aligned to any lifeline.
  - Showing mutually exclusive branch messages one after another without an enclosing native `combined_fragment` (`alt`) frame.
  - Faking a combined fragment with a plain native rectangle, grouped rectangles, or a freeform polygon — the operator pentagon and region separators are part of the semantics and only the native `combined_fragment` shape provides them.

## Forbidden mixings

- Process branches and swimlanes — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- Use case actors with boundaries — those belong in `lark-uml:usecase`.
- Deployment layering — that belongs in `lark-uml:architecture`.
- Network link topology — that belongs in `lark-uml:network`.

## Native node composition

Build the sequence diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **Participant header** — native rectangle (or actor stick-figure for human actors), one per role. All headers share the same top `y` and uniform spacing. Headers are produced by cloning an existing header.
- **Lifeline** — native vertical line / dashed line directly below its participant header. Lifelines are strictly vertical, identical length, bottom-`y` aligned, with `x` equal to the center `x` of the header.
- **Activation bar** — native narrow rectangle centered on the lifeline. It belongs to the lifeline visually but remains a distinct node so that messages can bind to it.
- **Message connector** — native `type: "connector"` whose `connector.from` and `connector.to` reference participant / lifeline / activation node ids. Both endpoints share the same `y`. Synchronous calls use solid line + filled arrow; returns use dashed line + open arrow. Message text goes in `connector.label`.
- **Self-call** — native connector from a lifeline back to the same lifeline, rendered as a small right-angle loop on the right side. Endpoints still bind to the same node id at top and bottom of the loop.
- **Combined fragment frame** — native **`type: "combined_fragment"`** shape (飞书画板"组合片段"图形) spanning the involved lifelines. The operator (`alt`, `opt`, `loop`, `par`) sits in the top-left pentagon corner via the node's `text` field; multi-region operators carry their regions inside `meta.row_num` / `meta.row_sizes`, with each region's guard label such as `[报名人数未满]` / `[报名人数已满]` written into the per-row header. **Never substitute** a plain rectangle, grouped rectangles, or SVG polygon — those render without the operator pentagon and region separators and fail the diagram's semantics. Fragment frames are annotations around messages; they do not replace lifelines, activation bars, or bound message connectors.

Every message connector must be a native connector with bound endpoints. Diagonal "point-to-point" lines or freeform polylines are not acceptable substitutes.
