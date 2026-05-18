# Diagram language rules

Every `lark-uml:*` skill accepts a `language` input. Default is **`zh-CN`** unless the user states otherwise.

## `zh-CN` (Chinese)

- All human-visible text on the diagram must be in Chinese: titles, group names, node labels, comments, legends, edge labels, status strings.
- Decision branch labels must be natural Chinese: 是 / 否, 通过 / 不通过, 成功 / 失败, 已完成 / 未完成. Do not keep `yes` / `no`, `true` / `false`, `if` / `else`.
- Do not paste Mermaid / PlantUML / UML keywords, source code snippets, JSON field names, function names, or database column names directly onto the canvas. Rewrite them as business-level Chinese.
- Product names, protocol names, interface names, and standard abbreviations may stay in their original form; surrounding explanatory text still must be Chinese.
- If the existing board contains text in another language inside the region you are about to touch, translate it to Chinese as part of this edit.

## `en-US` (English)

- All human-visible text on the diagram must be in English.
- Decision branch labels must be natural English: Yes / No, Approved / Rejected, Success / Failure, Done / Pending. Do not mix Chinese.
- Do not paste Mermaid / PlantUML / UML keywords, source code, JSON fields, function names, or database columns directly. Rewrite as business-level English.
- API, HTTP, OAuth, Kafka, SKU, etc. may stay in their original form; explanatory text still must be English.
- If the existing board contains text in another language inside the region you are about to touch, translate it to English as part of this edit.
