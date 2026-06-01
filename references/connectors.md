# Native connector rules

Every `lark-uml:*` skill inherits these rules. They are correctness gates, not styling preferences.

## Raw-first principle

- For flowcharts, swimlanes, architecture diagrams, organization-style structures, network topologies, use case diagrams, sequence diagrams, class diagrams, ER diagrams, and any task where nodes must remain editable and connectors must follow moved nodes, use this route only: `raw query -> raw modify -> raw update`.
- Business relationships must be native whiteboard `connector` nodes. Do not degrade them into geometric lines, paths, SVG, images, Mermaid-rendered strokes, PlantUML-rendered strokes, or any other representation whose endpoints cannot be verified as node-bound.
- Visual contact is not success. A connector that merely touches a node edge but is not bound to that node is a failed relationship.

## Business relationship connector

A line is a business relationship connector when it expresses a relationship between diagram objects, including:

- process steps, decisions, handoffs, returns, retries, and exception flows
- roles, actors, participants, use cases, lifelines, and messages
- modules, services, storage, queues, gateways, dependencies, calls, and data flows
- entities, tables, fields, primary/foreign-key relationships, cardinality, and ownership
- devices, zones, hosts, links, peerings, and network reachability

Every business relationship connector must satisfy all of these conditions:

- It is a native node with `type: "connector"`.
- `connector.from` is a string id of an existing node in `nodes`.
- `connector.to` is a string id of an existing node in `nodes`.
- The referenced ids still exist after all additions, deletions, merges, and renames.
- The connector remains in the top-level `nodes` collection, not hidden inside another node's children or converted into a decoration.
- Its anchors and direction make semantic sense for the relative positions of the source and target nodes.

Forbidden for business relationship connectors:

- `{ x, y }` coordinate endpoints.
- Dangling ids, temporary ids, ids of nodes scheduled for deletion, or ids copied from a stale raw snapshot.
- Replacing the relationship with a line, path, polyline, SVG node, bitmap, code-rendered diagram, or visually aligned geometry.
- Using waypoints as a substitute for endpoint binding.

## Coordinate endpoints

Coordinate endpoints are allowed only for non-business support marks:

- coordinate axes
- measurement marks
- annotation arrows that point to empty space
- decorative dividers
- helper lines with no attached object

These marks are not primary diagram relationships and must not replace a real node-to-node connector. If a line says "this object relates to that object", it is a business relationship connector and must bind by node id.

## Waypoints

- `waypoints` control the middle of a route only.
- `waypoints` do not attach a connector to a node.
- A business relationship connector with `waypoints` must still use string node ids for both `connector.from` and `connector.to`.
- Prefer the whiteboard's automatic routing first. Use manual waypoints only when routing must be constrained after endpoint binding is already correct.

## ER cardinality via arrow styles

For `lark-uml:er`, cardinality is expressed through the native `arrow_style` on each connector end — the arrow shape **is** the notation. Text labels on the line are not a substitute.

| Cardinality | `arrow_style` |
|---|---|
| `1` (one) | `"zero_or_single_arrow"` |
| `*` (many) | `"zero_or_multi_arrow"` |

- `connector.shape` must be `"straight"` for direct FK relationships; use `"polyline"` only when routing around other entities is unavoidable.
- `turning_points` must be `[]` for straight connectors.
- Every FK relationship gets one connector. The FK (child) row id is `connector.from`; the PK (parent) row id is `connector.to`.
- Many-to-many is materialized as a junction `table_uml` with two FK connectors — never a single connector with `*` on both ends.

## Per-skill arrow_style enforcement

Every `lark-uml:*` skill MUST set `arrow_style` correctly on every business connector end. The native whiteboard has **exactly two** `arrow_style` values, and the field lives at a **fixed path** inside the connector node:

```
connector.start.arrow_style    ← arrow on the source end (sibling of attached_object)
connector.end.arrow_style      ← arrow on the target end (sibling of attached_object)
```

`arrow_style` does **not** appear on `connector.start_object` or `connector.end_object` — those carry only the target node `id`, `position`, and `snap_to`.

