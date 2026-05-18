# Execution workflow

All `lark-uml:*` skills follow this loop. Diagram-specific quality rules live inside each `SKILL.md`.

## Pre-flight

1. Load [`lark-shared`](../../lark-shared/SKILL.md) (auth, scopes, `--as` identity).
2. Load [`lark-whiteboard`](../../lark-whiteboard/SKILL.md) (the actual `+query` / `+update` surface).
3. Confirm `lark-cli` is reachable: `lark-cli --version`.

## Step 1 — Resolve the board token

| What the user gave you | How to get the `board_token` |
| --- | --- |
| Whiteboard token (`wbcnXXX`) | Use directly. |
| Whiteboard URL | Extract the token from the URL. |
| Docx URL / `doc_id` with an existing whiteboard block | `lark-cli docs +fetch --api-version v2 --doc <URL> --as user`, read `<whiteboard token="..."/>` from the response. |
| Docx URL / `doc_id` without a whiteboard block | `lark-cli docs +update --api-version v2 --doc <doc_id> --command append --content '<whiteboard type="blank"></whiteboard>' --as user`, read `data.new_blocks[0].block_token` from the response (the entry whose `block_type == "whiteboard"`). |

## Step 2 — Read current state (only if there is existing content)

```bash
lark-cli whiteboard +query <board_token> --output_as code --as user
```

- Use `--output_as code` to see whether the board is already driven by Mermaid / PlantUML.
- Use `--output_as raw` only when you must inspect or edit native node JSON.
- Skip Step 2 for a clean board.

## Step 3 — Generate the diagram source

- Each `lark-uml:*` skill specifies the preferred source format for that diagram type (usually **PlantUML** for layout-strict types — sequence / usecase / class / ER — and **Mermaid** for free-form types — flowchart / swimlane / architecture / network).
- Apply the diagram-specific quality rules from this skill's `SKILL.md`.
- Apply the language rules from [`language.md`](./language.md).
- Apply the core boundaries from [`boundaries.md`](./boundaries.md).

## Step 4 — Write the board

Always pipe through stdin. The `lark-whiteboard` skill mandates this for any source loaded from a local file or generated content:

```bash
cat diagram.plantuml | lark-cli whiteboard +update <board_token> \
  --source - \
  --input_format plantuml \
  --overwrite \
  --as user
```

Format flag values:

- `plantuml` — for `.puml` / `.plantuml` payloads
- `mermaid` — for `.mmd` / Mermaid payloads
- `raw` — for native OpenAPI node JSON

## Step 5 — Verify

- Re-run `+query --output_as image` and confirm the layout matches the diagram-type rules.
- For layout-strict diagrams (sequence, usecase), specifically verify alignment / lifelines / boundaries before reporting done.

## Failure modes to watch for

- Permission denied → check `lark-shared` for scope guidance.
- Token resolves but the block is not a whiteboard → re-confirm with the user.
- `+update` succeeds but the board renders empty → re-check the source format flag and stdin piping.
