---
name: lark-uml:sequence
description: 飞书画板时序图：执行型 Skill。当用户要在飞书画板上绘制或修改时序图 / 序列图（参与者、生命线、消息调用、返回、激活区）时使用。严格要求生命线垂直、消息水平、按时间从上到下单调推进。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 PlantUML / SVG 代码。
---

# `lark-uml:sequence`

Specialist skill for **sequence diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

**Hard contract:** everything on the canvas is a Feishu / Lark native whiteboard node or native connector. A valid result is an editable whiteboard made from raw native nodes, not a rendered image, embedded SVG, Mermaid / PlantUML conversion, screenshot, or freeform drawing that merely looks like a sequence diagram.

Sequence diagrams are the most layout-sensitive diagram type in this plugin. The rules below are not stylistic — they are correctness gates.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the sequence diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native participants, lifelines, activations, combined fragments, notes, and native message connectors in place, then write raw back. Sequence messages are business relationships, so endpoints must bind to participant, lifeline, or activation node ids and remain horizontally aligned. **No PlantUML / Mermaid / SVG / Graphviz / draw.io / raster image / screenshot anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing participants, lifelines, activations, message connectors, and combined fragment frames. Only redraw when the user explicitly asks, or the diagram is the wrong type.

## Diagram-specific rules

### Top participant contract

These are hard validity rules for the participant headers at the top of every sequence diagram.

- **Required top order.** The participant columns must appear left-to-right as exactly this layered chain unless the user explicitly supplies a different architecture and the real source code proves it: `Browser` → `:{Business}Window` / `:{Business}Page` / `:{Business}View` → `:{Business}Controller` → `:{Business}Service` → `:{Business}Dao` → `DB`.
- **Browser column.** The first column is the user entry point. For web systems label it `Browser`; for non-web systems use the real user/client entry point only when the source proves it. This column represents the user action source, not the business view. It uses a native participant head and lifeline; do not use an imported actor icon, `text_shape`, screenshot, or freeform drawing.
- **Business view column.** The second column is the concrete UI view/window/page that receives the user action, such as `:SearchBookWindow`, `:QAPage`, or `:CheckoutView`. Derive this name from the actual frontend page/component/window when available. This column is mandatory for standard flows; do not collapse it into `Browser`.
- **Controller column.** The third column is the real controller / route handler reached by the business view. Its label is `:{Business}Controller`; derive `{Business}` from the real controller class / route module, not from a guessed feature title.
- **Service column.** The fourth column is the real service method called by that controller. Its label is `:{Business}Service`; use the service that actually sits on the call path.
- **Dao column.** The fifth column is the persistence access layer. Its label is always `:{Business}Dao`, even when the code uses `Mapper`, `Repository`, `Prisma`, `Drizzle`, `Model`, or framework-specific storage names. The diagram may show the real method name in messages, but the participant header suffix is always `Dao`.
- **DB column.** The sixth column is `DB`, representing the database operation executed by the Dao layer. Do not replace it with a framework object, ORM model, table-shaped fake, screenshot, or generic storage drawing.
- **No skipped layers.** Do not omit Browser, business view, Controller, Service, Dao, or DB just because the implementation is thin. If a project truly collapses two layers in code, keep the standard visible columns and map the real function call to the nearest layer, adding only source-derived method names in message labels. Ask the user before writing if the code has no backend call path at all.
- **One business chain per diagram.** A standard diagram models one Browser → View/Page/Window → Controller → Service → Dao → DB call path. Do not add extra service columns, helper utilities, schedulers, external APIs, or multiple databases unless the user explicitly asks and the actual source call path requires them.
- **Top headers must be native and aligned.** All six headers must be cloned or created as native participant heads with identical top `y`, uniform sizes, and uniform spacing. A text-only label floating above a lifeline is invalid.

### Source-derived call-chain rules

