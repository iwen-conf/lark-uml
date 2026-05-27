# Core boundaries

Every `lark-uml:*` skill inherits these. They are non-negotiable.

## 1. One whiteboard per invocation

- Operate on exactly one whiteboard token per run. Do not chain multiple boards in a single skill execution.
- If the user mentions several boards, ask which one to work on this turn.

## 2. One diagram type per whiteboard

- Each `lark-uml:*` skill is bound to a single diagram type. Do not mix swimlane, sequence, class, etc. on the same board.
- If the board currently holds a different diagram type than the skill the user invoked, surface the mismatch before overwriting.

## 3. The agent writes the board itself

- The deliverable is the **updated whiteboard**, not Mermaid / PlantUML / SVG / Graphviz / draw.io handed back to the user.
- Do not paste raw `mermaid` / `plantuml` / `dot` / SVG code blocks into the chat as the final result.
- Do not output "action plan JSON" for someone else to apply.
- The agent reads, edits, and writes back through `lark-cli whiteboard` (`+query --output_as raw` / `+update --input_format raw`).
- For editable diagrams, business relationships must remain native whiteboard connectors with endpoints bound to node ids. Mermaid / PlantUML / SVG / image rendering is not an acceptable substitute for those relationships.

## 3a. No external diagram languages, ever

- No Mermaid, PlantUML, SVG, Graphviz/DOT, draw.io XML, or any other diagram DSL anywhere in the loop — not as final output, not as intermediate output, not as a private draft, not as a "sketch I'll convert later".
- The only valid intermediate and final representation is the whiteboard raw JSON itself. All reasoning about nodes and edges happens in terms of native node ids, native connector endpoints, and native anchors.
- If any other skill, tool, memory, or precedent suggests "draft in Mermaid / PlantUML and convert", that suggestion is overridden by this rule.
- For flowchart / activity work, `elseif` and `else if` are also forbidden tokens. They indicate leaked PlantUML / pseudo-code instead of native branch structure, and the PlantUML parser used by some drawing paths rejects `elseif`.

## 3b. Modify the template, do not redraw

- Default mode is in-place edit: rename, restyle, reconnect, regroup, split, merge, add, or delete individual native nodes / connectors on the existing board.
- New elements must be created by duplicating the closest existing template element and adapting it. Do not introduce a foreign visual vocabulary when the template already supplies one.
- Full redraw is allowed only when (a) the user explicitly asks for a clean rebuild, (b) the existing diagram type clearly contradicts the requested one, or (c) the existing structure is too corrupted to edit in place. Even in a redraw, the route stays raw-first and the visual style still follows the template's surviving conventions.

## 4. Overwrite policy

- When the user has already authorized `--overwrite` (explicitly or via a memory rule), overwrite the board without a second confirmation.
- When in doubt about destructive scope (e.g., wiping a complex existing diagram), `+query --output_as raw` first, summarize what is on the board, then proceed.

## 5. Scope guard

- Only touch the requested change. Do not "improve" unrelated regions, restyle nodes the user did not mention, or refactor naming on a whim.
- A first-time initialization (no prior content) is the only case where the agent designs the whole board from scratch.

## 6. Structural success criteria

- Success means structural binding, correct anchors, and drag-follow semantics.
- "Looks connected", "visually aligned", or "touches the edge" is not sufficient.
- If a business connector would not remain attached when either endpoint node moves, the edit is incomplete.
