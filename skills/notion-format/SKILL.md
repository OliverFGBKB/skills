---
name: notion-format
description: Write Notion pages that look composed instead of dumped — monochrome callouts, unnumbered headings, tables and toggles instead of walls of prose — plus the block-JSON recipes and the API limits that silently ruin a page. Use when creating or restructuring a Notion page, publishing a doc or report into Notion, converting markdown into Notion blocks, or when a page you just wrote came out looking like pasted plain text.
---

Compose a Notion page as **blocks chosen on purpose**, not as paragraphs of markdown pushed through an API.

This skill is the style and the block vocabulary. It is deliberately transport-agnostic: whether you write through the Notion MCP server, the REST API, or your own wrapper, the page that comes out is what this governs.

## Transport

Two things must be true before writing: you can create a page, and you can append blocks to it.

**REST** — the ground truth, and the fallback when a tool is missing:

```
POST  /v1/pages                      {"parent": …, "properties": …, "children": [ … ]}
PATCH /v1/blocks/{block_id}/children {"children": [ … ]}
GET   /v1/blocks/{block_id}/children
```

Every request needs `Authorization: Bearer <token>` **and** a `Notion-Version` header — the API rejects the call without it. `PATCH …/children` returns `{"results": [ … ]}` holding the newly created **top-level** blocks with their ids; ids of nested children are not in that response, so fetch them with `GET …/children` when you need to reach deeper.

**MCP** — if a Notion MCP server is connected, use its create/update tools instead; tool names differ per server, so read the tool list rather than guessing. Note that some MCP servers expose page creation but not block append. If append is missing, you cannot restructure an existing page — say so instead of overwriting it.

**Access, not ids.** A 404 on a page that plainly exists almost always means the integration was never shared with it. Tell the user to open the page → `···` → Connections → add the integration. Child pages inherit it.

## House style

Six rules. They are what separates a composed page from a dumped one.

1. **No numbering in headings.** Never `## 1. Findings`, `## Part One`, `## 一、结论`. The heading level already carries the hierarchy; a hand-typed ordinal is only correct until a section moves, and Notion has no auto-numbering to repair it.
2. **Monochrome icons.** Callout icons carry structure, not decoration — a page of coloured emoji reads as a toy. Use the black-and-white set: ◻️ ◼️ ▪️ ▫️ ⬜ ⬛ ⚪ ⚫ ✔️ ✖️. One glyph per role, used consistently down the page.
3. **Restrained surface.** Default to no background colour. Reserve callouts for the conclusion and for risk — a page with five callouts has none. At most one highlighted number per table.
4. **Structure over prose.** Numbers → `table`. Definitions and methodology → toggleable `heading_3`. Open decisions → `to_do`. Parallel comparison → `column_list`. Long caveats → `toggle`. Keep the top of the page to one conclusion callout plus one table; a reader should get the answer before scrolling.
5. **Space CJK against latin** (中文 / 日本語 / 한국어 pages only). One space between CJK and adjacent latin letters, digits, and `$`/`%`: `用 notion CLI 把 Q3 毛利率 41.2% 写进 Notion`. Never around full-width punctuation — `硬件（Hardware）双线`, not `硬件 （Hardware） 双线`. Never inside code spans, URLs, or equations.
6. **Read the page back after writing.** A write that returns `200` is not a page that came out right. Fetch it and look: unparsed markers, a heading rendered as literal `####`, a callout that swallowed the four blocks below it. Every mistake in this file's history was found this way and none of them raised an error.

## Structure thresholds

Everything above can be decided one block at a time. These three cannot — they only
become visible once the whole page exists, which is why a page can satisfy every rule
above and still read as a flat list of twenty headings. Check them on a full re-read.

- **A heading owning fewer than two blocks loses the heading.** Merge the content into
  the section above. A heading sitting over a single code block or a single table is a
  label on something that already speaks for itself, and it costs a line in the outline.
- **More than six `heading_3` under one `heading_2` means folding is overdue.** Set
  `is_toggleable` on the ones a reader skips while following the main path — error
  lookups, platform caveats, permission workarounds, optional routes, appendices. The
  steps someone executes in order stay open. The test is not "is this important" but
  "does the reader walk past it the first time through".
- **Two tables or two code blocks in a row need a sentence between them**, stating how
  the second differs from the first. Without it the reader has to infer the relationship,
  and back-to-back tables get read as one table with a strange break in it.

A section that grows past roughly twenty blocks is no longer a subsection. Promote it to
`heading_2`, or split it — do not leave a `heading_3` carrying more content than the three
sections above it combined.

## Block recipes

Minimal, valid, and enough to compose any page. Every block goes in a `children` array.

**Callout** — the conclusion, and nothing else at the top of a page:

```json
{"type": "callout", "callout": {
  "rich_text": [{"type": "text", "text": {"content": "Margin recovered to 41.2%, up 1.8pt QoQ"}}],
  "icon": {"type": "emoji", "emoji": "◼️"},
  "color": "gray_background"}}
```

