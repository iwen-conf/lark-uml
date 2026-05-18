# lark-uml

Feishu / Lark whiteboard UML skills, packaged as a Claude Code plugin. One specialist skill per diagram type, all of them drive the [`lark-whiteboard`](https://github.com/larksuite) tooling via `lark-cli` so the agent reads native raw, edits native nodes / connectors, and writes raw back itself. The deliverable is an editable whiteboard with structurally bound connectors, not Mermaid / PlantUML / SVG handed back to the user.

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

Each skill is scoped to a single diagram type. The skill carries the layout discipline (alignment, node shapes, connector semantics, forbidden patterns) for that type. The agent stays inside one whiteboard per invocation and one diagram type per whiteboard.

## Native Connector Rule

`lark-uml` is raw-first. For any diagram where nodes must remain editable and connectors must follow moved nodes, business relationships must be native whiteboard `connector` nodes with `connector.from` / `connector.to` bound to existing node ids.

Coordinate endpoints are allowed only for annotation, decoration, measurement, axes, or other non-business helper marks. `waypoints` may shape a route but never replace endpoint binding. "Looks connected" is not a success criterion; success means structural binding, valid anchors, and drag-follow semantics.

## How it works

1. The user gives a whiteboard URL / token and an optional change request.
2. The matching skill loads its diagram-specific quality rules.
3. The skill delegates the actual raw read / write to `lark-whiteboard` (`lark-cli whiteboard +query --output_as raw` / `+update --input_format raw`).
4. Output is the updated whiteboard, not a code blob.

`lark-cli` must be installed and authenticated. See the [`lark-whiteboard`](https://github.com/larksuite) and [`lark-shared`](https://github.com/larksuite) skills.

## Requirements

- `lark-cli` available on PATH and logged in.
- Claude Code with the `lark-whiteboard` and `lark-shared` skills enabled.
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
    └── network/SKILL.md
```

## License

MIT
