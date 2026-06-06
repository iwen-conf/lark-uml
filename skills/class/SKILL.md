---
name: lark-uml:class
description: >
  飞书画板系统主要类图：执行型 Skill。当用户要在飞书画板上绘制或修改"系统主要类图"（OOAD 业务对象、属性 / 参数、主要方法、继承 / 实现 / 关联 / 依赖 / 聚合 / 组合关系）时使用。
  必须使用飞书原生 table_uml 类框和 connector 关系线：属性行按 `-name: Type` 或模板既有的 `- name: Type`，主要方法按 `+method(): ReturnType` 或模板既有格式；普通关联默认用带箭头的 `line_arrow`，端点附近标 `1` / `0..*` 等多重性。
  本图不是 Java / Go / Python 源码图，也不是数据库表结构；严格区分 lark-uml:er，直接改写白板，不输出 PlantUML / Mermaid / SVG。
---

# `lark-uml:class`

Specialist skill for **system main class diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

This diagram is an **object-oriented analysis and design (OOAD) abstraction**. It captures the system's main business objects, their responsibilities, and their relationships at the software-design level. It is **language-agnostic**: not a Java class diagram, not a Go struct diagram, not a Python module dump, **not a database table physical design**.

For database tables / columns / primary-and-foreign keys / cardinality between storage tables, use `lark-uml:er` instead.

### Class diagram ≠ database physical design

This is the single most common drift, so spell it out before drawing anything:

| Aspect | `lark-uml:class` (this skill) | `lark-uml:er` |
| --- | --- | --- |
| What it models | Business objects, responsibilities, and course-paper entity classes | Storage tables, columns, constraints |
| Members | OO attributes + optional key business methods (`-field: Type` / `+method(): ReturnType`) | Field name + DB type + `PK` / `FK` / `NN` markers |
| Relationships | Inheritance, realization, association, aggregation, composition, dependency | Foreign-key references with crow-foot cardinality |
| Multiplicity notation | Visible endpoint text such as `1`, `0..1`, `1..*`, `0..*` | `1:1`, `1:N`, `N:M` crow-foot |
| What it survives | Storage swap, ORM swap, language swap | Only the current schema |
| Audience | Architects, OO designers, paper readers | DBAs, ORM authors, migration writers |

A class diagram that carries `VARCHAR(64)`, `BIGINT`, `PK` / `FK` row markers, junction tables, or `ON DELETE CASCADE` has drifted into the wrong skill. Stop, move that material to `lark-uml:er`, and bring the class diagram back to business objects and behaviors.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the class diagram from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

Do **not** accept an `implementation_language` / `language_hint` input. The class diagram is an OOAD model and must not be reshaped around any specific programming language's syntax or framework idioms.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native class boxes and native connectors in place, then write raw back. Inheritance, realization, association, aggregation, composition, and dependency relationships are business relationships, so their endpoints must bind to class / interface node ids. **No PlantUML / Mermaid / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing class boxes and relationship connectors. Only redraw when the user explicitly asks, or the existing diagram is the wrong type entirely.

## Hard failure gates

Before drawing, editing, or writing, apply these gates. If any gate fails, **do not call `whiteboard +update`**. Fix the raw structure first; if the missing information cannot be inferred from code / docs / the board, stop and ask.

- **MUST use native nodes.** Every class / interface / abstract class MUST be one `table_uml` node. Every business relationship MUST be one native `connector`.
- **MUST NOT fake class boxes.** Three stacked rectangles, grouped shapes, free text, SVG, Mermaid, PlantUML, images, or geometric lines that only look like UML are invalid.
- **MUST use Feishu member notation.** Attribute / parameter rows MUST use `-name: Type` or the existing template's `- name: Type`; main method rows, when present, MUST use `+method(): ReturnType` or the existing template's `+ method(): ReturnType`.
- **MUST NOT use the old member format.** Rows such as `+Long userId`, `+login()`, DB column definitions, annotations, tags, source-language signatures, or framework decorators are invalid.
- **MUST NOT pad methods.** Include methods only when they are important business behaviors or when the existing template already has a method region. Reference-style entity class diagrams MAY omit method rows entirely; do not add getters, setters, trivial CRUD wrappers, one-line helpers, or framework hooks just to fill the box.
- **MUST use the class arrow standard.** Class relationships MUST use `empty_triangle_arrow`, `line_arrow`, `diamond_arrow`, or `empty_diamond_arrow` exactly as defined below. In the teacher-reference / course-paper style, ordinary associations default to a visible open arrow (`line_arrow`) on the target end. `zero_or_single_arrow` and `zero_or_multi_arrow` are invalid for class relationships.
- **MUST show endpoint multiplicity.** Associations, aggregations, and compositions MUST show multiplicity text near both line endpoints whenever the count can be inferred. For school / graduation diagrams, prefer `1` and `0..*` exactly as in the accepted reference image.
- **MUST bind relationship endpoints.** Final business connectors MUST bind to surviving class / interface node ids. Coordinate-only endpoints are invalid for relationships and are allowed only inside legends or non-business annotations.
- **MUST keep class / ER boundaries.** If the requested content is mostly tables, columns, PK / FK, indexes, storage constraints, or DB cardinality, stop and use `lark-uml:er`.