- **Code evidence is mandatory.** Before writing the board, inspect the target project source and identify the concrete Browser/user action, business view/page/window handler, Controller handler, Service method, Dao persistence method, SQL / ORM operation, parameter types, and return types. The visible sequence must follow that real call path.
- **No invented identifiers.** Do not invent class names, method names, DTOs, return types, table names, or column names. If the source is unavailable or ambiguous, ask the user for the identifiers before updating the board.
- **Method labels follow source names.** Controller / Service / Dao message labels must use the actual invoked function names and parameter types from the code. You may normalize the participant header to the standard `:{Business}{Layer}` format, but message labels must remain tied to the real call.
- **SQL / DB labels follow real persistence.** The DB message must reflect the actual SQL, ORM query intent, table, collection, or model touched by the Dao method. Prefer a concise real SQL-style label such as `INSERT INTO ...`, `SELECT ... FROM ...`, `UPDATE ... SET ...`, or `DELETE FROM ...` using real table / column names when available.
- **If the code path is not layered.** For frontends calling APIs, serverless routes, RPC handlers, or ORMs directly, map the real functions into the six required columns without changing the top order: user/client = Browser, UI component/page = Window/Page/View, API route / handler = Controller, domain operation = Service, persistence adapter / repository / ORM call = Dao, physical table / collection = DB. Do not expose framework plumbing as extra participants.

### Message format & layering convention

These rules define the standard visual vocabulary for sequence diagram messages. They apply to every board produced or modified by this skill.

- **Participant naming.** The first column is `Browser` for web user input. The second column is the business view/window/page (e.g. `:SearchBookWindow`, `:QAPage`, `:CheckoutView`). System-layer participants use the `:{Business}{Layer}` format with a colon prefix and the business domain abbreviation followed by the layer name (e.g. `:BookController`, `:BookService`, `:BookDao`). The storage layer is simply `DB`. **Never** use framework-specific class names like `XxxRecordMapper` or `XxxMapper` for the Dao layer — the layer suffix is always `Dao`.
- **Mandatory layer order (left to right).** `Browser` → `:{Business}Window` / `:{Business}Page` / `:{Business}View` → `:{Business}Controller` → `:{Business}Service` → `:{Business}Dao` → `DB`. This mirrors the standard layered architecture: user/client → view → controller → service → dao → database. The `{Business}` prefix is the short business domain name (e.g. `Book`, `Emotion`, `Match`, `Chat`, `Ai`, `Social`). Derive both the business prefix and the method names from the system's actual source code.
- **Message numbering.** Every message label starts with an Arabic numeral followed by a colon and a space: `"N: description"`. Numbers increase top-to-bottom across the entire diagram in one continuous sequence covering both forward (solid) and return (dashed) messages. Letter prefixes such as `B:`, missing colons, missing spaces after the colon, duplicated numbers, and unnumbered message labels are invalid.
- **Message description required.** Every connector label must contain a meaningful operation / data description after the number. Examples: `"1: input Book name or ISBN"`, `"2: searchBook(name, ISBN)"`, `"5: SELECT * FROM book WHERE name = ? OR isbn = ?"`, `"7: books"`. A number-only label or blank connector caption is invalid.
- **Forward message structure.** The forward call chain consists of exactly five messages progressing left-to-right across adjacent lifelines after the user entry:
  1. Browser → business view: user input / action data (e.g. `"1: input Book name or ISBN"`)
  2. business view → Controller: Controller method with parameters (e.g. `"2: searchBook(name, ISBN)"`)
  3. Controller → Service: Service method with parameters (e.g. `"3: searchBook(name, ISBN)"`)
  4. Service → Dao: Dao method with parameters (e.g. `"4: queryBook(name, ISBN)"`)
  5. Dao → DB: SQL / persistence operation (e.g. `"5: SELECT * FROM book WHERE name = ? OR isbn = ?"`)
  All method messages (2–4) must include their parameter list. Bare method names without parameters are not allowed.
