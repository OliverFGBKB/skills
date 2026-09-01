---
name: brief
description: Turn the current conversation into a shareable project brief for human teammates — goal, chosen route, routes rejected, key decisions, progress. Published as an Artifact in the house style.
argument-hint: "Who is going to read this?"
disable-model-invocation: true
---

Compact the current conversation into a **project brief for a human colleague** and publish it as an Artifact.

This is the human counterpart of `handoff`. Same input, opposite reader — so one rule inverts: **inline the substance**. A teammate reading a web page cannot open `docs/plan.md:42`. Put the actual reasoning on the page; paths and URLs go in a footnote at most.

## Audience

The argument names the reader. It sets the abstraction level for the whole page — an engineer wants the mechanism, a lead wants the tradeoff, a cross-functional partner wants the impact and the ask. If no argument was passed, ask who the reader is before writing. Do not guess.

Assume the reader was **not** in this session. No "as we discussed", no unexplained internal terms, no chronological narration of how the session unfolded. Organise by why it matters.

## Sections

1. **Goal** — one sentence a reader could later judge as met or missed. Not "improve X" but "get X to Z under condition Y". Then why it is worth solving now.
2. **How we get there** — the chosen route: one sentence of mechanism, and a status badge the reader sees before anything else — **Locked** (`.badge.locked`), **Exploring** (`.badge.wip`), or **Undecided** (`.badge.todo`). A brief that hides whether the approach is settled is worse than no brief; the reader's next move depends entirely on that word. Add one diagram only if it shows the mechanism.
3. **Routes considered** — exactly one is *chosen*; every other one states why it was eliminated. One row each: candidate / verdict / why. The elimination reason must be a fact someone could argue with — a cost, a missing capability, a constraint it violates — never "not elegant enough" or "too complex" standing alone. A route nobody seriously considered does not belong here; padding this section with strawmen is worse than listing two real candidates.
4. **Key decisions** — tradeoffs *within* the chosen route, not the route itself (that is §3 — do not restate it here). Each: context / chose / rejected / cost. This is what people come back for; do not strip it to a list of conclusions.
5. **Progress** — done / in flight / deliberately not done. Being explicit about what was left out matters as much as what shipped.
6. **Risks and open questions** — known ceilings, what could still go wrong, what nobody owns yet.
7. **What we need from you** — the ask, or "nothing, FYI". Never leave this implicit.

§2 and §3 are the spine: what we are doing, whether it is settled, and what we are not doing instead. Drop any other section that has no real content — do not pad it.

## Language

Follow the dominant language of the conversation — a Chinese session produces a Chinese page. No translation, no bilingual output. Keep technical terms and code identifiers in their original form.

The section names above are the canonical English ones. Render each heading in the conversation's language; the structure is fixed, the wording is not.

## Voice

Concise, and structured enough to skim.

- Lead each section with the conclusion, then support it. Body paragraphs of three lines or fewer.
- Let components carry the structure instead of prose: parallel items → `.card`, numbers → `.kpi`, states → `.badge` (`.locked`/`.done`/`.wip`/`.todo`/`.risk`), tradeoff comparisons → `.tokentable`.
- Anything that fits a table or a list is not a paragraph.

## Cut

The failure mode of this document is self-justification — defending choices nobody questioned. Keeping the reasoning behind a real decision is the whole point of §4; defending a detail is noise. The two are easy to confuse, so test every candidate: it earns a place only if the reader could reasonably have chosen otherwise **and** knowing why changes what they do. Three or four survivors makes a good brief. Ten means most of them are working notes.

Cut on sight:

- A defense of something nobody challenged — and any defense of a defense ("this isn't fussiness, it's…").
- Housekeeping dressed as a decision: where a file lives, what it is named, how it is formatted.
- A decision restated later as a risk or an open question.
- Self-reference — the brief describing its own production — and metrics that measure only your own effort: lines written, files touched, searches run.
- Process narration: what you searched, what you read, how many options you weighed. The reader wants the conclusion, not the trail.

## Redact

Session context can hold API keys, passwords, tokens, internal URLs and PII. Publishing sends it outward. Strip all of it before writing the file.

## Render

- Load the `artifact-design` skill first — required before writing any artifact. Load `artifact-diagramming` too if the page earns a diagram. If either skill is not installed in this environment, skip it and rely on the skeleton and [style.css](./style.css) below; do not abort.
- Inline [style.css](./style.css) verbatim into `<style>`, and pull Raleway via `<link href="https://fonts.googleapis.com/css2?family=Raleway:wght@400;500;600;700;800;900&display=swap">` (Google Fonts is the one host the Artifact CSP allows).
- Use the house skeleton: `header.hero` (eyebrow + h1 + `.lead` + `.meta` chips), then one `section.block` per section, each opening with a `.section-label` (name the section — do not number it; a brief is jumped into, not read as a sequence), closing with a `footer`.
- **Single light theme.** This is a deliberate commitment — warm off-white ground, coral as the only accent, no dark mode, no second high-saturation colour competing for attention. Skip the `prefers-color-scheme` blocks; `body` is already explicitly painted.
- Wrap tables and diagrams in `.scroll`. The page itself never scrolls horizontally.
- Write the file to the scratchpad directory, publish with the `Artifact` tool (favicon `📋`), and give the user the link.

### Fallback: no `Artifact` tool

If the `Artifact` tool is unavailable in this environment, do not abort and do not fall back to Markdown — the page is the deliverable. Write the same self-contained HTML to a local file instead:

- Path: `./brief-<slug>-<YYYYMMDD>.html` in the current working directory, where `<slug>` is a few words from the goal. If the cwd is not writable, use the system temp directory.
- Self-contained is now mandatory, not just tidy: CSS inlined in `<style>`, no build step, no local asset references. The Google Fonts `<link>` stays — it degrades to a system font offline, which is acceptable.
- Report the absolute path, and tell the user it opens in a browser (`open <path>` on macOS). Do not claim it was published.
- Everything else — sections, language, voice, cut and redact rules — is unchanged.
