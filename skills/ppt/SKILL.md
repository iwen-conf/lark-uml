---
name: lark-uml:ppt
description: 飞书原生演示文稿 / PPT：执行型 Skill。当用户要把项目代码、README、接口文档、测试数据或毕业设计材料生成或改写为飞书 Slides 演示文稿（答辩 PPT、项目汇报、技术方案、路演材料）时使用。本 skill 直接驱动 lark-cli slides / 飞书 Slides OpenAPI 创建和编辑云端演示文稿，不输出 PPTX、HTML、Markdown 或截图；严禁用 python-pptx / 本地 PPTX / 飞书导入作为替代方案；除非用户明确只要 payload，否则交付物必须是可编辑的飞书原生 Slides。
---

# `lark-uml:ppt`

Specialist skill for **native Feishu / Lark Slides presentations**. The agent turns project evidence into an editable cloud presentation through `lark-cli slides` and, only when needed, `lark-cli api` for uncovered Slides OpenAPI surfaces.

The final artifact is the created or updated Feishu Slides document. It is not a `.pptx`, Markdown outline, HTML prototype, screenshot deck, imported Office file, or pasted explanatory answer. A locally generated PPTX that is uploaded or imported into Feishu is still a forbidden PPTX route, not a native Slides success.

## Inputs

- `presentation` — existing Slides URL / `xml_presentation_id` when modifying an existing deck. Optional for first-time creation.
- `task` — what the deck should communicate: graduation defense, project report, technical proposal, demo script, investor-style pitch, etc.
- `source` — repository path, README, API docs, SQL, test reports, pasted code, screenshots, or user-provided facts. Required unless the task is only visual polish on an existing deck.
- `audience` — defense committee, technical reviewers, product team, client, investors. Default: `technical reviewers`.
- `language` — `zh-CN` (default) or `en-US`. Slide-visible text only.
- `page_count` — default `12-15` for graduation defense / project report. Keep fewer pages only when the user explicitly asks for a short deck.

## Native Slides hard contract

- **Native-only creation.** Create or edit decks only through `lark-cli slides` commands or official Feishu / Lark Slides OpenAPI called with `lark-cli api`.
- **No PPTX import fallback.** Never use `python-pptx`, PowerPoint, Keynote, LibreOffice, Office conversion tools, local `.pptx` generation, Drive upload, or Feishu document import as a workaround for Slides API / CLI friction.
- **No "more reliable PPTX" route.** Do not say or act on variants of "switch to a more reliable plan: generate a PPTX file, then import it into Feishu." That is an invalid route for this skill.
- **Block instead of degrading.** If native Slides creation is blocked by missing CLI support, schema uncertainty, permission, scopes, payload limits, or API errors, stop and report the exact blocker, attempted native command/API, and the next native-only fix. Do not create a substitute artifact.
- **Schema before XML.** Before writing any SML XML, read `references/slides-native.md` and check `lark-cli schema slides.xml_presentation.slide.create` / `slides.xml_presentation.slide.replace`. Do not invent XML elements from memory.

## Workflow

1. Load `lark-shared` for auth, identity, permission, and write-safety rules.
2. Read `references/slides-native.md` before generating or editing slide XML.
3. Prefer the registered `lark-cli slides` commands. Check `lark-cli slides --help` and command-specific `--help` before falling back to raw OpenAPI. A CLI failure is not permission to switch to PPTX/import tooling.
4. If the needed Slides operation is not registered, load `lark-openapi-explorer`, discover the official API, then call it with `lark-cli api`. Do not guess paths, request bodies, or scopes. If official native creation cannot be verified, stop as blocked.
5. Read the project evidence before writing content. Use local files, local indexes, and repository docs first.
6. Build a compact deck model in memory: slide goal, action title, evidence, visual metaphor, layout family, speaker takeaway.
7. Convert the model into SML 2.0 slide XML using the native reference: visible text must be inside `<data><shape type="text" ...><content ...><p>...</p></content></shape></data>`. Do not put `<title>`, `<body>`, or bare `<p>` directly under `<slide>`.
8. Preview write requests with `--dry-run` when practical, then create or update the cloud deck through native Feishu Slides operations. Do not stop at a local JSON, Markdown outline, HTML mock, image export, PPTX, or imported PPTX.
9. Read the deck back and verify slide count, page order, visible title text, and non-empty `<data>` for every intended page.