- **Return message structure.** Return messages follow the reverse path right-to-left, each labeled with the return type:
  6. DB → Dao: DB result data / affected row count (e.g. `"6: book records"`)
  7. Dao → Service: domain result / collection (e.g. `"7: books"`)
  8. Service → Controller: service return type / result (e.g. `"8: books"`)
  9. Controller → business view: wrapped response / view model (e.g. `"9: books"`)
  10. business view → Browser: displayed result / user-visible response (e.g. `"10: show book info"`)
- **Activation bar discipline.** Every participant column **except the first Browser column** gets an activation bar. The activation bar starts at the first incoming message to that participant and ends at the last return message from that participant. The Browser column normally carries only its lifeline and no activation bar.
- **One round trip only.** The diagram shows exactly one forward call chain (messages 1–5) and one return chain (messages 6–10). Do not add extra acknowledgment messages, intermediate processing steps, or additional "render / display" messages beyond the single return from the business view to Browser. When branching logic is required, use a `combined_fragment` (`alt` / `opt`) around the relevant messages — but each branch still follows the same numbered forward-then-return structure internally.
- **Derive names from real code.** Participant class names, method signatures, parameter types, return types, and SQL table/column names must be extracted from the actual project source code. Do not invent placeholder names. If the code is unavailable, ask the user for the correct identifiers before writing.

### Canonical reference example

Use this book-search sequence as the baseline shape when interpreting ambiguous requests. Replace names and labels with identifiers from the real project, but keep the same layer split, native construction, message numbering, and return path.

- **Top headers:** `Browser` → `:SearchBookWindow` → `:BookController` → `:BookService` → `:BookDao` → `DB`
- **Forward messages:**
  1. `1: input Book name or ISBN`
  2. `2: searchBook(name, ISBN)`
  3. `3: searchBook(name, ISBN)`
  4. `4: queryBook(name, ISBN)`
  5. `5: SELECT * FROM book WHERE name = ? OR isbn = ?`
- **Return messages:**
  6. `6: book records`
  7. `7: books`
  8. `8: books`
  9. `9: books`
  10. `10: show book info`

### Native-only whiteboard rules

- **Only native raw nodes.** Every visible element must be represented by Feishu / Lark whiteboard raw JSON using approved native node types. Do not upload or embed SVG, PNG, HTML canvas output, screenshots, Mermaid renderings, PlantUML renderings, Graphviz renderings, draw.io exports, or pasted images.
- **Only native connectors for messages.** Every message arrow must be a `connector` with both ends bound to native participant, lifeline, or activation node ids. A coordinate-only line, freeform polyline, imported arrow, or manually drawn stroke is invalid even if it looks correct.
- **Only native lifelines and activations.** Lifelines must use native `life_line` nodes. Activation bars must use native `life_line` or native `activation` nodes, depending on the template. Do not use generic rectangles or connectors to imitate them.
- **Only native combined fragments.** Branching / optional / loop / parallel regions must use native `combined_fragment` nodes. Plain rectangles, grouped rectangles, SVG polygons, section panels, or images are invalid substitutes.
- **Clone native template elements.** Prefer duplicating the board's existing native participant, lifeline, activation, connector, note, and combined-fragment nodes, then overwrite geometry and labels. If the needed native type is absent, create that native type directly in raw JSON; do not fall back to a visual approximation.
- **Stop before write on non-native content.** If the edited region contains a non-native substitute for a required sequence element, fix it before `+update`. If it cannot be fixed with available raw node types, stop and report the blocker instead of writing a degraded board.

### Layout & geometry rules

