# lark-uml

Feishu Whiteboard 原生操作 Agent for UML-style diagrams.

This plugin is intentionally strict: it modifies Feishu / Lark whiteboard templates through native `lark-cli whiteboard` raw nodes and native connectors. It must not draw by Mermaid, PlantUML, SVG, Graphviz, draw.io, screenshots, or any converted diagram source. The agent reads the board, edits the existing native template in place, and writes the board back.

Feishu / Lark whiteboard UML and native Slides skills, packaged as a Claude Code plugin. Diagram skills drive the [`lark-whiteboard`](https://github.com/larksuite) tooling via `lark-cli` so the agent reads native raw, edits native nodes / connectors, and writes raw back itself. The Slides skill drives `lark-cli slides` / Slides OpenAPI so the agent creates or edits a cloud presentation directly. The deliverable is an editable Feishu artifact, not Mermaid / PlantUML / SVG / PPTX / HTML handed back to the user.

## Diagram Hard Contract

All diagram skills inherit [`references/native-agent.md`](references/native-agent.md):

- Use `lark-cli whiteboard +query --output_as raw` before editing.
- Modify the current template in place by renaming, moving, reconnecting, regrouping, splitting, merging, adding, or deleting native elements.
- Duplicate existing same-kind template nodes / connectors when new elements are needed.
- Preserve visual vocabulary and useful node ids instead of redrawing the whole board.
- Write back only through `lark-cli whiteboard +update --input_format raw`.
- Verify business connectors are native `type: "connector"` nodes with `connector.from` / `connector.to` bound to existing node ids.
- Treat Mermaid / PlantUML / SVG / DOT / draw.io / screenshot conversion as a failure path, including private sketches.

## Skills

| Skill | Diagram |
| --- | --- |
| `lark-uml:swimlane` | 泳道图 |
| `lark-uml:usecase` | 用例图 |
| `lark-uml:class` | 系统主要类图 |
| `lark-uml:er` | 数据库关系图 |
| `lark-uml:flowchart` | 流程图 |
| `lark-uml:sequence` | 时序图 |
| `lark-uml:architecture` | 架构图 |
| `lark-uml:network` | 网络拓扑图 |
| `lark-uml:ppt` | 飞书原生演示文稿 / PPT |

Each diagram skill is scoped to a single diagram type. The skill carries the layout discipline (alignment, node shapes, connector semantics, forbidden patterns) for that type. The agent stays inside one whiteboard per invocation and one diagram type per whiteboard.

`lark-uml:ppt` is scoped to editable Feishu Slides. It turns project evidence into a young, minimal, data-driven deck by using `lark-cli slides` and official Slides OpenAPI payloads. It does not create local `.pptx` files, HTML prototypes, screenshot decks, or hallucinated coordinate JSON.

## Native Connector Rule

`lark-uml` is raw-first. For any diagram where nodes must remain editable and connectors must follow moved nodes, business relationships must be native whiteboard `connector` nodes with `connector.from` / `connector.to` bound to existing node ids.

Coordinate endpoints are allowed only for annotation, decoration, measurement, axes, or other non-business helper marks. `waypoints` may shape a route but never replace endpoint binding. "Looks connected" is not a success criterion; success means structural binding, valid anchors, and drag-follow semantics.

## How it works

1. The user gives a whiteboard URL / token and an optional change request.
2. The matching skill loads its diagram-specific quality rules.
3. Diagram skills delegate the actual raw read / write to `lark-whiteboard` (`lark-cli whiteboard +query --output_as raw` / `+update --input_format raw`).
4. The Slides skill delegates presentation creation / editing to `lark-cli slides` (`+create`, `+replace-slide`, or `xml_presentation.slide.*`), falling back to official OpenAPI only when needed.
5. Output is the updated Feishu artifact, not a code blob.

`lark-cli` must be installed and authenticated. See the [`lark-whiteboard`](https://github.com/larksuite) and [`lark-shared`](https://github.com/larksuite) skills.

## Requirements

- `lark-cli` available on PATH and logged in.
- Claude Code with the `lark-whiteboard`, `lark-shared`, and Slides-capable `lark-cli` skills enabled.
- A Feishu / Lark account with edit permission on the target whiteboard.

## Install (manual)

```bash
git clone https://github.com/iwen-conf/lark-uml.git ~/.claude/plugins/lark-uml
```

Then point your Claude Code plugin loader at `~/.claude/plugins/lark-uml`. The skills register under the `lark-uml:` namespace.

## Layout

```
lark-uml/
├── .claude-plugin/plugin.json
├── references/                     # shared workflow & language rules
│   ├── workflow.md
│   ├── native-agent.md
│   ├── language.md
│   ├── boundaries.md
│   └── connectors.md
└── skills/
    ├── swimlane/SKILL.md
    ├── usecase/SKILL.md
    ├── class/SKILL.md
    ├── er/SKILL.md
    ├── flowchart/SKILL.md
    ├── sequence/SKILL.md
    ├── architecture/SKILL.md
    ├── network/SKILL.md
    └── ppt/SKILL.md
```

## License

MIT