## Execution route

### Create a new deck

Use native creation only. For real decks, especially 12-15 page reports, complex Chinese text, images, or nested XML, default to the two-step route:

1. Create a blank native Slides document:

```bash
lark-cli slides +create \
  --as user \
  --title "项目毕业答辩"
```

2. Add each page with `xml_presentation.slide.create`:

```bash
lark-cli slides xml_presentation.slide create \
  --as user \
  --params '{"xml_presentation_id":"...","revision_id":-1}' \
  --data "$(jq -n --arg content '<slide xmlns="http://www.larkoffice.com/sml/2.0"><data><shape type="text" topLeftX="80" topLeftY="80" width="800" height="120"><content textType="title"><p>页面标题</p></content></shape></data></slide>' '{slide:{content:$content}}')"
```

Use `lark-cli slides +create --slides` only for simple decks with no more than 10 short, low-risk pages:

```bash
lark-cli slides +create \
  --as user \
  --title "项目毕业答辩" \
  --slides '@slides.json'
```

`slides.json` is a JSON array of complete `<slide>...</slide>` XML strings. `+create --slides` has a hard limit of 10 pages and is not atomic: the presentation and earlier pages may remain if a later page fails. For more than 10 pages, create a blank deck first, then add pages one by one with `xml_presentation.slide.create`.

If the installed CLI version does not accept `@file` for `--slides`, pass the same JSON array string directly or pipe through the command's documented input support after checking `lark-cli slides +create --help`. For complex XML, avoid long inline shell strings and use `jq -n --arg content "$SLIDE_XML" '{slide:{content:$content}}'` with `xml_presentation.slide.create`.

If `+create` cannot create the deck, try the official native route only: `xml_presentation.slide.create` or a verified Slides OpenAPI call through `lark-cli api`. If those native routes are unavailable, report the blocker and stop. Do not generate a PPTX and do not import one into Feishu.

### SML XML minimum shape

Every created page must use SML 2.0 structure that survives read-back:

```xml
<slide xmlns="http://www.larkoffice.com/sml/2.0">
  <style>
    <fill><fillColor color="rgb(248,250,252)"/></fill>
  </style>
  <data>
    <shape type="text" topLeftX="80" topLeftY="80" width="800" height="120">
      <content textType="title">
        <p>页面标题</p>
      </content>
    </shape>
    <shape type="text" topLeftX="80" topLeftY="200" width="800" height="180">
      <content textType="body">
        <p>正文内容</p>
      </content>
    </shape>
  </data>
</slide>
```

Rules:

- `<slide>` direct children are only `<style>`, `<data>`, and `<note>`.
- Text belongs in `<content>` inside a `shape`; never write bare `<p>`, `<text>`, `<title>`, or `<body>` directly under `<slide>`.
- Escape visible text before XML embedding: `&`, `<`, and `>`.
- For gradients, use `rgba()` stops with percentages; malformed gradients can silently become white.
- After every write, read back XML and treat empty `<data/>` as failure even if the API returned success.

### Modify an existing deck

1. Read full deck XML:

```bash
lark-cli slides xml_presentations get \
  --as user \
  --params '{"xml_presentation_id":"..."}'
```

2. Read individual slide XML when needed:

```bash
lark-cli slides xml_presentation.slide get \
  --as user \
  --params '{"xml_presentation_id":"...","slide_id":"..."}'
```

3. Replace or insert elements with optimistic locking:

```bash
lark-cli slides +replace-slide \
  --as user \
  --presentation "..." \
  --slide-id "..." \
  --revision-id -1 \
  --parts '@parts.json'
```

Use `str_replace` for text-only updates and `block_replace` / `block_insert` for structural changes. Keep each replacement batch under the CLI/API limit.

### Images and media