- **Participant head alignment.** Every participant header (`actor` / `participant` / `object` rectangle) sits on the **same top horizontal baseline**. Identical `y`. Left-to-right ordering. Uniform horizontal spacing. Uniform size. No drifting heads.
- **Lifeline discipline.** Every participant has a vertical lifeline directly below its header. Lifelines are **parallel, strictly vertical, identical length, bottom-`y` aligned**. Each lifeline's `x` is **exactly** the center `x` of its participant header.
- **Messages are horizontal.** Every message connector endpoint is bound to a lifeline (or its participant head). The two endpoints share the **same `y`** (within rendering tolerance). Messages stack top-to-bottom; their `y` coordinates increase monotonically. **No diagonal lines. No freeform polylines. No "point a to point b" geometry that bypasses lifelines.**
- **Self-call.** A participant calling itself draws as a small right-angle loop on the right side of its own lifeline: horizontal out → short drop → horizontal back. Never a long diagonal. Never routed through another lifeline.
- **Activation bar.** The activation rectangle is a narrow strip **centered on the lifeline**, uniform width. It must not overlap the neighbor lifeline. It does not replace the lifeline.
- **Crossings.** A message that visually crosses an intermediate lifeline does **not** change its `y` to dodge — it stays a straight horizontal arrow. Routing tools may render a small jump, but the source `y` is identical at both endpoints.
- **Combined fragments.** Use the whiteboard's **native combined-fragment shape** (`type: "combined_fragment"`, 飞书画板"组合片段"图形) when sequence logic needs conditional, optional, looping, or parallel behavior. **Do not** substitute with plain rectangles, grouped rectangles, or freeform frames — they render without the operator pentagon and the row separators, and break the diagram's semantics. The operator label sits in the top-left pentagon corner, branch guards are written in square brackets inside each region's header, and all messages inside each region still obey the horizontal-message and monotonic-`y` rules. Multi-region operators (`alt`, `par`) must use the fragment's native multi-row layout (`meta.row_num` ≥ 2 with matching `row_sizes`), not stacked separate fragments. Do not flatten branches into a single sequential list of messages.
  - `alt` means alternative / conditional branches: exactly one region runs based on its guard, with each mutually exclusive outcome represented by its own guarded region. When the source facts contain mutually exclusive outcomes, the diagram **must** include an `alt` frame covering those branches. Required triggers include success / failure, approved / rejected, authorized / unauthorized, exists / not found, enough quota / insufficient quota, in stock / out of stock, paid / unpaid, valid / invalid, and any user-described "如果...否则..." logic.
  - `opt` means an optional region: the region runs only when its guard is true, like a single `if` without `else`.
  - `loop` means repeated execution while the guard or count condition holds, like `for` / `while`.
  - `par` means parallel regions that may execute concurrently. Keep each region visually separated and do not imply an ordering between parallel branches unless messages make that ordering explicit.
- **Fragment scope discipline (vertical and horizontal).** A `combined_fragment` is a *minimal* box around the messages that genuinely depend on its guard.
  - **Vertical:** the frame's top edge sits just above the first message of the first region; the bottom edge sits just below the last message of the last region. Do not let it spill into earlier setup or later unrelated flows.
  - **Horizontal:** the frame spans only the lifelines that send or receive at least one message inside the fragment. Do not stretch it across every lifeline on the canvas "for symmetry" — a fragment crossing an idle lifeline implies that participant is part of the decision, which is misleading. If an outer participant must appear inside for a single side-call, keep the frame tight; do not absorb unrelated downstream lifelines.