| `arrow_style` | Shape | Meaning |
|---|---|---|
| `"zero_or_single_arrow"` | Single arrowhead (`→`) | One-side, flow direction, call, dependency, message |
| `"zero_or_multi_arrow"` | Multi-prong / crow's foot (`⇒`) | Many-side of ER cardinality only |

For an end with **no arrow**, omit the `arrow_style` field entirely — do NOT set it to `"none"` or any other made-up value.

Per-skill requirements:

| Skill | Connector type | `arrow_style` |
|---|---|---|
| `lark-uml:er` | FK relationship | FK child side: `"zero_or_multi_arrow"` (many) or `"zero_or_single_arrow"` (one); PK parent side: `"zero_or_single_arrow"` |
| `lark-uml:class` | Inheritance / Realization | Child→Parent end: `"zero_or_single_arrow"` (UML triangle decoration supplements, not replaces) |
| `lark-uml:class` | Association / Dependency | Direction end: `"zero_or_single_arrow"`; undirected end: omit `arrow_style` |
| `lark-uml:class` | Aggregation / Composition | Whole-side: omit `arrow_style` (diamond decoration on this end); part-side: omit `arrow_style` |
| `lark-uml:usecase` | Actor↔UseCase association | Both ends: omit `arrow_style` or direction end: `"zero_or_single_arrow"` |
| `lark-uml:usecase` | Include / Extend | Arrow end: `"zero_or_single_arrow"`; other end: omit `arrow_style` |
| `lark-uml:usecase` | Generalization | Specific→General end: `"zero_or_single_arrow"` (UML triangle decoration supplements) |
| `lark-uml:flowchart` | Flow edge | Target end: `"zero_or_single_arrow"`; source end: omit `arrow_style` |
| `lark-uml:swimlane` | Cross-lane handoff, branch | Target end: `"zero_or_single_arrow"`; source end: omit `arrow_style` |
| `lark-uml:sequence` | Forward message (solid) | Target end: `"zero_or_single_arrow"`; source end: omit `arrow_style` |
| `lark-uml:sequence` | Return message (dashed) | Target end: `"zero_or_single_arrow"` (open style); source end: omit `arrow_style` |
| `lark-uml:architecture` | Call / Data flow | Target end: `"zero_or_single_arrow"`; source end: omit `arrow_style` |
| `lark-uml:network` | Symmetric link | Both ends: omit `arrow_style` |
| `lark-uml:network` | Asymmetric / NAT / directed | Direction end: `"zero_or_single_arrow"`; other end: omit `arrow_style` |

These are correctness gates. A connector whose `arrow_style` does not match the table above is invalid — fix it before writing.

## Pre-write validation

Before `+update --input_format raw`, validate the edited raw structure:

- Every business relationship is a native `connector`.
- Every business `connector.start.attached_object.id` and `connector.end.attached_object.id` is a string node id.
- Every referenced id exists in the final `nodes` set.
- No business relationship endpoint is a coordinate object.
- Every business connector end with an arrow has `connector.start.arrow_style` / `connector.end.arrow_style` set to `"zero_or_single_arrow"` or `"zero_or_multi_arrow"` per the per-skill table above.
- No `connector.start_object` or `connector.end_object` carries `arrow_style` — it belongs on `connector.start` / `connector.end` only.
- No `arrow_style` is set to `"none"` or any made-up value — omit the field when no arrow is needed.
- No business relationship was converted into a decorative line, path, SVG, image, Mermaid output, PlantUML output, or other unverifiable shape.
- No connector only "looks connected" by sitting against a node edge while remaining structurally unbound.

If any check fails, do not write the board. Fix the raw structure first or stop and ask for the missing binding decision.

## Post-write verification

After writing raw:

1. Read the board again with `+query --output_as raw`.
2. Re-check the key business relationships as `node id -> node id`.
3. Query an image preview only after the structural checks pass.
4. Apply the drag semantics check: if moving either endpoint node would not theoretically keep the connector attached, the structure is invalid and must be corrected.