Use `lark-cli slides +media-upload` or `<img src="@./local.png">` placeholders supported by `+create --slides`. Do not embed base64 image blobs in slide XML or JSON.

When adding images through `xml_presentation.slide.create` or `+replace-slide`, do not use `@./local.png`. Upload first:

```bash
lark-cli slides +media-upload \
  --as user \
  --file ./local.png \
  --presentation "..."
```

Then use the returned `file_token` in `<img src="...">`.

Images are allowed and often useful, but every image must be replaceable:

- Prefer explicit image slots with stable labels such as `ARCHITECTURE_IMAGE_SLOT`, `DASHBOARD_SCREENSHOT_SLOT`, or `WORKFLOW_SCREENSHOT_SLOT`.
- Use project screenshots, architecture exports, UI captures, or neutral placeholder images only when they support the slide's claim.
- Never make the image the only carrier of critical information; keep the quantified takeaway as editable text.
- If the real image is not available, create an obvious replaceable placeholder with a short caption and recommended replacement type.

## Content rules

- **Action titles only.** Every slide title must state the conclusion or decision: `采用分层架构降低模块耦合`, not `系统架构`.
- **Young, minimal, data-driven.** Use short phrases, hard numbers, ratios, counts, latencies, coverage, throughput, cost, or timeline metrics. Keep 3-5 high-signal bullets and never paste long code blocks.
- **Evidence before claims.** Extract facts from code, README, API docs, SQL, tests, commits, or user-provided material. If a metric is not real, mark it as a placeholder with brackets: `接口 P95 延迟目标 [Y] ms`, `并发吞吐提升 [X] 倍`.
- **Numbers on every content slide.** Except for pure cover / closing pages, every slide needs at least one numeric expression: measured data, count, percentage, before/after delta, target, SLA, timeline, scale, or bracketed placeholder.
- **Lifecycle coverage.** For a 12-15 page defense/report deck, cover pain point, project goal, requirements, architecture, core workflow, technical highlights, data/storage model, API design, testing/performance, deployment/maintenance, risk control, roadmap, and closing value.
- **Code becomes concepts.** Convert classes, handlers, SQL, configs, and algorithms into diagrams, cards, metrics, and concise explanations. A slide should explain the engineering decision, not display source code.
- **One takeaway per page.** If a slide needs two unrelated conclusions, split it.

## Visual direction

The user may ask for inconsistent styles. Implement that as **intentional chapter-level variety**, not random layout drift.

- Keep a shared baseline: 16:9 canvas, consistent safe margins, readable type scale, and stable title hierarchy.
- Vary visual language by page or section: monochrome architecture sheet, neon metric card, blueprint system map, terminal-inspired technical highlight, editorial problem page, roadmap strip.
- Avoid default corporate sameness. Do not reuse one template on every page unless the user asks for uniformity.
- Prefer bold negative space, large numerals, sparse bullets, and simple geometric composition.
- Do not let decorative variety break scanability: no overlapping text, low contrast, tiny labels, or unsupported fonts.

## Graduation defense default outline

Use this structure when the user says "毕业答辩", "项目汇报", or provides code without a deck outline:

