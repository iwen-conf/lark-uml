# Lessons

- When working on `lark-uml`, never use Mermaid, PlantUML, SVG, DOT, draw.io, screenshots, or other diagram DSL routes, even as private sketches. The required route is `lark-cli whiteboard +query --output_as raw`, modify existing native template nodes / connectors in place, then `+update --input_format raw` and verify native `connector.from` / `connector.to` bindings.
