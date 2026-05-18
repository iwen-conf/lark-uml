---
name: lark-uml:architecture
description: 飞书画板架构图：执行型 Skill。当用户要在飞书画板上绘制或修改系统架构图（系统边界、模块 / 服务、存储、外部依赖、调用 / 数据关系）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 Mermaid / SVG 代码。
---

# `lark-uml:architecture`

Specialist skill for **system architecture diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the architecture diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native nodes and native connectors in place, then write raw back. Architecture call / data-flow relationships are business relationships, so every relationship line must be a native connector whose endpoints bind to service, module, storage, gateway, external-system, or boundary node ids. **No Mermaid / PlantUML / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing boundaries, shapes, and connector styles rather than redrawing. Only redraw when the user explicitly asks, or the template is fundamentally wrong for the system.

## Diagram-specific rules

- **Boundaries first.** Every module sits inside a labeled boundary: a system, a tenant, a domain, a deployment zone, a third-party perimeter. Boundaries are `subgraph`s; nested boundaries are allowed up to two levels deep — beyond that, split the diagram.
- **Element vocabulary.** Use a stable set of shapes:
  - Service / module → rectangle.
  - External system / third party → rectangle with a distinct outline (e.g., dashed border).
  - Storage (DB, cache, queue, object store) → cylinder or labeled rectangle marked `DB` / `Cache` / `Queue`.
  - Edge / gateway → trapezoid or a labeled rectangle.
- **Naming granularity.** Names are at the module / service level, not at the class or function level. `订单服务` is correct; `OrderServiceImpl.placeOrder()` is not.
- **Direction = data / call direction.** Every arrow encodes either a synchronous call or a data flow. Choose one notation per relationship and label it (`HTTP`, `gRPC`, `Kafka`, `read` / `write`). Mixed unlabeled arrows are not acceptable.
- **Upstream / downstream.** Upstream sits above (or to the left of) downstream. External traffic enters from one edge, never from multiple unrelated edges. Sync vs async traffic is visually separable — different line styles (solid / dashed) are fine but must be applied consistently.
- **Deployment hint, not deployment manifest.** Architecture diagrams may indicate deployment context (region, cluster, cloud) via boundary labels but should not become Kubernetes YAML. Keep one level of deployment grouping max.

## Forbidden mixings

- Business process steps — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- Use case actors and boundaries — those belong in `lark-uml:usecase`.
- Network connectivity (subnets, VLANs, firewall rules) — that belongs in `lark-uml:network`.
- Class members or methods — those belong in `lark-uml:class`.

## Native node composition

Build the architecture diagram out of these native whiteboard primitives. Do not express any part of the diagram as Mermaid, PlantUML, or SVG.

- **Boundary** — native group / container, one per system, tenant, domain, deployment zone, or third-party perimeter. Nest at most two levels.
- **Service / module** — native rectangle inside its owning boundary. Same fill / border across same-tier services.
- **External system** — native rectangle with a visually distinct outline (e.g., dashed border) supplied by the template.
- **Storage** — native cylinder, or a rectangle whose label clearly says `DB` / `Cache` / `Queue` / `Object Store`, matching whichever style the template already uses.
- **Edge / gateway** — native trapezoid, or rectangle labeled `网关` / `Gateway`, matching the template.
- **Connector** — native `type: "connector"` whose `connector.from` / `connector.to` reference real node ids. Label the connector with the protocol or data direction (`HTTP`, `gRPC`, `Kafka`, `读` / `写`, `事件`). Sync / async distinction may use line style (solid / dashed) but must stay consistent.

When the template is missing a kind of element, duplicate the closest existing element and adapt it. Do not introduce a foreign visual vocabulary.