- **One outcome per branch message (no mixed labels).** A message inside a specific region must describe only that region's outcome. Labels like `成功或业务错误`, `结果（成功 / 失败）`, `返回 200 或 400` violate `alt` semantics because they smuggle both branches into one arrow. Split them: the success arrow lives inside `[成功]` and reads `返回 201 报名成功`; the failure arrow lives inside `[失败]` and reads `返回 400 名额不足`. The guard already encodes the outcome — the message label must agree with the guard, not hedge against it.
- **Branch validation checklist.** Before writing the board back, walk every `combined_fragment` and answer **yes** to all of these. A single **no** means do not write yet — fix the raw first.
  1. **Type is `combined_fragment`?** Not a plain rectangle, not grouped rectangles, not an SVG polygon.
  2. **Operator matches intent?** `alt` for mutually exclusive, `opt` for single-side optional, `loop` for repetition, `par` for concurrent.
  3. **Multi-region operator uses one fragment?** `meta.row_num` ≥ 2 with matching `meta.row_sizes`. Never stacked separate frames.
  4. **Every region carries a `[guard]` label?** And the guards in one `alt` are mutually exclusive and collectively exhaustive against the modeled scope.
  5. **No empty region?** Each region has ≥ 1 bound message connector. If a region is truly empty, switch to `opt`.
  6. **Causality respected?** The read / validate that produces the guard sits *above and outside* the fragment.
  7. **Closed loop in every region?** Every backend-touching region has an explicit `后端 → 前端` response before any `前端 → 用户` prompt — including negative regions (error code / 4xx).
  8. **Negative outcome is a domain event?** `[缺勤]` / `[拒绝]` / `[超时]` / `[作废]` regions persist an explicit status (write to DB) instead of leaving silence.
  9. **One outcome per message?** No label combines `成功 / 失败` or `200 / 400` into a single arrow.
  10. **Frame scope is minimal?** Vertical span covers only the dependent messages; horizontal span covers only the participating lifelines; no unrelated downstream flow is enclosed.
  11. **Monotonic `y` across fragments?** Even with nested or sequential fragments, business causality (`报名 → 出勤 → 评价`) reads top-to-bottom without re-ordering.
  12. **Contended-resource flow has an annotation?** When the flow modifies 容量 / 积分 / 库存 / 余额 or asserts write-once (评价 / 支付 / 退款), a `note_shape` marks the transaction boundary or idempotency key.
