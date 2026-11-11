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

| Need | Default | Alternatives |
|---|---|---|
| PDF text | `pdftotext` (`brew install poppler`) | Any local extractor. Add OCR for scans: `ocrmypdf`, MinerU |
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

Fill in the placeholders first. The agent stops if any are left:

```bash
grep -rn '<[A-Z_]*>' ~/.claude/skills/helios
```

Then restart the session, log in to Helios once in the driver's browser, then:

```bash
/helios
/helios review only
/helios ER00000001 PUBCA0000000001
```

It runs only on `/helios` (`disable-model-invocation: true`), because it approves documents. Nothing should approve on a hunch.

## Adapting it to your tenant

- **Host.** `references/helios-ui.md` uses `<HELIOS_HOST>`. Replace it with your tenant's console host.
- **Thresholds.** The table at the end of `SKILL.md` has placeholders (`<LINK_ANY_USD>`, `<LINK_CONTRACT_USD>`, `<CEO_APPROVAL_USD>`, `<KOL_FLAG_USD>`). Fill them in with your company's policy; they are not Helios behaviour.
- **Form names.** The rules key on document prefixes (EA / ER / PUBCA / PUKOL) and English form labels. Another tenant's custom forms may label fields differently.

## Known limits

- **CPV has no absolute standard.** The skill compares each document with the median of the same batch. A small or skewed batch makes a weak reference, so the report states the reference it used.
- **Unsigned contracts are flagged even when Docusign follows approval.** They are labelled "Docusign pending" so you can clear them as a group. If your process always signs after approval, relax that rule.
- **Duplicate detection is only as wide as the history search.** That search allows a 12-month window per request. An invoice paid outside Helios is invisible to it.
- **It is slow on purpose.** It spends about 10 seconds per document opening the drawer, plus attachment downloads, and it opens every attachment.
