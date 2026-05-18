---
name: lark-uml:er
description: 飞书画板数据库关系图：执行型 Skill。当用户要在飞书画板上绘制或修改数据库关系图 / ER 图（表 / 字段 / 主键外键 / 约束 / 表间关系基数）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 PlantUML / SVG 代码。本图与系统主要类图（lark-uml:class）严格区分。
---

# `lark-uml:er`

Specialist skill for **database relationship (ER) diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

This skill is **not** a software-design class diagram. For business objects with methods, Services, Controllers, inheritance / aggregation / composition, use `lark-uml:class` instead.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the ER diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native entity / table shapes and native connectors in place, then write raw back. Foreign-key, cardinality, and table / field relationships are business relationships, so endpoints must bind to the relevant entity or row node ids. **No PlantUML / Mermaid / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing entity tables, field rows, and relationship connectors. Only redraw when the user explicitly asks, or the diagram is the wrong type entirely.

## Diagram-specific rules

- **Entities are tables.** Every entity is a header-rectangle: the table name in the header row, fields stacked underneath. Same-style headers across all entities; same-style rows across all entities; identical column widths within one entity.
- **Field columns.** Each field row shows, in order: field name, type, constraint markers. Constraint markers use the standard set: `PK` (primary key), `FK` (foreign key), `NN` (not null), `UQ` (unique), `IDX` (indexed). One marker per role, no ad-hoc abbreviations.
- **Primary and foreign keys.** Always mark them. A foreign key row must visually point at the referenced primary key — bind the relationship line endpoints to the actual `PK` / `FK` rows, not just to the entity header.
- **Cardinality.** Every relationship line carries cardinality on both ends, written in crow-foot or `1..1` / `1..*` / `0..*` / `0..1` form. Pick one notation per diagram and keep it consistent.
- **Many-to-many.** Always materialize the junction table. Do not draw a single `*..*` line. The junction shows up as its own entity with foreign keys back to both sides.
- **Referential integrity.** Annotate `ON DELETE` / `ON UPDATE` behavior only when it carries meaning (cascade / restrict). Never invent it.
- **Storage-only.** This diagram captures **storage** structure. No class methods, no Service / Controller objects, no business orchestration. Field-level domain logic stays in `lark-uml:class`.

## Forbidden mixings

- Class methods, Service / Controller objects — those belong in `lark-uml:class`.
- Business process steps — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- Actors and system boundaries — those belong in `lark-uml:usecase`.
- Network devices — those belong in `lark-uml:network`.
- Deployment layering — that belongs in `lark-uml:architecture`.

## Native node composition

Build the ER diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **Entity** — native rectangle with a header row (table name + optional Chinese display name) and one stacked row per field. New entities are produced by duplicating an existing entity and editing the header and rows.
- **Field row** — native sub-rectangle / table-row element inside the entity, with columns `字段名 | 类型 | 约束`. Constraint markers come from the fixed set `PK`, `FK`, `NN`, `UQ`, `IDX`.
- **PK / FK rows** — same field-row primitive, marked with `PK` or `FK`. Foreign-key rows are the actual endpoints relationship lines bind to (not just the entity header).
- **Relationship connector** — native `type: "connector"`, with `connector.from` bound to the FK row id on the child side and `connector.to` bound to the PK row id on the parent side. Cardinality is shown via the connector's end style (crow-foot or `1..1` / `1..*` / `0..*` / `0..1` labels) — keep one notation per board.
- **Junction entity** — for many-to-many, materialize a native entity in its own right with two FK rows; never substitute a single `*..*` line.

When you add a field, clone an existing field row from the same or an adjacent entity to preserve column widths and text style.