## System discovery — read the code first

A class diagram describes the system that actually exists. Before drawing or editing, the agent must inspect the codebase and other system evidence, then abstract OOAD concepts from it. Do not design from the user's prose alone, and do not invent classes that have no counterpart in the real system. "Language-agnostic" applies to the **output**; the **input** is the project's real code.

### What to read (in priority order)

1. **Domain / model code** — packages or folders named `domain`, `model`, `entity`, `aggregate`, `core`. These are the closest counterparts to the OO objects you will draw.
2. **Service / use-case code** — packages or folders named `service`, `usecase`, `application`, `biz`. These reveal the business behaviors each object exposes (the methods on the diagram).
3. **Controller / handler / route code** — these reveal which objects are exposed externally and how the system is sliced.
4. **README, design docs, ADRs, API specs, prior whiteboards, the user's current task description** — these carry intent that code alone cannot.
5. **Database schema** — only as a cross-check for "what business concept this object represents". Do **not** copy column lists into class boxes; that is `lark-uml:er`'s job.

If the system spans multiple modules / repos, scope discovery to what the user actually asked about. Do not try to model the whole company.

### How to abstract from code to OOAD

- **Collapse infrastructure noise.** ORM annotations, DTO / VO / DO / PO classes, request / response wrappers, mappers, repositories, JSON tags, dependency-injection wiring — these are plumbing, not design. Hide them behind the business concept they serve.
- **Keep responsibilities, drop accessors.** `getX()` / `setX()` / `toString()` / `equals()` / `hashCode()` rarely belong on a design diagram. Surface methods that change state or drive use cases (`publish()`, `submit()`, `approve()`, `assign()`).
- **Merge near-duplicates.** `UserDO`, `UserDTO`, `UserVO`, `UserEntity`, `UserModel` collapse into one business `User` on the design diagram.
- **Promote relationships, not foreign keys.** A `Course.ownerId` field in code becomes a `1..* — 1` association from `Course` to `User` on the diagram, not a column.
- **Stateless services still count.** A service class that is a transaction script with no fields can still appear — its responsibilities (the methods) are what the diagram documents.
- **Course-paper entity diagrams are allowed.** If the target is a graduation / course-design "系统主要类图" and the source project is a CRUD-style information system, it is acceptable to show persistent entity classes with attributes only, matching the source entity names and simple logical types. Still do not show `PK` / `FK` / `NN` markers, SQL DDL types, indexes, or cascade rules; those remain ER-only.

### Language-agnostic ≠ code-blind

Reading the code does not mean copying its syntax. The job is to recognize the OOAD concepts the code is implementing and draw those concepts.

- If the codebase is Go (no class inheritance) but the design is clearly "an `AdminUser` is a kind of `User`", the diagram may legitimately show generalization.
- If the codebase is Java with deep inheritance hierarchies that are pure technical scaffolding, the diagram may collapse them.
- The diagram represents the **design**, not the source tree.

### When the system is unknown

If the codebase is not reachable and the user has not supplied enough business context, stop and ask. Do not draw a generic textbook class diagram and present it as the system's design.

## Diagram-specific rules

### OOAD abstraction, not language syntax

- The diagram models **business objects** (User, Admin, Course, Exam, Question, Submission, Message, ...), their **responsibilities** (what each object is in charge of), and the **relationships** between them.
- It does **not** model: Java packages, Go structs vs interfaces stereotypes, Python module layout, C++ templates, framework-specific decorators, ORM annotations, serialization tags.
- Treat the diagram as something that would be valid in any OO language. If a reader has to know your implementation stack to read it, you have drifted into source code, not design.

### Class box structure

Every class is a two- or three-region box, top to bottom:

