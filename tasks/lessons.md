# Lessons

- When working on `lark-uml`, never use Mermaid, PlantUML, SVG, DOT, draw.io, screenshots, or other diagram DSL routes, even as private sketches. The required route is `lark-cli whiteboard +query --output_as raw`, modify existing native template nodes / connectors in place, then `+update --input_format raw` and verify native `connector.from` / `connector.to` bindings.
- For flowchart / activity diagram work, treat `elseif` and `else if` as failure signals. Native diagrams need decision diamonds with labeled connectors; if a forced external PlantUML compatibility path ever appears, rewrite multi-way branches as nested `if` / `else` / `endif` before rendering or writing.
