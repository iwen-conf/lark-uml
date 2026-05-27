# Native Feishu Slides Reference

This reference is the local `lark-uml:ppt` guardrail for native Slides creation. It is distilled from the installed `lark-cli` schema and the official `larksuite/cli` `lark-slides` skill. Use it before emitting any Slides XML.

## Non-Negotiable Route

- Use only `lark-cli slides` or official `/open-apis/slides_ai/v1` APIs.
- Never use `python-pptx`, local `.pptx`, Office conversion, Drive upload/import, or Feishu import as a fallback.
- If native APIs fail, read back the deck state, fix the native XML/API call, or stop with the exact native blocker.

## Service And Schemas

- `lark-cli schema slides.xml_presentation.slide.create`
- `lark-cli schema slides.xml_presentation.slide.replace`
- `lark-cli schema slides.xml_presentation.slide.get`
- `lark-cli schema slides.xml_presentations.get`
- API base path: `/open-apis/slides_ai/v1`
- Create page: `POST /open-apis/slides_ai/v1/xml_presentations/{xml_presentation_id}/slide`
- Replace page parts: `POST /open-apis/slides_ai/v1/xml_presentations/{xml_presentation_id}/slide/replace`
- Read full deck: `GET /open-apis/slides_ai/v1/xml_presentations/{xml_presentation_id}`

## SML 2.0 Minimum Rules

- Namespace: `http://www.larkoffice.com/sml/2.0`
- Standard slide root: `<slide xmlns="http://www.larkoffice.com/sml/2.0">...</slide>`
- `<slide>` direct children are only `<style>`, `<data>`, and `<note>`.
- Visible page elements must be inside `<data>`.
- Text is not written directly under `<slide>`. Use `<shape type="text" ...><content ...><p>...</p></content></shape>`.
- A text/shape element needs coordinates and size: `topLeftX`, `topLeftY`, `width`, `height`.
- Escape text before embedding XML: `&` to `&amp;`, `<` to `&lt;`, `>` to `&gt;`.
- `shape` elements used in `+replace-slide` should contain `<content/>` or a real `<content>...</content>` child. The shortcut auto-injects missing `<content/>`; raw API calls do not provide the same safety.

Minimum visible slide:

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

## Native Creation Strategy

Use two-step creation by default for real decks:

1. Create a blank native Slides document:

```bash
lark-cli slides +create --as user --title "项目汇报"
```

2. Add each slide with `xml_presentation.slide.create` and `jq` JSON wrapping:

```bash
lark-cli slides xml_presentation.slide create --as user \
  --params '{"xml_presentation_id":"YOUR_ID","revision_id":-1}' \
  --data "$(jq -n --arg content '<slide xmlns="http://www.larkoffice.com/sml/2.0"><data><shape type="text" topLeftX="80" topLeftY="80" width="800" height="120"><content textType="title"><p>标题</p></content></shape></data></slide>' '{slide:{content:$content}}')"
```

Use `slides +create --slides '[...]'` only for simple decks with 10 or fewer short, low-risk pages. It still creates the presentation first and then adds pages; it is not atomic. For more than 10 pages, or any deck with complex Chinese text, nested XML, images, long bullets, or special characters, use the two-step route.

When inserting before another page, put `before_slide_id` in `--data`, not `--params`:

```bash
jq -n --arg content "$SLIDE_XML" --arg before "$TARGET_SLIDE_ID" \
  '{slide:{content:$content}, before_slide_id:$before}'
```

## Images

- In `slides +create --slides`, `<img src="@./local.png" .../>` is allowed and the shortcut uploads the file.
- In `xml_presentation.slide.create` or `+replace-slide`, `@./local.png` is not resolved. Upload first:

```bash
TOKEN=$(lark-cli slides +media-upload --as user \
  --file ./pic.png --presentation "$PRES_ID" | jq -r '.data.file_token')
```

Then write:

```xml
<img src="FILE_TOKEN" topLeftX="600" topLeftY="120" width="240" height="135"/>
```

- Do not use `http(s)` image URLs in SML; the Slides renderer does not reliably proxy them.
- The path for `@./local.png` must be relative to the current working directory and inside it.

## Replace Existing Slides

Prefer the shortcut for local element edits:

```bash
lark-cli slides +replace-slide --as user \
  --presentation "$PRES_ID" \
  --slide-id "$SLIDE_ID" \
  --parts '[{"action":"block_insert","insertion":"<shape type=\"rect\" topLeftX=\"500\" topLeftY=\"100\" width=\"200\" height=\"100\"><content/></shape>"}]'
```

Allowed `parts` actions for the shortcut are `block_replace` and `block_insert`. Do not rely on raw `str_replace`.

Useful minimal roots for `insertion` / `replacement`:

- `<shape type="text" topLeftX="80" topLeftY="80" width="800" height="120"><content textType="title"><p>标题</p></content></shape>`
- `<shape type="rect" topLeftX="120" topLeftY="120" width="240" height="120"><fill><fillColor color="rgb(100,149,237)"/></fill><content/></shape>`
- `<line startX="120" startY="120" endX="420" endY="120"><border color="rgb(43,47,54)" width="2"/></line>`
- `<img src="FILE_TOKEN" topLeftX="80" topLeftY="120" width="320" height="180"/>`

For `block_replace`, first read the page and use the current three-character block id from the returned XML. The shortcut injects the correct root `id` for `block_replace`.

## Failure Handling

If a create/append/replace command reports failure, do not assume no write happened:

1. Record `xml_presentation_id`.
2. Run `xml_presentations.get`.
3. Count existing `<slide>` elements and inspect every `<data>`.
4. If there are duplicates from retry, delete only the extra slide ids after reading the deck state.
5. Fix XML structure, escaping, image tokens, or request wrapping, then continue with native APIs.

Treat these as broken output:

- `<data/>` is empty.
- The expected title/body text is absent from the read-back XML.
- A slide contains only background or empty `<content/>`.
- Images still use `@./path` after creation, or use `http(s)` URLs.
- Shell quoting likely truncated the XML.

Common fixes:

- Empty page after "success": SML structure is wrong or XML was stripped; use the minimum visible slide pattern above.
- Complex inline `--slides` fails or partially succeeds: switch to two-step blank create plus `slide.create`.
- `block_insert requires non-empty insertion`: use `+replace-slide`, pass `--parts @parts.json` or stdin, and ensure `insertion` is a non-empty single XML element string.
- `3350001`: check XML escaping, direct children of `<slide>`, missing `slide.content` wrapper, stale `block_id`, or invalid root element.

## Verification

Before reporting completion:

```bash
lark-cli slides xml_presentations get --as user \
  --params '{"xml_presentation_id":"YOUR_ID"}'
```

Verify:

- Actual page count equals plan.
- Every page has non-empty `<data>` with visible `shape`, `img`, `line`, `table`, `chart`, or `icon` elements.
- Each intended title appears in the read-back XML.
- No page is placeholder-only.
- Images use file tokens, not unresolved local placeholders or URLs.
- The final artifact is the native Slides URL/id from `slides +create`, not a converted Office file.
