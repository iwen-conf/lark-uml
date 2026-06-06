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

**Execution route:** raw-first, native-only. Read the board as raw, edit native entity / table shapes and native connectors in place, then write raw back. Foreign-key, cardinality, and table / field relationships are business relationships expressed as native `connector` nodes with explicit `arrow_style` — cardinality is rendered by the arrow shape itself. **No PlantUML / Mermaid / SVG anywhere in the loop, not even as a private sketch.** Any shared reference defaulting to PlantUML is overridden for this skill.

**Default mode is modify-in-place.** Duplicate and adapt the template's existing entity tables, field rows, and relationship connectors. Only redraw when the user explicitly asks, or the diagram is the wrong type entirely.

## Diagram-specific rules

- **Source of truth first.** For code-backed projects, derive the physical schema from the actual migrations / DDL / schema files before drawing. Include later `ALTER TABLE ADD COLUMN` migrations, not only initial `CREATE TABLE` files. Do not invent tables, omit tables, keep obsolete tables, or copy class-diagram objects into ER.
- **Entities are tables.** Every entity is a header-rectangle: the table name in the header row, fields stacked underneath. Same-style headers across all entities; same-style rows across all entities; identical column widths within one entity.
- **Field columns.** Each field row shows, in order: field name, type, constraint markers. Constraint markers use the standard set: `PK` (primary key), `FK` (foreign key), `NN` (not null), `UQ` (unique), `IDX` (indexed). One marker per role, no ad-hoc abbreviations.
- **Primary and foreign keys.** Always mark them. A foreign key row must visually point at the referenced primary key — bind the relationship line endpoints to the actual `PK` / `FK` rows, not just to the entity header.
- **Cardinality.** Every relationship line carries cardinality on both ends via the native `arrow_style` on each connector end. `"zero_or_single_arrow"` means "one", `"zero_or_multi_arrow"` means "many". The arrow shape **is** the notation. If the user or target document explicitly needs visible text such as `0..* - 1`, put it in the connector's own `connector.captions.data`, never as separate `text_shape` nodes.
- **One line per table pair.** A table may connect to many different tables, but the same unordered pair of tables must have at most one visible relationship connector. If several FK columns point from one table to the same parent table, keep all FK field markers in the table rows but merge the visual relationship into one connector between the two tables.
- **Straight-line default.** ER relationship connectors should be `shape: "straight"` with `turning_points: []`. Use polyline routing only when the user explicitly allows it or when a straight connector would make the diagram materially unreadable.
- **Layout before write.** Choose table positions before writing, not after user feedback. Put high-degree hub tables near the center, place directly related tables around them, keep parent-child chains adjacent, and minimize line crossings / line-through-table cases while keeping the diagram compact. Avoid long diagonal bundles through dense table groups.
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

Pick every native `type` from the matrix in [`../../references/native-types.md`](../../references/native-types.md) (see the ER row). Every entity must be a single `table_uml` node (header row + field rows inside `table.cells`), **never** stacked rectangles. Relationship `connector`s bind to PK / FK row ids — not to the entity header — and many-to-many is materialized as its own junction `table_uml`, never as a `*..*` line.

Build the ER diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **Entity** — native rectangle with a header row (table name + optional Chinese display name) and one stacked row per field. New entities are produced by duplicating an existing entity and editing the header and rows.
- **Field row** — native sub-rectangle / table-row element inside the entity, with columns `字段名 | 类型 | 约束`. Constraint markers come from the fixed set `PK`, `FK`, `NN`, `UQ`, `IDX`.
- **PK / FK rows** — same field-row primitive, marked with `PK` or `FK`. Foreign-key rows are the actual endpoints relationship lines bind to (not just the entity header).
- **Relationship connector** — native `type: "connector"`, `shape: "straight"`, `turning_points: []`. `connector.from` binds to the FK row id on the child side; `connector.to` binds to the PK row id on the parent side. Cardinality is expressed **exclusively through native arrow styles** on the connector ends, not through text labels on the line. The two arrow styles map directly to crow's-foot notation:

  | Cardinality | `arrow_style` | Crow's-foot |
  |---|---|---|
  | `1` (one) | `"zero_or_single_arrow"` | Single line (`\|`) |
  | `*` (many) | `"zero_or_multi_arrow"` | Crow's foot (`}\|`) |

  Relationship ends combine independently. Examples:
  - `1..1` — both ends `zero_or_single_arrow`
  - `1..*` — FK (child) side `zero_or_multi_arrow`, PK (parent) side `zero_or_single_arrow`
  - `0..*` — FK side `zero_or_multi_arrow`, PK side `zero_or_single_arrow`

  The native arrow shape **is** the cardinality notation. Never substitute text labels (`1..*`, `0..1`) for `arrow_style`. When the user asks for visible cardinality text, add one concise connector caption (for example `0..* - 1`) to that same connector and keep `arrow_style` consistent across all connectors on the same board. Do not create independent label nodes.
- **Junction entity** — for many-to-many, materialize a native entity in its own right with two FK rows; never substitute a single `*..*` line.

When you add a field, clone an existing field row from the same or an adjacent entity to preserve column widths and text style.

## One-shot validation checklist

Before writing the board, and again after `+update` by re-querying raw, verify:

- Table count matches the source schema exactly; missing tables = 0; extra tables = 0.
- Field rows match the source schema exactly for field name, type, and `PK` / `FK` / `NN` / `UQ` markers; field diffs = 0.
- No obsolete diagram-only tables remain.
- Every connector is native, bound to existing table / row ids, has valid `arrow_style`, and uses `shape: "straight"` unless a deliberate exception was requested.
- Each unordered table pair has at most one connector; duplicate table-pair connectors = 0.
- Connector captions, when used, live on `connector.captions.data`; separate cardinality text nodes = 0.
- Preview image is checked for layout: related tables are near each other, high-degree tables are central, no avoidable line bundle crosses the core of the diagram.
