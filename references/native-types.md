# Native Type Cheat-Sheet (all `lark-uml:*` skills)

Pick the native `type` by **intent**, not by appearance. Whenever a dedicated shape exists, use it — generic rectangles cannot fake the operator pentagon (`combined_fragment`), the lifeline tail (`life_line`), the actor stick figure (`actor`), the cylinder for storage (`cylinder`), or the class / table grid (`table_uml`). If your emitted `type` is not in the matrix below, you picked the wrong shape.

## Global type matrix (the only types you should ever emit)

| Native `type` | Use it for | Key fields | Never use it for |
|---|---|---|---|
| `composite_shape` | All "named box" needs: process step, service / module, gateway, host, VM, action, system boundary, participant header, use case oval, start / end, decision diamond. Sub-shape (`round_rect`, `round_rect2`, `actor`, `ellipse`, `diamond`, `stadium`, `trapezoid`…) is picked via `composite_shape.type` and **must be cloned from the template's existing element**. | `composite_shape.type`, `text.text`, `x/y/width/height` | Free-floating text (use `text_shape`), database storage (use `cylinder`), tabular class / entity (use `table_uml`) |
| `round_rect` | Legacy plain rounded rectangle nodes already present in the board. Prefer `composite_shape{round_rect}` for anything new so style stays template-consistent. | `text.text`, `style.*` | Anything semantically richer than "a labeled box" |
| `actor` | Standalone human-actor stick figure when the template doesn't pack it into `composite_shape{actor}`. | `text.text` = role name | System / service / module heads (use `composite_shape`) |
| `cylinder` | Database / object store / cache / queue storage in architecture & network diagrams. | `text.text` = store name, `style.*` | Generic rectangles labeled "DB" (lose the cylinder silhouette) |
| `life_line` | Sequence-diagram lifeline tail under a participant head, **and** the activation bar (same type, narrow width, short `lifeline.size`). | `lifeline.size` (vertical length), `lifeline.type` = `"round_rect"` (object) \| `"actor_lifeline"` (human); `x` = center `x` of header | A simple vertical line (use a `connector`-shaped life_line nonetheless, this type is mandatory inside sequence diagrams) |
| `activation` | Some templates expose activation bars as a first-class node instead of a narrow `life_line`. Clone whichever variant the template already uses. | `width` (narrow), `height` (matches busy span), centered on the parent lifeline | Decoration outside a sequence diagram |
| `combined_fragment` | Sequence-diagram structured regions: `alt` / `opt` / `loop` / `par`. Multi-region operators MUST use `meta.row_num` ≥ 2 + matching `meta.row_sizes` inside one fragment (never multiple stacked frames). | `text.text` = operator + guard; `meta.row_num`, `meta.row_sizes`; per-row header carries the `[guard]` label | Plain rectangle "frames", grouped rectangles, SVG polygons |
| `table_uml` | Class boxes and ER entities: a true table grid with `table.title` + per-row `cells`. New rows / fields are added inside `table.cells`, not as separate rectangles. | `table.title`, `table.meta.row_num` / `row_sizes`, `table.cells[]` with `row_index` / `col_index` / `text` | Plain stacked rectangles trying to look like a class / entity |
| `connector` | Every relationship line: flowchart edge, sequence message, class / ER / use-case relationship, network link, swimlane handoff, architecture call. | `connector.start.attached_object.id` + `connector.end.attached_object.id` (BOTH must bind to real node ids), `connector.shape`, end `arrow_style`, `connector.captions.data[].text` (label) | Coordinate-only endpoints, diagonals that bypass a lifeline, freeform polylines disguised as edges |
| `note_shape` | Side annotation pinned next to flow content (sequence note, flowchart comment, architecture caveat). | `text.text`, `x/y` near the anchor | Carrying message labels (those belong in `connector.captions`) |
| `text_shape` | Free-floating labels only: diagram title, legend, watermark. | `text.text`, `x/y` | Participant heads, class names, entity headers, message labels, fragment guards, note bodies |
| `section` | Background panel grouping a whole diagram or sub-diagram region (e.g. "管理员用例图"). | `section.title`, contained child ids | Replacing `combined_fragment` for sequence regions; replacing `composite_shape` boundaries for use case / architecture |

