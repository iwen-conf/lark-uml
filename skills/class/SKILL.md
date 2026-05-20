---
name: lark-uml:class
description: 飞书画板系统主要类图：执行型 Skill。当用户要在飞书画板上绘制或修改"系统主要类图"（面向对象分析与设计：业务对象、属性、行为方法、关联 / 聚合 / 组合 / 继承 / 实现 / 依赖关系）时使用。本图是语言无关的 OOAD 抽象，不是 Java / Go / Python 的语法图，也不是数据库表结构。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 PlantUML / Mermaid / SVG 代码。本图与数据库关系图（lark-uml:er）严格区分。
---

# `lark-uml:class`

Specialist skill for **system main class diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

This diagram is an **object-oriented analysis and design (OOAD) abstraction**. It captures the system's main business objects, their responsibilities, and their relationships at the software-design level. It is **language-agnostic**: not a Java class diagram, not a Go struct diagram, not a Python module dump, **not a database table physical design**.

For database tables / columns / primary-and-foreign keys / cardinality between storage tables, use `lark-uml:er` instead.

### Class diagram ≠ database physical design

This is the single most common drift, so spell it out before drawing anything:

| Aspect | `lark-uml:class` (this skill) | `lark-uml:er` |
| --- | --- | --- |
| What it models | Business objects, behaviors, responsibilities | Storage tables, columns, constraints |
| Members | OO attributes + business methods | Field name + DB type + `PK` / `FK` / `NN` markers |
| Relationships | Inheritance, realization, association, aggregation, composition, dependency | Foreign-key references with crow-foot cardinality |
| Multiplicity notation | `1`, `n`, `0..1`, `1..*`, `0..*` | `1:1`, `1:N`, `N:M` crow-foot |
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

Every class is a three-region box, top to bottom:

1. **Class name** — capitalized, singular, business-meaningful noun. Examples: `User`, `Admin`, `Course`, `Exam`, `Question`, `Submission`, `Message`.
2. **Attributes** — business state fields written with a **generic OO type prefix**. The academic UML convention is `<visibility><Type> <name>`, where the type is a language-neutral OO type (not a storage type, not a framework decorator).
   - Format: `+Long userId`, `+String title`, `+Integer status`, `+LocalDateTime startTime`, `+Boolean enabled`, `+Double score`.
   - Allowed type vocabulary: `Long`, `Integer`, `String`, `Boolean`, `Double`, `Float`, `LocalDate`, `LocalDateTime`, `BigDecimal`, simple collection wrappers, and project-specific enums. These read as generic OO type hints (Java / Kotlin / C# / TypeScript-style); they are **not** a commitment to Java.
   - Never write **storage-layer types** (`VARCHAR(64)`, `BIGINT NOT NULL`, `TEXT`, `DATETIME(3)`), **framework decorators or tags** (`@Column`, `@JsonProperty`, `db:"user_id"`, `binding:"required"`), or **language-specific syntax noise** (`*string` Go pointer, `Optional[T]`, complex generics like `Map<K, List<V>>`).
3. **Methods** — OO business behaviors that express the object's responsibility.
   - Format: `+login()`, `+publish()`, `+submit()`, `+edit()`, `+delete()`, `+enter()`, `+deleteMessage()`.
   - Empty parentheses are the default. Do not write concrete language signatures with parameter types or return types (`func (s *Service) Place(o *Order) error`, `public Order placeOrder(OrderDTO dto) throws Exception`).

### Visibility markers

- `+` public, `-` private, `#` protected, `~` package. Apply to both attributes and methods.
- A class with zero methods is almost always wrong — it is a data record, not a business object. Either give it responsibilities or move the table to `lark-uml:er`.

### Relationships (UML standard, language-agnostic)

| Relationship | Line | Arrow head | Direction |
| --- | --- | --- | --- |
| Inheritance (generalization) | solid | hollow triangle | child → parent |
| Realization | dashed | hollow triangle | implementer → interface |
| Association | solid | optional open arrow, role / multiplicity labels | as the business dictates |
| Aggregation | solid | hollow diamond on the whole side | part → whole |
| Composition | solid | filled diamond on the whole side | part → whole |
| Dependency | dashed | open arrow | dependent → supplier |

### Multiplicity

- Label both ends of associations, aggregations, and compositions when the count is meaningful.
- Use the standard set: `1`, `n`, `0..1`, `1..*`, `0..*`. Pick one notation per diagram and stay consistent.

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

- **Class box** — rectangle with a thin black border, divided by horizontal lines into three regions stacked top-to-bottom: name / attributes / methods.
- **Name region** — light grey fill, class name centered, bold or regular weight per the template; the attribute and method regions are white.
- **Member rows** — left-aligned, one per line, leading visibility marker (`+` / `-` / `#` / `~`) immediately followed by type and name (no extra space): `+Long userId`, `+login()`.
- **No card chrome** — no drop shadows, no rounded corners, no full-color body fills, no gradient backgrounds, no glyph icons. The reference is StarUML / Visio / ProcessOn UML exports, not a marketing slide.
- **Connectors** — black thin lines. **Solid** for inheritance, association, aggregation, composition. **Dashed** for realization, dependency, and `manage`-style control arrows.
- **Endpoint labels** — multiplicity numbers (`1`, `n`, `0..1`, `1..*`) sit at each line endpoint, close to the class box.
- **Mid-line labels** — relationship verbs (`manage`, `审核`, `分配`) sit on the middle of the dashed control line.
- **Layout flow** — top tier (management classes) at the top of the canvas, junction / record classes at the bottom; control arrows go top-down, association lines go vertically between the middle and bottom tiers.

When the board already has a template, follow the template's existing class-box style (modify-in-place rule from `boundaries.md`) — do not retheme it into a different visual vocabulary just to match the academic look.

### Naming style

- Class names: business nouns in PascalCase (`User`, `Admin`, `Course`, `Exam`, `Question`, `Message`, `Submission`).
- Attribute names: lowerCamelCase business fields (`userId`, `title`, `status`, `createTime`).
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

Build the class diagram out of these native whiteboard primitives. Do not express any part of the diagram as PlantUML, Mermaid, or SVG.

- **Class box** — native rectangle, divided into three stacked regions (class name on top, attributes in the middle, methods at the bottom). The board's existing class boxes already use a specific structure; clone one and overwrite its text rather than building a new shape.
- **Interface / abstract class box** — same native rectangle layout, with a `«interface»` or `«abstract»` stereotype on the name row (or whatever marker the template already uses).
- **Visibility markers** — keep `+` / `-` / `#` / `~` in front of each member.
- **Inheritance connector** — native `type: "connector"`, hollow-triangle arrow head, solid line, from child class id to parent class id.
- **Realization connector** — native connector, hollow-triangle arrow head, dashed line, from implementer id to interface id.
- **Association connector** — native connector, solid line, optional role / multiplicity labels on both ends.
- **Aggregation connector** — native connector, solid line, hollow diamond on the whole-class side.
- **Composition connector** — native connector, solid line, filled diamond on the whole-class side.
- **Dependency connector** — native connector, dashed line, open arrow head; use this for `manage` / `审核` / `分配` style control relationships and for transient usage dependencies.

For every relationship connector, `connector.from` and `connector.to` must be ids of real class / interface boxes that survive the edit. Clone the closest existing connector with the right arrow style before adjusting endpoints.
