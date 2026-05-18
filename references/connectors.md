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

## Pre-write validation

Before `+update --input_format raw`, validate the edited raw structure:

- Every business relationship is a native `connector`.
- Every business `connector.from` and `connector.to` is a string node id.
- Every referenced id exists in the final `nodes` set.
- No business relationship endpoint is a coordinate object.
- No business relationship was converted into a decorative line, path, SVG, image, Mermaid output, PlantUML output, or other unverifiable shape.
- No connector only "looks connected" by sitting against a node edge while remaining structurally unbound.

If any check fails, do not write the board. Fix the raw structure first or stop and ask for the missing binding decision.

## Post-write verification

After writing raw:

1. Read the board again with `+query --output_as raw`.
2. Re-check the key business relationships as `node id -> node id`.
3. Query an image preview only after the structural checks pass.
4. Apply the drag semantics check: if moving either endpoint node would not theoretically keep the connector attached, the structure is invalid and must be corrected.