## Per-skill quick selection

Each skill should only emit types from its row below (plus `connector`, `note_shape`, `text_shape`, `section`, which are universal). If you reach for any other type, stop and re-check the intent.

| Skill | Primary node types | Common pitfalls |
|---|---|---|
| `lark-uml:flowchart` | `composite_shape` (start / end, process step, decision diamond, merge), `connector` | Adding `cylinder` / `actor` / `table_uml` — those belong in architecture / sequence / class. Free-text branch labels via `text_shape` instead of `connector.captions`. |
| `lark-uml:swimlane` | `section` (lane), `composite_shape` (action, decision, start / end), `connector` (cross-lane handoff) | Drawing lanes as plain rectangles instead of `section`. Cross-lane arrows with coordinate-only endpoints. |
| `lark-uml:sequence` | `composite_shape` / native participant heads (exact top order for standard backend flows: `Browser`, `:{Business}Window` / `:{Business}Page` / `:{Business}View`, `:{Business}Controller`, `:{Business}Service`, `:{Business}Dao`, `DB`), `life_line` (lifeline tail), `life_line` / `activation` (activation bar), `combined_fragment` (alt / opt / loop / par), `connector` (message / self-call), `note_shape` (side note) | Faking a fragment with a plain rectangle; using `connector` as a lifeline; coordinate-only message endpoints; diagonal messages; message labels without `N: description`; stacking multiple frames instead of multi-row `combined_fragment`; collapsing `Browser` and the business view into one participant; using framework-specific names like `XxxMapper`, `Repository`, `Model`, or ORM client names for Dao (use `Dao` suffix); embedding SVG / Mermaid / PlantUML / Graphviz / draw.io / images instead of editable native nodes. |
| `lark-uml:usecase` | `composite_shape` (system boundary rectangle, use case ellipse), `actor` (or `composite_shape{actor}`), `connector` (association / include / extend / generalization) | Putting actors **inside** the boundary; substituting `section` for the system boundary; using `text_shape` for use case names. |
| `lark-uml:class` | `table_uml` (class / interface / abstract box with name / attribute / method rows), `connector` (inheritance / realization / association / aggregation / composition / dependency) | Building a class out of three stacked rectangles instead of `table_uml`. Forgetting hollow / filled diamond / triangle end styles. |
| `lark-uml:er` | `table_uml` (entity with header row + field rows), `connector` (relationship, bound to PK / FK row ids) | Drawing entities as plain rectangles; binding relationship connectors to the entity header instead of the FK row; using a single `*..*` line in place of a junction `table_uml`. |
| `lark-uml:architecture` | `composite_shape` (service / module / gateway / external system), `cylinder` (storage), `section` (boundary), `connector` (call / data flow) | Labeling a rectangle `DB` instead of using `cylinder`; nesting more than two `section` levels; mixing protocol labels with business labels on the same connector. |
| `lark-uml:network` | `composite_shape` (router / firewall / switch / LB / host / VM / pod / service endpoint, plus internet cloud shape), `section` (zone / subnet / VPC), `connector` (link) | Annotating links with business semantics (`下单`) instead of network-layer facts (`TCP/443`); flattening zones into bare rectangles. |

## Decision algorithm (apply before emitting any node)

1. **What does this node represent in the real world?** (a person? a class? a storage? a region?)
2. **Which row of the global type matrix matches that intent?** Pick that `type`.
3. **Does the skill's row above list this type?** If not, you are about to draw something that belongs in a different skill — stop.
4. **Are the key fields all present** (e.g. `connector.start.attached_object.id`, `combined_fragment.meta.row_sizes`, `table_uml.table.cells`)? If not, the renderer will silently degrade the shape.
5. **Did you clone the closest existing element of the same kind from the template?** If not, style will drift. Clone first, then overwrite content.
6. **Final sweep before writing back:** every node's `type` must appear in the global matrix. Any rogue `type` is a bug — re-map the intent.