1. **Class name** — business-meaningful noun. Prefer PascalCase for greenfield OOAD diagrams (`User`, `Admin`, `Course`, `Exam`, `Question`, `Submission`, `Message`), but in teacher-reference / course-paper entity diagrams preserve the project's canonical entity names exactly, including lowercase pinyin names such as `yonghu`, `shangjia`, `cheliangxinxi`, and mixed names such as `User`.
2. **Attributes** — business state fields written as private member rows: `<visibility><name>: <Type>` in the accepted teacher-reference style, or `<visibility> <name>: <Type>` when preserving a template that already uses a space. Use `-` by default for class attributes.
   - Preferred school-reference format: `-id: Long`, `-title: String`, `-startTime: DateTime`, `-enabled: Boolean`, `-score: Double`.
   - Template-preserving format: `- id: Long`, `- title: String`, `- startTime: LocalDateTime`.
   - Allowed type vocabulary: `Long`, `Integer`, `String`, `Boolean`, `Double`, `Float`, `LocalDate`, `LocalDateTime`, `DateTime`, `Timestamp`, `BigDecimal`, `LongText`, simple collection wrappers, and project-specific enums. These read as generic OO type hints (Java / Kotlin / C# / TypeScript-style); they are **not** a commitment to Java.
   - Never write **storage-layer types** (`VARCHAR(64)`, `BIGINT NOT NULL`, `TEXT`, `DATETIME(3)`), **framework decorators or tags** (`@Column`, `@JsonProperty`, `db:"user_id"`, `binding:"required"`), or **language-specific syntax noise** (`*string` Go pointer, `Optional[T]`, complex generics like `Map<K, List<V>>`).
3. **Methods** — optional. Include only the class's main OO business behaviors when they matter for the diagram. Simple accessors, trivial helpers, framework hooks, and obvious CRUD wrappers usually do not belong here. If the reference / template uses attribute-only boxes, do not create an empty method region.
   - Preferred school-reference format: `+login(): Boolean`, `+publish(): void`, `+submit(): void`.
   - Template-preserving format: `+ login(): Boolean`, `+ publish(): void`.
   - Empty parameter lists are the default. Add parameters only when they are essential to the design meaning. Add a generic return type after `:` when known; use `void` for command-style behavior. Do not write concrete language signatures (`func (s *Service) Place(o *Order) error`, `public Order placeOrder(OrderDTO dto) throws Exception`).

### Feishu standard class box

When creating a class box from scratch, reproduce the standard model:

- One `table_uml` node, one column, one row per visible line.
- Row 1 is the class name: light grey fill, centered, bold, e.g. `DisplayObject`.
- Attribute rows are white, left-aligned, e.g. `-pos: Point`, `-matrix: Matrix`, `-style: Style` (or preserve existing `- pos: Point` spacing).
- Method rows are white, left-aligned, e.g. `+draw(): void` (or preserve existing `+ draw(): void` spacing).
- Clone the existing template's row height, border width, font size, fill colors, and text alignment whenever available. Only fall back to the standard model when no template class box exists.

### Visibility markers

- `+` public, `-` private, `#` protected, `~` package in UML terms.
- For this project's Feishu class-diagram standard, `-` is also the required marker for class parameters / fields, and `+` is the required marker for the class's main methods. Treat this as the diagram convention first; use `#` / `~` only when that visibility is design-significant or already used by the board's template.
- Core business classes should show their main responsibilities as `+` methods when this helps the diagram. Teacher-reference entity diagrams, simple value objects, lightweight records, or classes whose behavior is obvious may omit methods; do not pad them with trivial getters, setters, or one-line helpers.

### Relationships (UML standard, language-agnostic)

| Relationship | Line | Arrow head | Native endpoint style | Direction |
| --- | --- | --- | --- | --- |
| Inheritance (generalization) | solid | hollow triangle | `empty_triangle_arrow` on parent end | child → parent |
| Realization | dashed | hollow triangle | `empty_triangle_arrow` on interface end | implementer → interface |
| Association | solid | open arrow by default; no arrow only for true bidirectional / symmetric association | `line_arrow` on target end; no-arrow only when explicitly bidirectional | source class → target class |
| Dependency | dashed | open arrow | `line_arrow` on supplier end | dependent → supplier |
| Composition | solid | filled diamond on the whole side | `diamond_arrow` on whole end | part → whole |
| Aggregation | solid | hollow diamond on the whole side | `empty_diamond_arrow` on whole end | part → whole |

Treat these six rows as a validation table. A class relationship connector that does not match its row is invalid. If a class diagram connector uses `zero_or_single_arrow` / `zero_or_multi_arrow`, it has inherited ER / flowchart notation and MUST be fixed before writing.

### Multiplicity

- Label both ends of associations, aggregations, and compositions when the count is meaningful. In course-paper / teacher-reference diagrams, do this for every visible relationship unless the count is genuinely unknowable.
- Use `1`, `0..1`, `1..*`, and `0..*` as the default notation. Prefer `0..*` over `N` / `M` when matching the accepted reference image; use `N`, `M`, or `N:M` only when the existing template or user explicitly asks for that notation.
- Place multiplicity text close to each line endpoint, just outside the class box edge. Do not put endpoint multiplicity in the middle of the line, and do not let labels overlap borders, arrowheads, or other labels.

### Management / control relationships

- When an actor-style class (e.g. `Admin`) controls or oversees others (e.g. `User`, `Course`, `Exam`), draw a **dashed dependency arrow** from the manager to the managed class and label it `manage` (or the user-supplied verb such as `审核` / `分配` / `调度`). Do not promote management into composition unless the managed object's lifecycle truly ends with the manager's.

### Layout discipline

Organize the canvas top-to-bottom by role:

- **Top tier** — system management / administrative classes (`Admin`, `Operator`, role / permission objects).
- **Middle tier** — core business objects (`User`, `Course`, `Exam`, `Question`, ...).
- **Bottom tier** — association, junction, and record classes (`Submission`, `Answer`, `Enrollment`, `Message`, logs / records).

Within a tier, group related classes near each other; route connectors to minimize crossings; keep class-box widths consistent.

### Visual style for first-time initialization

When the board is empty and the agent is drawing from scratch (no template to clone), reproduce the traditional academic / software-engineering UML look. Concretely:

- **Class box** — rectangle with a thin black border, divided by horizontal lines into two or three regions stacked top-to-bottom: name / attributes / optional methods.
- **Name region** — light grey fill, class name centered, bold or regular weight per the template; the attribute and method regions are white.
- **Member rows** — left-aligned, one per line. Attribute rows use the teacher-reference form `-name: Type` by default (`-id: Long`, `-addtime: Timestamp`), while existing templates that already use `- name: Type` keep their spacing. Method rows, when present, use the same spacing rule (`+method(): ReturnType` or template-preserved `+ method(): ReturnType`).
- **No card chrome** — no drop shadows, no rounded corners, no full-color body fills, no gradient backgrounds, no glyph icons. The reference is StarUML / Visio / ProcessOn UML exports, not a marketing slide.
- **Connectors** — black thin lines. **Solid** for inheritance, association, aggregation, composition. **Dashed** for realization, dependency, and `manage`-style control arrows.
- **Association arrows** — ordinary associations use visible arrowheads. In native raw this means `line_arrow` on the target end and no arrow style on the source end. Do not leave a normal association as a bare line just because the multiplicity text is present.
- **Endpoint labels** — multiplicity numbers (`1`, `0..1`, `1..*`, `0..*`) sit at each line endpoint, close to the class box, matching the accepted reference image's `1` / `0..*` placement.
- **Mid-line labels** — relationship verbs (`manage`, `审核`, `分配`) sit on the middle of the dashed control line.
- **Layout flow** — top tier (management classes) at the top of the canvas, junction / record classes at the bottom; control arrows go top-down, association lines go vertically between the middle and bottom tiers.

When the board already has a template, follow the template's existing class-box style (modify-in-place rule from `boundaries.md`) — do not retheme it into a different visual vocabulary just to match the academic look.

### Naming style

- Class names: business nouns. For greenfield OOAD diagrams, use PascalCase (`User`, `Admin`, `Course`, `Exam`, `Question`, `Message`, `Submission`). For a reference-driven school diagram, preserve the source / template's class names exactly, even when they are lowercase pinyin or generated entity names (`yonghu`, `shangjia`, `huanchejilu`).
- Attribute names: lowerCamelCase business fields (`userId`, `title`, `status`, `createTime`) for greenfield diagrams; preserve source / template names in reference-driven school diagrams (`addtime`, `shangjiazhanghao`, `haicheshijian`).
- Method names: lowerCamelCase business verbs (`login()`, `publish()`, `submit()`, `edit()`, `delete()`).
- Do not embed language keywords (`class`, `struct`, `interface { ... }`, `def`, `func`) into the visible text. Stereotypes such as `«interface»` or `«abstract»` are allowed on the name row when they carry design meaning.

## Forbidden mixings

- **ER / database schema** (column types, `PK` / `FK` markers, `NN` / `UQ` / `IDX`, junction tables, crow-foot cardinality) — those belong in `lark-uml:er`.
- **Language source code or syntax** (Java `class … extends …`, Go `type … struct { … }`, Python `class …:`, `func (...) (...)`, `@Annotation`, generics like `List<Order>`) — the diagram is OOAD, not source.
- **Process steps and decisions** — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- **Sequence messages, lifelines, activation bars** — those belong in `lark-uml:sequence`.
- **Deployment / network topology** — those belong in `lark-uml:architecture` / `lark-uml:network`.
- **Modern UI / marketing card style** — coloured tiles, drop shadows, glyph icons, gradient fills. The class diagram is a design artifact, not a slide.

## Native node composition

Pick every native `type` from the matrix in [`../../references/native-types.md`](../../references/native-types.md) (see the class row). Every class / interface / abstract box must be a single `table_uml` node (header row + attribute rows + optional method rows inside `table.cells`), **never** three stacked `composite_shape` rectangles. Relationships are `connector` nodes, with arrow-head style (hollow / filled triangle / diamond / open) carrying the relationship semantics.

Build the class diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **Class box** — native rectangle, divided into two or three stacked regions (class name on top, attributes below, optional methods at the bottom). The board's existing class boxes already use a specific structure; clone one and overwrite its text rather than building a new shape.
- **Interface / abstract class box** — same native rectangle layout, with a `«interface»` or `«abstract»` stereotype on the name row (or whatever marker the template already uses).
- **Visibility markers** — keep `+` / `-` / `#` / `~` in front of each member.
- **Inheritance connector** — native `type: "connector"`, solid line, `empty_triangle_arrow` on the parent end, from child class id to parent class id.
- **Realization connector** — native connector, dashed line, `empty_triangle_arrow` on the interface end, from implementer id to interface id.
- **Association connector** — native connector, solid line, `line_arrow` on the target end by default and no arrow style on the source end. Use no-arrow ends on both sides only for explicitly bidirectional / symmetric associations or when preserving an existing template's deliberate bare association style. Role / multiplicity labels sit near both ends.
- **Dependency connector** — native connector, dashed line, `line_arrow` on the supplier end. Use this for `manage` / `审核` / `分配` style control relationships and for transient usage dependencies.
- **Composition connector** — native connector, solid line, `diamond_arrow` on the whole-class end.
- **Aggregation connector** — native connector, solid line, `empty_diamond_arrow` on the whole-class end.

For no-arrow ends on bound business relationships, omit `arrow_style` unless the existing template connector already uses another native no-arrow representation. `arrow_style: "none"` is allowed only when preserving a non-business legend / template stroke; it MUST NOT replace a semantic UML arrow.

For every relationship connector, `connector.from` and `connector.to` must be ids of real class / interface boxes that survive the edit. Clone the closest existing connector with the right arrow style before adjusting endpoints.

## Class-specific pre-write checklist

All checks MUST pass before `whiteboard +update --input_format raw`:

- Every class-like object is `type: "table_uml"` and has `table.cells`; no class is represented by child rectangles, free text, SVG, Mermaid, PlantUML, or an image.
- Class names are business nouns; greenfield OOAD diagrams use singular PascalCase, while reference-driven school diagrams preserve the source / template names exactly. Request / response wrappers, DTO / VO / DO duplicates, mappers, repositories, and ORM artifacts are collapsed or omitted unless they are real design concepts.
- Attribute rows match `^- ?[A-Za-z][A-Za-z0-9_]*: [A-Za-z][A-Za-z0-9_]*(.*)?$` in spirit; no `PK` / `FK` / `NN` markers, SQL DDL types, annotations, tags, or source-code syntax.
- Method rows, when present, match `^\+ ?[a-z][A-Za-z0-9_]*\(.*\): [A-Za-z][A-Za-z0-9_]*$` in spirit; no accessors, trivial helpers, or source-language signatures.
- Relationship connectors bind to final class / interface node ids and use the exact class arrow styles listed above.
- No class relationship connector uses `zero_or_single_arrow`, `zero_or_multi_arrow`, coordinate-only endpoints, decorative lines, SVG strokes, Mermaid strokes, or PlantUML strokes.
- Ordinary association connectors in teacher-reference style have a visible `line_arrow` at the target end; a bare no-arrow association has a documented bidirectional / symmetric reason.
- Plain association, aggregation, and composition labels use one notation consistently (`1`, `0..1`, `1..*`, `0..*` by default; `N`, `M`, `N:M` only when requested or template-established).
- The preview image still reads as an academic UML class diagram: black thin lines, rectangular class boxes, visible relationship arrows, endpoint multiplicity labels, no colorful cards, no gradients, no decorative icons.

If any check fails after writing, immediately re-query raw, fix the invalid node / connector, and overwrite the board again. Do not report success while a failed check remains.
