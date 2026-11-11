# `helios`

> Clear your Helios approval queue: approve what checks out, and get a list of what doesn't, with reasons.

## An approval queue is a matching job, and the buttons don't help

Every document in Helios My Approvals asks the same questions. Does the invoice's bill-to match our entity? Does the payee's account appear on the invoice? Is the contract signed? Does it say what Planned Payment says? Has this invoice been paid before?

None of that is on the list page. It is spread across a detail drawer, a payment schedule, an invoice record that the drawer never loads, and PDFs behind signed URLs that expire within the hour. So in practice most documents get approved on the reason text alone.

This skill does the matching. It opens every pending document, pulls the attachments down locally, checks them field by field against the form, approves the clean ones, and leaves the rest untouched with a reason table (illustrative):

| Doc | Applicant | Reason |
|---|---|---|
| PUBCA… ×4 | … | Same vendor, same day, each just under the CEO-approval threshold, well above it combined. The contract disclaims any obligation to publish, yet pays 100% unconditionally. Not signed. |
| ER… | … | "Current Payment Amount" 25,000.00 ≠ payment / invoice 5,000.00 |
| EA… | … | CPV 0.30 against a batch reference of 0.02 (15×) |

## What it will and won't do

- **Approves only**, through the detail panel's own Approve button. It confirms the panel's document number first and never uses the row-level shortcut.
- **Never** rejects, returns, comments, edits or messages. Problem documents look exactly as they did before.
- **Judges only what the page shows.** Fields that a form's linkage rules hide are treated as absent. API JSON is used to find data, not as evidence.
- **Keeps bank data local.** Attachments are read on your machine. The report never prints an account number.
- **Stops at a login or 2FA screen** and asks you to sign in.

## Requirements

### Browser driver

The skill needs a browser it can drive **in the background**. That means a real Chromium session that already holds your Helios login, driven through CDP or an extension. Your mouse and keyboard stay yours while it runs.

| Option | Notes |
|---|---|
| **[ego lite](https://lite.ego.app/)** (default) | A Chromium built for agents. Install the app and finish onboarding, which registers `ego-browser` on PATH and lets you import your Chrome login. Then `ego-browser skill` prints the API guide. It works in the background in task spaces with its own cursor overlay. The reference scripts are written for it. |
| Any harness browser-use plugin that drives a logged-in Chromium over CDP or an extension | Examples are a Playwright / CDP MCP server attached to your profile, or a Chrome-extension driver. It must be able to run page JavaScript, fetch with page cookies, read request headers and response bodies, and click a CSS selector. `references/helios-ui.md` has a porting table. |
| ✖️ Screenshot + OS-mouse "computer use" tools | Not supported. They can't read network headers, and they take over your mouse for the whole run. |

### Everything else

`/helios setup` installs all of these for your OS:

| Need | Default | Alternatives |
|---|---|---|
| PDF text | `pdftotext` (poppler) | Any local extractor. Add OCR for scans: `ocrmypdf` + tesseract, MinerU |
| Signature check | `pdftoppm` (poppler) + an agent that can view images | — |
| `.docx` text | `textutil` (macOS built-in) | `pandoc` |
| Downloads | `curl`, `python3` | — |
| Helios access | You are an approver in the tenant, and logged in once in the driver's browser | — |

Nothing is uploaded anywhere. The only network calls go to Helios and its attachment storage.

## Install

```bash
git clone https://github.com/OliverFGBKB/skills.git
cp -r skills/skills/helios ~/.claude/skills/
```

On Windows, copy it into the host app's skills folder: `%USERPROFILE%\.claude\skills\` for Claude Code (desktop or CLI), or `%USERPROFILE%\.codex\skills\` for Codex in the ChatGPT desktop app. The ChatGPT app without Codex has no shell and can't run this skill.

Restart the session, then run setup once:

```bash
/helios setup
```

Setup is a guided checklist. Each step offers clickable choices, and the agent installs whatever you pick:

1. **Dependency checklist.** It detects your OS and checks each dependency, marking it ✓ or ✗.
2. **Browser driver.** Pick ego lite (recommended), Playwright MCP in extension mode, Chrome DevTools MCP, or a CDP tool your harness already has. Each option comes with its link, and the agent installs it. You only do the parts a human must do: an installer window, the app's onboarding, logging in to Helios.
3. **Command-line tools.** poppler, OCR (ocrmypdf + tesseract with Chinese), pandoc and python, using Homebrew on macOS, winget on Windows, or apt on Debian/Ubuntu. On Windows, setup also covers a missing winget (Store App Installer, Scoop, or direct installers), the UAC prompts, refreshing PATH so the agent's shell sees new tools, and the `curl.exe` / `py` quirks. Every row is re-checked until it shows ✓.
4. **Your approval policy.** About 15 questions in four rounds: how to handle clean documents, your company's thresholds, how strict to be, and cross-document checks. The answers become `~/.config/helios/policy.md`. It lives outside the skill folder, so updating the skill never overwrites it.

Then use it:

```bash
/helios review only                    # dry run: review and report, approve nothing
/helios                                # whole queue
/helios ER00000001 PUBCA0000000001     # only these documents
```

It runs only on `/helios` (`disable-model-invocation: true`), because it approves documents. Nothing should approve on a hunch.

## Your policy

| Group | Keys | Decides |
|---|---|---|
| Run | `host`, `approve_mode`, `report_language`, `carryover`, `minor_issues` | Your tenant; auto-approve, confirm first, or review only; whether unchanged carry-overs are re-reviewed; whether minor issues block approval |
| Company thresholds | `link_any_usd`, `link_contract_usd`, `ceo_approval_usd`, `kol_flag_usd` | When a payment must link a request or contract, the top-level approval line, and the KOL form flag |
| Strictness | `cpv_outlier_multiple`, `views_above_median`, `unsigned_docusign`, `fx_tolerance_pct` | How far off a creator's CPV may be, whether a Docusign-pending contract may pass, the FX tolerance |
| Cross-document | `split_window_days`, `duplicate_lookback_months` | The window for spotting a split, and how far back to look for duplicate invoices |

Change a value by editing the file, or by re-running `/helios setup`, which asks only about what you want to change. After each run, the report may suggest a policy change. It is applied only when you say yes.

The skill ships with no company values. The template in `references/policy-template.md` holds only placeholders.

## Adapting it to another tenant

- **Form names.** The rules key on document prefixes (EA / ER / PUBCA / PUKOL) and English form labels. Another tenant's custom forms may label fields differently.
- **Browser driver.** The reference scripts are written for ego-browser. `references/helios-ui.md` ends with a porting table for Playwright-style drivers.

## Known limits

- **CPV has no absolute standard.** The skill compares each document with the median of the same batch. A small or skewed batch makes a weak reference, so the report states the reference it used.
- **Duplicate detection is only as wide as the history search.** Each request to that search covers at most 12 months, so longer lookbacks take several requests. An invoice paid outside Helios is invisible to it.
- **It is slow on purpose.** It spends about 10 seconds per document opening the drawer, plus attachment downloads, and it opens every attachment.
