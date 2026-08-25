# `brief`

> Turn the current conversation into a project brief a colleague can read.

## Your best thinking is sitting in a terminal nobody will ever read.

You just spent two hours deciding something. You weighed five approaches, killed
four, and you know exactly why. Then a teammate asks what you're building — and
you start improvising. Badly.

So you ask an agent to write it up, and you get the genre: four paragraphs on why
the file lives where it lives, a confident metric counting its own edits, a
"risks" section that's the previous section wearing a hat. Nobody reads that
either.

[`handoff`](https://www.aihero.dev/skills-handoff) compacts a session for the
next agent. **`brief` compacts it for the humans** — and it's under standing
orders to shut up about itself.

One command, one link: the goal, the route you picked, the four you didn't, and
the honest cost of the one that won. It opens with a single word — **Locked**,
**Exploring**, **Undecided** — because that's what your reader needs before they
decide anything else.

Every decision has to pass one test: *you could reasonably have chosen
otherwise, and knowing why changes what the reader does.* Three or four survive
it. Ten means you wrote working notes and called them a brief.

Stop re-explaining it. Send the link.

## Why not just reuse `handoff`

Because one of its rules has to invert. `handoff` says *do not duplicate what
already exists in a PRD, plan, ADR or commit — reference it by path*. That is
correct when the reader has a filesystem. A teammate reading a link cannot open
`docs/plan.md:42`, so `brief` **inlines the substance** and demotes paths to
footnotes.

|  | `handoff` | `brief` |
| --- | --- | --- |
| Reader | the next agent, with a filesystem | a colleague who was not in the session |
| Existing docs | referenced by path | substance inlined |
| Organised by | what to do next | why it matters |
| Output | `HANDOFF.md` in a temp dir | a private published page |
| Argument | what the next session is for | who is going to read it |

## Usage

```bash
/brief for the backend team
```

The argument is the reader, and it drives the whole page: an engineer gets the
mechanism, a lead gets the tradeoff, a cross-functional partner gets the impact
and the ask. Called with no argument, the skill asks instead of guessing.

## The seven sections

The structure is the whole value of this skill. Everything else is wording.

1. **Goal** — one sentence a reader could later judge as met or missed.
2. **How we get there** — the chosen route, plus a status badge: `Locked`,
   `Exploring`, or `Undecided`. A brief that hides whether the approach is
   settled is worse than no brief — the reader's next move depends on that word.
3. **Routes considered** — exactly one chosen; every other one states *why* it
   lost. The reason has to be a fact someone could argue with (a cost, a missing
   capability, a constraint violated), never "too complex" on its own.
4. **Key decisions** — tradeoffs *within* the chosen route: context / chose /
   rejected / cost. Not a restatement of §3.
5. **Progress** — done / in flight / deliberately not done.
6. **Risks and open questions** — known ceilings, unowned problems.
7. **What we need from you** — the ask, or an explicit "nothing, FYI".

Sections 2 and 3 are the spine. A section with no real content gets dropped,
not padded.

## What it refuses to write

The characteristic failure of an agent-written document is self-justification —
defending choices nobody questioned. Keeping the reasoning behind a real
decision is the entire point of §4; defending a detail is noise, and the two are
easy to confuse. So the test is explicit: a decision earns a place only if the
reader could reasonably have chosen otherwise **and** knowing why changes what
they do.

Cut on sight:

- A defense of something nobody challenged — and any defense of a defense.
- Housekeeping dressed as a decision: where a file lives, what it is named.
- A decision restated later as a risk or an open question.
- Self-reference, and metrics that measure only the author's own effort:
  lines written, files touched, searches run.
- Process narration: what was searched, read, or weighed. The reader wants the
  conclusion, not the trail.

Three or four surviving decisions makes a good brief. Ten means most of them
are working notes.

## Language

The output follows the language of the conversation — a Chinese session produces
a Chinese page, with no translation and no bilingual duplication. The section
names in `SKILL.md` are canonical English; the headings are rendered in whatever
language the session is in. Technical terms and code identifiers stay as-is.

## Design

[`style.css`](./style.css) holds the tokens and components, inlined into the
published page. It is a single light theme by choice, not omission:

| Token | Value | Role |
| --- | --- | --- |
| `--bg` | `#FBF9F6` | warm off-white ground — not pure white |
| `--ink` | `#2f2b2a` | warm near-black — not pure black |
| `--coral` | `#E8857F` | the only accent |
| `--maroon` | `#7A3F40` | headings, selected state, `Locked` badge |
| `--blue` `--gold` `--taupe` | | secondary, used sparingly |

Type is Raleway throughout, separated by weight (900 / 800 / 700 / 500) rather
than by family. The signature component is `.card` — a 4px left accent rail
coloured per card through `--ac`.

To make it yours, replace the `:root` block and leave the component classes
alone; `SKILL.md` refers to components by class name, not by colour.

## Install

```bash
git clone https://github.com/OliverFGBKB/skills.git
cp -r skills/skills/brief ~/.claude/skills/
```

Restart the session before calling it — Claude Code enumerates skills at
startup.

## Known limits

- **Not a document of record.** The output is a published page: no git history,
  no diffs. Inlining substance means it duplicates the source docs and will
  drift from them. That is an acceptable trade for something read once and
  discussed; it is the wrong tool for a decision log you intend to maintain.
- **No decision ledger.** §4 deliberately has no numbering, no
  Proposed/Accepted/Superseded lifecycle, no index. If you need long-term
  traceability, pair this with a real ADR skill rather than stretching this one.
