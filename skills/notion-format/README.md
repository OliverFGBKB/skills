# `notion-format`

> Write Notion pages that look composed, not dumped.

## Everything an agent writes into Notion looks like it was pasted from a terminal.

Ask an agent to publish a report to Notion and you get one long column of
paragraphs. Every number is inside a sentence. Headings are numbered by hand, so
the first time a section moves they are all wrong. A callout shows up every
fourth block wearing a different coloured emoji, which means the reader has
learned nothing from any of them.

Nothing errored. The API returned `200` on every call. The page is just unreadable.

That happens because Notion is a **block** system and markdown is a **stream**,
and nothing in between makes the translation decisions: which numbers become a
table, which caveats collapse into a toggle, which single sentence earns the one
callout at the top. This skill makes those decisions, and it carries the block
JSON so the agent stops inventing shapes the API rejects.

## Why not just let the MCP server do it

Because a transport has no opinion. The Notion MCP server will happily accept
sixty paragraph blocks in a row — that is exactly what it is for. And it does
not know that `icon.emoji` silently refuses `◆`, that nesting is capped at two
levels per request, or that there is no `heading_4`.

|  | MCP server / raw API | `notion-format` |
| --- | --- | --- |
| Decides block types | no — you send what you wrote | yes — numbers to tables, caveats to toggles |
| Knows the failure modes | no | the seven that fail quietly |
| Heading numbering | whatever you typed | never numbered |
| Callout discipline | unlimited | conclusion and risk only |
| CJK / latin spacing | untouched | one space, never inside code or URLs |

It sits on top of whatever transport you already have. It does not replace one.

## The six rules

The style is the skill; the JSON is just plumbing.

1. **No numbering in headings.** The level is the hierarchy. A typed ordinal is
   correct only until a section moves, and Notion has no auto-numbering to fix it.
2. **Monochrome icons** — ◻️ ◼️ ▪️ ✔️ ✖️ ⚫ ⚪. One glyph per role, all the way
   down the page. Coloured emoji read as a toy.
3. **Restrained surface.** No background by default. Callouts for the conclusion
   and for risk — a page with five callouts has none.
4. **Structure over prose.** Numbers to a table, methodology to a toggleable
   heading, decisions to checkboxes, comparisons to columns. A reader should get
   the answer from the first callout and the first table.
5. **Space CJK against latin** on mixed-script pages, and never around full-width
   punctuation or inside code spans.
6. **Read the page back.** A `200` is not a correct page.

Rule 6 is not hygiene advice. Every limit documented in this skill was found by
writing a page, fetching it, and seeing something wrong that no error had
reported: `[!TIP]` sitting in a callout as literal text, `####` printed as four
hashes, four blocks absorbed into the callout above them.

## The limits it encodes

Verified against the live API, because the documentation does not lead with them:

- `icon.emoji` accepts **only real emoji**. `◆ ▸ ● ※ ❖`, and text-variation
  forms like `⚠︎`, all return `400 validation_error`. Emoji that are themselves
  black-and-white pass, and an `external` URL icon is accepted — which is the
  escape hatch for a genuinely grey glyph.
- **100 blocks per request**, and **two levels of nesting** per request. Deeper
  trees have to be created in stages against the ids the API hands back — and
  those ids only cover the top level.
- **2000 characters per text span**, split across spans rather than blocks.
- **Three heading levels, no more.** Anything deeper is bold text or a toggle.
- `table`, `column_list` and `column` are **invalid without children**, so they
  cannot be created empty and filled in later, which puts them in tension with
  the nesting cap.

## Install

```bash
git clone https://github.com/OliverFGBKB/skills.git
cp -r skills/skills/notion-format ~/.claude/skills/
```

Restart the session before using it — Claude Code enumerates skills at startup.

Unlike `brief`, this one is meant to fire on its own: it has no
`disable-model-invocation`, because the moment to apply it is the moment someone
says "put this in Notion", not a moment you remember to invoke.

## Known limits

- **It formats; it does not transport.** You need a Notion MCP server, an API
  token, or your own wrapper. The skill tells the agent what to build and what
  the API will refuse, not how to authenticate.
- **The mechanical work stays with the agent.** Chunking at 100, splitting text
  at 2000 characters, re-attaching children deeper than two levels — the skill
  documents each rule, and the agent has to honour it every time. On a page you
  write once, that is fine. If you publish to Notion daily, wrap the API in a
  script and let code enforce the arithmetic.
- **No markdown dialect.** It deliberately does not define a shorthand for
  callouts or columns. A dialect only works next to a parser that understands it;
  written into Notion without one, the markers land as literal text — the exact
  failure this skill exists to prevent.
- **Style, not taste.** It will stop a page from looking dumped. It will not tell
  you the analysis is thin.
