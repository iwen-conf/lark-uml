# Core boundaries

Every `lark-uml:*` skill inherits these. They are non-negotiable.

## 1. One whiteboard per invocation

- Operate on exactly one whiteboard token per run. Do not chain multiple boards in a single skill execution.
- If the user mentions several boards, ask which one to work on this turn.

## 2. One diagram type per whiteboard

- Each `lark-uml:*` skill is bound to a single diagram type. Do not mix swimlane, sequence, class, etc. on the same board.
- If the board currently holds a different diagram type than the skill the user invoked, surface the mismatch before overwriting.

## 3. The agent writes the board itself

- The deliverable is the **updated whiteboard**, not Mermaid / PlantUML / SVG handed back to the user.
- Do not paste raw `mermaid` / `plantuml` / `dot` code blocks into the chat as the final result.
- Do not output "action plan JSON" for someone else to apply.
- The agent reads, edits, and writes back through `lark-cli whiteboard` (`+query` / `+update`).

## 4. Overwrite policy

- When the user has already authorized `--overwrite` (explicitly or via a memory rule), overwrite the board without a second confirmation.
- When in doubt about destructive scope (e.g., wiping a complex existing diagram), `+query --output_as raw` first, summarize what is on the board, then proceed.

## 5. Scope guard

- Only touch the requested change. Do not "improve" unrelated regions, restyle nodes the user did not mention, or refactor naming on a whim.
- A first-time initialization (no prior content) is the only case where the agent designs the whole board from scratch.
