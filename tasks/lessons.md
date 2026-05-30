# Lessons

## PlantUML is the practical primary route (2026-05-30)

- **PlantUML via `whiteboard +update --input_format plantuml --overwrite` is the recommended path** for first-time creation and full-board rewrites. Raw native JSON editing is fragile, verbose, and error-prone for multi-participant sequence diagrams. Reserve raw native for precise in-place tweaks on boards that already have native `combined_fragment` nodes.
- A single `docs +update --command overwrite` creates the document structure (callout, table, headings, blank whiteboards), then **separate** `whiteboard +update` calls fill each board. Never put multiple `@startuml`/`@enduml` blocks in one PlantUML input — the Feishu parser rejects it.
- The Feishu PlantUML parser rejects `newpage` between diagrams. One board = one `@startuml` block.
- For Mermaid (when forced): the Feishu parser requires `flowchart TD` not `graph TD`. But PlantUML via `whiteboard +update` is more reliable than `<whiteboard type="mermaid">` inline in docs.

## Document co-design pattern

When delivering a set of related diagrams inside a Feishu doc/wiki:

```
callout ✅ + summary table (序号/业务/角色/图示组成)
---
# 1. {业务名称}
  说明段落
  ## 活动图 (whiteboard block)
  ## 时序图 (whiteboard block)
---
# N. ...
```

- Activity diagram **always before** sequence diagram (活动图在前).
- Each section covers **one single focused action** — don't merge multiple business flows.
- Use `<whiteboard type="blank"></whiteboard>` in `docs +update` XML to create blank whiteboard placeholders, then fill each via `whiteboard +update --input_format plantuml --overwrite`.

## Sequence diagram conventions (battle-tested)

- **Actor**: `Browser`, never 客户/用户/User. For web systems the user entry point is the browser.
- **No background**: never `skinparam backgroundColor`.
- **Lifelines mandatory**: every participant gets `activate`/`deactivate`. Browser activates first, deactivates last.
- **Layer order**: `Browser` → `:Page` → `:Controller` → `:Service` → `:Mapper` → `DB`.
- **Real code names**: use actual class suffixes from source (`Mapper`, `Service`, `Controller`). Don't force `Dao` when the code says `Mapper`.
- **Message numbering**: `"N: description"` with continuous top-to-bottom sequence.
- **Single action**: one diagram = one core business action. No page-load or login branches.
- **Assume authenticated**: no `alt 未登录` blocks.
- **Simple participants**: for 4-5 layer diagrams, use `participant ":CartService" as CS` aliases to keep labels short.

## Activity diagram conventions (battle-tested)

- Use `if`/`else`/`endif` for decisions, with labels `(是)`/`(否)`.
- No `elseif` — nest instead.
- `start` → steps → decisions → `stop`.
- Use `partition "用户端" { ... }` for activity diagrams that span roles (e.g., chat flow).
- Same layer names as sequence diagrams for consistency.

## Things that silently fail

- Multiple `@startuml` blocks in one input → parser error.
- `newpage` between PlantUML diagrams → parser error.
- `graph TD` in Mermaid whiteboards → use `flowchart TD`.
- `<whiteboard type="mermaid">` inline in doc XML → parse failures more frequent than `whiteboard +update --input_format plantuml`.