**Heading** — three levels, and the third can fold:

```json
{"type": "heading_3", "heading_3": {
  "rich_text": [{"type": "text", "text": {"content": "Definitions and assumptions"}}],
  "is_toggleable": true,
  "children": [ … ]}}
```

**Table** — `table_row` children are mandatory, and `table_width` must match every row:

```json
{"type": "table", "table": {
  "table_width": 2, "has_column_header": true, "has_row_header": false,
  "children": [
    {"type": "table_row", "table_row": {"cells": [
      [{"type": "text", "text": {"content": "Metric"}}],
      [{"type": "text", "text": {"content": "Q3"}}]]}}]}}
```

Each cell is its own `rich_text` **array** — a common mistake is passing a bare string.

**Columns** — at least two, each holding children:

```json
{"type": "column_list", "column_list": {"children": [
  {"type": "column", "column": {"children": [ … ]}},
  {"type": "column", "column": {"children": [ … ]}}]}}
```

**Toggle, to-do, code, equation, bookmark, divider, table of contents:**

```json
{"type": "toggle", "toggle": {"rich_text": [ … ], "children": [ … ]}}
{"type": "to_do", "to_do": {"rich_text": [ … ], "checked": false, "children": [ … ]}}
{"type": "code", "code": {"rich_text": [ … ], "language": "python"}}
{"type": "equation", "equation": {"expression": "1 - \\frac{COGS}{Revenue}"}}
{"type": "bookmark", "bookmark": {"url": "https://example.com"}}
{"type": "divider", "divider": {}}
{"type": "table_of_contents", "table_of_contents": {}}
```

**Rich text** — annotations and links live on the span, not the block:

```json
[{"type": "text", "text": {"content": "gross margin "}},
 {"type": "text", "text": {"content": "41.2%"}, "annotations": {"color": "yellow_background"}},
 {"type": "text", "text": {"content": "see basis", "link": {"url": "https://example.com"}}},
 {"type": "equation", "equation": {"expression": "x^2"}}]
```

Available annotations: `bold`, `italic`, `strikethrough`, `underline`, `code`, and `color` (nine text colours, nine `*_background`).

**Images** are external URLs only — `{"type": "image", "image": {"type": "external", "external": {"url": "https://…"}}}`. The block API cannot upload a local file; do not try to inline one.

## Hard limits

Verified against the live API. Each one fails quietly or confusingly, which is why they are here.

| Limit | Consequence |
| --- | --- |
| **`icon.emoji` accepts only real emoji** | `◆ ▸ ● ※ ❖ ▍`, and text-variation forms like `⚠︎`, all return `400 validation_error`. Emoji that are themselves black-and-white (◻️ ✔️ ⚫) pass. For a true grey glyph use `{"type": "external", "external": {"url": "…"}}`, which is accepted. |
| **100 blocks per request** | Chunk longer bodies and append the rest in follow-up calls. |
| **Two levels of nesting per request** | One request carries a block, its children, and their children. Deeper than that, strip the excess, create the parents, then append the remainder to the returned block ids. |
| **2000 characters per text span** | Split long prose across several spans in the same `rich_text` array, not several blocks. |
| **Exactly three heading levels** | There is no `heading_4`. Flatten deeper levels to bold body text or a `toggle` — never emit a literal `####`. |
| **`table` / `column_list` / `column` are invalid without children** | They cannot be created empty and filled later, so they must fit inside the nesting budget above. Keep them at the top level or one level in. |
| **`replace`-style rewrites delete blocks** | There is no page-level replace. Deleting every child and rewriting loses comments and block links anchored to those blocks. Prefer appending; read the page first. |

## Checklist

Before writing:

- Does the reader get the answer from the first callout plus the first table, without scrolling?
- Is every number in a table rather than in a sentence?
- Has anything with a hand-typed ordinal in a heading been rewritten?
- Are all icons from the monochrome set, and is each one used for one role only?

After writing — fetch the page and confirm:

- Does any heading own fewer than two blocks?
- Is any `heading_2` carrying more than six unfolded `heading_3`?
- Does any `heading_3` hold more content than the sections around it combined?
- Do adjacent tables or code blocks have a line of text between them?
- No literal markers (`####`, `[!NOTE]`, `:::`) rendered as text.
- Callouts contain what they should, and did not absorb the blocks that followed.
- Nested lists kept their depth instead of flattening.
- CJK/latin spacing survived, and code spans were left alone.

## Cut

- Emoji as decoration in body text. Icons belong to callouts and mean something.
- A background colour on every callout. Colour used everywhere signals nothing.
- Headings that only announce the next paragraph (`## Overview`, `## Introduction`).
- A `table_of_contents` on a page with three headings, and a `divider` between every section — both add chrome to a page short enough not to need it.
- Restating a table in the paragraph above it.
- Screenshots of tables. Rebuild them as tables; they stay searchable and diffable.