| Page | Action-title pattern | Content focus |
| --- | --- | --- |
| 1 | `用 [项目名] 解决 [核心痛点]` | Title, role, one-line value proposition |
| 2 | `[现状数据] 暴露了 [痛点]` | Pain points with measurable placeholders if needed |
| 3 | `目标被收敛为 [3-5] 个可验证指标` | Scope, success criteria, measurable outcomes |
| 4 | `[N] 类用户场景决定系统边界` | Requirements, actors, primary use cases |
| 5 | `分层架构把 [复杂度] 隔离到清晰边界` | Architecture diagram, module responsibilities, replaceable architecture image slot |
| 6 | `核心流程以 [关键机制] 保证业务闭环` | Main workflow, state transitions, replaceable workflow image slot |
| 7 | `[技术亮点 1] 将 [指标] 优化到 [X]` | Key algorithm, concurrency, caching, auth, or reliability design |
| 8 | `[技术亮点 2] 让接口协作保持可维护` | API design, contract shape, error handling, versioning |
| 9 | `数据模型支撑 [N] 个关键业务关系` | ER/class-level summary, not raw DDL dumps |
| 10 | `测试覆盖验证 [核心假设]` | Unit/integration/e2e coverage, defect count, pass rate |
| 11 | `性能结果证明 [瓶颈] 已被控制` | Latency, QPS, throughput, resource usage, before/after placeholders |
| 12 | `部署方案把交付风险压缩到 [N] 个步骤` | Deployment, config, environment, rollback |
| 13 | `运维策略让故障定位控制在 [T] 分钟内` | Logs, monitoring, alerting, recovery, security |
| 14 | `路线图按 [3] 个阶段扩展系统能力` | Roadmap, maintainability, scalability risks |
| 15 | `项目价值最终落在 [业务/教学/工程] 指标` | Closing value, summary metrics, defense Q&A hook |

## Payload-only mode

If the user explicitly asks only for a payload instead of executing `lark-cli`, output a single pure JSON code block and nothing else. The payload must target the currently verified CLI/API surface:

- For `lark-cli slides +create`: JSON array of slide XML strings.
- For `xml_presentation.slide.create`: request body containing `slide.content`.
- For `xml_presentation.slide.replace`: request body containing `parts`.

Do not claim that Feishu accepts a generic, model-invented page coordinate JSON. Use official schema fields verified by `lark-cli schema` or OpenAPI docs.

## Quality gates

Before reporting completion:

- Verify `lark-cli` created or updated a real Slides document and capture the URL / id from the response.
- Re-read the deck XML and confirm every intended slide exists.
- Confirm every intended slide has non-empty `<data>` and at least one visible native element such as `shape`, `img`, `line`, `table`, `chart`, or `icon`.
- Confirm intended title/body text appears in the read-back XML, not only in the local payload.
- Confirm all titles are action titles, not neutral labels.
- Confirm the deck has 12-15 pages unless the user requested a different count.
- Confirm every non-cover/non-closing content slide has at least one numeric expression or bracketed numeric placeholder.
- Confirm placeholder metrics are bracketed and not presented as measured facts.
- Confirm media references resolve through uploaded file tokens or supported local image placeholders.
- Confirm replaceable image slots are labeled and critical claims remain editable as text.
- Confirm no deliverable is a PPTX, imported PPTX, HTML page, Markdown outline, screenshot-only deck, or unsupported hand-written API fantasy.
- Confirm no `python-pptx`, Office export/conversion tool, Drive upload/import, or Feishu import route was used as a fallback.

## Failure modes

- **Deep JSON hallucination.** The model invents nested coordinate objects that do not match Slides API. Fix by using `lark-cli slides` XML or official schema-derived payloads.
- **Invalid SML structure.** The API returns success but read-back shows `<data/>` or missing text because XML used `<text>`, bare `<p>`, `<title>`, or `<body>` in the wrong place. Fix by using `<data><shape type="text" ...><content><p>...</p></content></shape></data>`.
- **Partial native create.** `+create --slides` fails after creating the deck or some pages. Fix by recording `xml_presentation_id`, reading the deck back, then continuing with `xml_presentation.slide.create`; delete duplicates only after confirming actual slide ids.
- **Template monotony.** Every page looks the same despite the user asking for varied style. Fix by assigning a visual family per slide while keeping typography and margins readable.
- **Fake metrics.** Unmeasured performance claims appear as facts. Fix by replacing them with bracketed placeholders or source-backed numbers.
- **Code dump slides.** Long source blocks consume the page. Fix by extracting the engineering decision, API contract, state transition, or metric.
- **Unverified write.** The command returns without checking the cloud result. Fix by reading the deck back before final response.
- **PPTX import fallback.** The agent proposes or creates a local PPTX with `python-pptx` or another Office tool and imports it into Feishu. This is forbidden. Fix by returning to `lark-cli slides` / official Slides OpenAPI, or stop with a native-route blocker if those APIs cannot complete the write.