- **Causality before branching (READ stays OUTSIDE the `alt`).** The query / validation that *produces* the branch condition must sit **above** and **outside** the `combined_fragment`, not inside one of the regions. A system cannot evaluate a guard before the data that determines the guard exists. Concretely: messages like `查询容量` / `校验积分` / `读取出勤记录` / `查询用户状态` belong on the lifelines **before** the `alt` frame opens; only the divergent **write / return** chains live inside the regions. Putting the read inside `[条件通过]` implies the system can foresee the result — that is a causality bug, not a stylistic choice. Triggers: any `alt` whose guard text references a fact the backend has to look up (`已出勤`, `名额足够`, `积分充足`, `订单存在`, `已审核`, `未重复`).
- **Closed-loop in every branch (no silent backend).** Each `alt` / `opt` region that involves a backend call must contain a full **request → backend work → backend-to-frontend response** chain. A region that only shows `前端 → 用户` (the toast/prompt) without an upstream `后端 → 前端` return is broken: the frontend would have to guess the outcome or rely on timeout. Synchronous calls return a solid-arrow request + dashed-arrow response on the same activation; asynchronous flows return via callback / push and the diagram must show that callback. Negative branches (`[校验失败]`, `[名额不足]`, `[已重复评价]`) carry the same closed-loop requirement as positive ones — show the explicit error response (业务错误码 / HTTP 4xx / RPC error) before the frontend prompts the user.
- **No empty branch, every negative path is a domain event.** Every region inside an `alt` / `par` must contain at least one bound message connector. An empty region is a bug — if a branch truly has no behavior, drop the `alt` and use `opt` for the single active branch. Negative outcomes (`[缺勤]`, `[拒绝]`, `[超时]`, `[作废]`) are **first-class domain events**: the backend must persist an explicit status (`status = ABSENT` / `REJECTED` / `EXPIRED`) and return a definite response. Conflating "未操作" with "明确缺勤 / 明确拒绝" is a data-modeling defect that propagates into analytics, settlement, and audit; the sequence diagram must make the persistence step visible, not silent.
- **Transactional & idempotency annotations (recommended).** When the flow touches contested resources (容量 / 积分 / 库存 / 余额) or write-once semantics (评价 / 支付 / 报名 / 退款), attach a `note_shape` next to the relevant activation or fragment to mark the transactional boundary or idempotency key — for example `事务: SELECT ... FOR UPDATE / 乐观锁 version`, `幂等键: enrollment_id + user_id 唯一索引`, `Redis 预扣减 + MQ 异步落库`. These annotations do not change the message structure; they tell the reader that the gap between `校验` and `写入` is protected. Omitting them on a flow that obviously needs them (秒杀报名、并发评价、积分扣减) is a design smell the diagram should surface, not hide.
- **Forbidden constructions.**
  - Creating the sequence diagram as an image, SVG, Mermaid / PlantUML render, Graphviz output, draw.io export, or any embedded artifact instead of editable Feishu native nodes.
  - Using top participant headers that do not follow `Browser` → `:{Business}Window` / `:{Business}Page` / `:{Business}View` → `:{Business}Controller` → `:{Business}Service` → `:{Business}Dao` → `DB` for a standard backend flow.
  - Collapsing `Browser` and the business view/window/page into one participant when the flow has a UI layer.
  - Writing message labels without a strict `N: description` prefix, using non-numeric prefixes like `B:`, or leaving message connectors undescribed.
  - Labeling the Dao participant as `Mapper`, `Repository`, `Model`, ORM client, or framework class instead of normalizing the header to `:{Business}Dao`.
  - Drawing a single diagonal connector between two participant heads with no lifelines.
  - Replacing "lifeline + horizontal message" with one point-to-point polyline.
  - Letting an arrow's `y` shift while it crosses other lifelines in the source.
  - Floating activation bars not aligned to any lifeline.
  - Showing mutually exclusive branch messages one after another without an enclosing native `combined_fragment` (`alt`) frame.
  - Faking a combined fragment with a plain native rectangle, grouped rectangles, or a freeform polygon — the operator pentagon and region separators are part of the semantics and only the native `combined_fragment` shape provides them.
  - Putting the **read / validate** message that decides the `alt` guard *inside* one of the regions instead of above the fragment — that asserts the impossible: branching before the data arrives.
  - Leaving a backend-touching branch with only a `前端 → 用户` prompt and no `后端 → 前端` response arrow — the frontend cannot legitimately know the outcome without the return message.
  - Leaving any `alt` / `par` region empty, or treating a negative branch as "no-op" instead of writing an explicit status (`ABSENT` / `REJECTED` / `EXPIRED`) and returning a definite response.
  - Stretching an `alt` frame vertically so it encloses messages that belong to an **unrelated** later flow (e.g. wrapping `教师读取名单` or `学员提交评价` inside the `[报名条件通过]` region). The frame's `y` range must cover only the messages that depend on its guard.
  - Stretching an `alt` frame horizontally across lifelines that send or receive **no** message inside any of its regions (e.g. dragging the frame across the `学员` actor lifeline when the branching is purely a backend / DB decision). The horizontal span must cover only participating lifelines.
  - Writing a single hedged label like `成功或业务错误` / `200 or 400` / `结果(成功 / 失败)` on one arrow inside an `alt`. Each region's arrow must describe exactly that region's outcome.
  - Reordering messages so that a downstream business step (e.g. `学员提交课程评价`) appears above an upstream step (e.g. `教师确认出勤`) on the canvas. Message `y` must increase monotonically along business causality, even when fragments are nested.

## Forbidden mixings

- Process branches and swimlanes — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- Use case actors with boundaries — those belong in `lark-uml:usecase`.
- Deployment layering — that belongs in `lark-uml:architecture`.
- Network link topology — that belongs in `lark-uml:network`.

## Native node composition

Build the sequence diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, SVG, Graphviz, draw.io, raster image, screenshot, or any other non-native artifact.

- **Participant header** — Exactly six standard top headers for normal backend flows: `Browser`, real business view/window/page (e.g. `:SearchBookWindow`), `:{Business}Controller`, `:{Business}Service`, `:{Business}Dao`, and `DB` (all native participant heads). All headers share the same top `y`, uniform size, and uniform spacing. Headers are produced by cloning an existing header of the same kind or by creating the same native type directly.
- **Lifeline** — native vertical line / dashed line directly below its participant header. Lifelines are strictly vertical, identical length, bottom-`y` aligned, with `x` equal to the center `x` of the header.
- **Activation bar** — native narrow rectangle centered on the lifeline. It belongs to the lifeline visually but remains a distinct node so that messages can bind to it.
- **Message connector** — native `type: "connector"` whose `connector.from` and `connector.to` reference participant / lifeline / activation node ids. Both endpoints share the same `y`. Synchronous calls use solid line + filled arrow; returns use dashed line + open arrow. Message text goes in `connector.label`.
- **Self-call** — native connector from a lifeline back to the same lifeline, rendered as a small right-angle loop on the right side. Endpoints still bind to the same node id at top and bottom of the loop.
- **Combined fragment frame** — native **`type: "combined_fragment"`** shape (飞书画板"组合片段"图形) spanning the involved lifelines. The operator (`alt`, `opt`, `loop`, `par`) sits in the top-left pentagon corner via the node's `text` field; multi-region operators carry their regions inside `meta.row_num` / `meta.row_sizes`, with each region's guard label such as `[报名人数未满]` / `[报名人数已满]` written into the per-row header. **Never substitute** a plain rectangle, grouped rectangles, or SVG polygon — those render without the operator pentagon and region separators and fail the diagram's semantics. Fragment frames are annotations around messages; they do not replace lifelines, activation bars, or bound message connectors.

Every message connector must be a native connector with bound endpoints. Diagonal "point-to-point" lines or freeform polylines are not acceptable substitutes.

## Native type cheat-sheet

Pick the native `type` by intent before you emit any node. The full matrix lives in [`../../references/native-types.md`](../../references/native-types.md); the sequence row spells out exactly which types are allowed and which substitutes will silently break the diagram. **Sequence-specific reminders that override generic instincts:**

1. **Participant head** → First column: native participant head labeled `Browser`. Second column: native participant head labeled with the real business view/window/page (e.g. `:SearchBookWindow`). System layers: native participant heads labeled `:{Business}{Layer}` (e.g. `:BookController`, `:BookService`, `:BookDao`). Storage: native participant head labeled `DB`. Never `text_shape`, imported images, screenshots, or freeform drawings.
2. **Vertical line under a head** → `life_line`. Never a `connector`.
3. **Activation bar** → narrow `life_line` (or `activation` if the template uses that variant). Never a plain rectangle.
4. **Arrow between two lifelines** → `connector` with `attached_object` ids on BOTH ends. Coordinate-only endpoints, diagonals, or polylines that bypass a lifeline are all bugs.
5. **Branching / looping / parallel region** → `combined_fragment`. Multi-region operators (`alt`, `par`) live inside ONE fragment's `meta.row_num` / `meta.row_sizes`, never as stacked separate frames, never as plain rectangles.
6. **Side commentary** → `note_shape` (anchored) or `text_shape` (legend only — never for participant / message / fragment text).
7. **Sanity sweep before writing back:** every emitted `type` must appear in the sequence row of `references/native-types.md`. Any rogue `type` means the wrong shape was picked.

## Final validation checklist

Before calling `lark-cli whiteboard +update --input_format raw`, validate all of the following. Any failure means fix the raw first or stop and report the blocker.

1. Top participants are exactly ordered as `Browser` → `:{Business}Window` / `:{Business}Page` / `:{Business}View` → `:{Business}Controller` → `:{Business}Service` → `:{Business}Dao` → `DB` for a standard backend flow.
2. All participant headers, lifelines, activation bars, message arrows, fragments, and notes are Feishu / Lark native raw nodes using approved native types.
3. No SVG, Mermaid, PlantUML, Graphviz, draw.io, raster image, screenshot, imported arrow, plain rectangle frame, or coordinate-only connector appears in the edited sequence region.
4. Controller, Service, Dao methods, parameter types, return types, and DB operation labels come from the real project source or user-provided identifiers.
5. Every message connector is bound to native node ids at both ends and is horizontal.
6. Message numbering is continuous from top to bottom across forward and return messages, and every label uses `N: description` with a non-empty description.
7. Dao participant header ends with `Dao`, not `Mapper`, `Repository`, `Model`, or an ORM-specific name.
