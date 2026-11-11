---
name: helios
description: Review and approve the documents waiting for you in Helios (Huilianyi / 汇联易 expense system) → My Approvals. Opens every pending request, contract and payment, checks each one against its attachments, approves the clean ones, and reports the rest with reasons. Use for "/helios", "approve my Helios documents", "审批 Helios / My Approvals", or reviewing a single Helios EA / ER / PUBCA / PUKOL document.
disable-model-invocation: true
argument-hint: "[optional: document numbers to limit the run, or 'review only']"
---

# Helios approvals

You review the approver's **My Approvals** queue one document at a time:

- **Clean** → approve it.
- **Anything wrong** → leave it untouched, and list it in the report at the end.

Argument handling: no argument means the whole queue. Document numbers limit the run to those. `review only` runs steps 1–2 and the report, with no approvals.

Before anything else, check that `<HELIOS_HOST>` (in the reference) and the threshold placeholders (end of this file) are filled in. If any `<…>` placeholder is left, stop and ask the user for the value.

Interfaces, selectors and ready-to-run scripts live in [references/helios-ui.md](references/helios-ui.md). Read it before the first browser call.

## Requirements

- **A background browser driver.** It must drive a real Chromium that already holds the approver's Helios login, through CDP or an extension, without taking the OS mouse or keyboard focus. The default is ego lite (`ego-browser`). Any harness browser-use tool with the same properties works; [the README](README.md#browser-driver) lists what it needs.
- **A local PDF text extractor.** Use `pdftotext` (poppler), with an OCR fallback for scanned PDFs. Attachments carry bank details, so read them on this machine only. Never upload them.

## Guardrails

- **The only write action** is the detail panel's **Approve**, then **OK** in its dialog, with the comment left empty.
- **Problem documents stay exactly as they are.** No Reject, Return, comment, edit or message. The reasons go only into the report.
- **Login or 2FA screen** → stop and ask the user to sign in. Never type credentials.
- **The report contains no bank account numbers, IBANs or credentials.** If a document's text holds a plaintext password, write only "reason field contains plaintext login credentials — rotate and scrub".
- **Downloads** go to a per-document folder under the session scratch directory.

## Visible-only

Base every finding on what the document page **actually shows**.

- API JSON is a fast way to locate data. Each field a finding cites must also appear in the detail panel's rendered text.
- Fields hidden by form linkage rules count as absent. Don't cite their stale values, and don't flag them as "missing". Example: on the KOL request form, the fee-threshold flag = NO (`<KOL_FLAG_USD>`) hides Audience, Reasons for Collab, Collaboration Breakdown, Usage Rights, Expected Live Month and Campaign.

## Steps

1. **List.** Capture the pending list and record each document's number, applicant, form, entity and amount. Mark carry-overs: documents already reviewed in an earlier run.
   - Done when the list count equals the page's `Pending - All (N)`.
2. **Capture.** For every document, save the detail panel text, its API JSON, its invoice JSON, and every attachment.
   - Done when every document has a panel text file, and every contract or payment has its attachments on disk. Attachments can fail to download because of an expired signed URL. Retry the other URL variants until each file is on disk.
3. **Review.** Apply the [Rules](#rules) by document type. Open every attachment; never judge a file by its name.
   - For a carry-over, compare the panel text and the attachment hashes with the earlier run. If both are unchanged, reuse the earlier verdict and say so in the report.
   - Done when every document is either **clean** or **problem + reasons**, and every reason points at a visible field or an attachment line.
4. **Approve the clean ones.** For each one, open its detail panel and confirm the panel's `Document No.` is the target. Then click the panel's Approve and confirm with OK.
   - Never use the row-level quick Approve.
   - Skip, and record, any document whose panel number does not match or that has left the queue.
5. **Verify.** Reload the list. Every approved document must be gone, and the remaining list must equal the problem list.
   - Documents that arrived during the run are out of scope. Report them, and ask whether to run again.
6. **Report** (in the user's language, conclusion first):
   - One line: how many were pending, approved, and left.
   - A table of approved documents: number, applicant, one-line basis.
   - A table of problem documents: number, applicant, reasons. Each reason names the field and the attachment that disagree.
   - Skipped documents and new arrivals, if any.
   - Any rule this run suggests changing, as an offer. Don't change the skill silently.

## Rules

### Requests (EA: KOL collab, sales & marketing expense)

Judge whether the request itself makes sense. A request does not need attachments.

- Creator type, market, content and purpose agree with each other. For example, the audience geography fits the market.
- The fee fits the data: followers, median views, estimated views, CPV.
  - Flag estimated views clearly above median views.
  - Flag CPV far from the batch reference. The reference is the median CPV of comparable requests in the same batch; state it in the report. Treat about 3× or more as far.
- The currency, the "in USD" amount field, and the fee-threshold flag (`<KOL_FLAG_USD>`) agree with each other.
- Flag one creator with two requests in the same period, whether a duplicate or a split to stay under a threshold.

### Contracts (PUBCA)

Apply the request checks, then check the contract text in the attachment:

- **Entity**: our party on the contract = the document's signing entity.
- **Amount and currency**: the contract amount and payment terms (installments, amounts, dates) = Planned Payment Info. When the currencies differ, compute the implied FX rate and flag it if it is off-market.
- **Counterparty**: = Counterparty Info. An influencer contract may use Payee-Influencer, but the payee account must belong to that counterparty.
- **Payee**: = the party the reason text says gets paid. For example, a reason that says "paid via Impact" needs Impact as payee.

Also flag:
- **Unsigned attachment.** If the form says Docusign will be initiated after approval, label it "unsigned — Docusign pending", so the user can decide that group in one go.
- **Our party undecided, or placeholders unfilled** (e.g. `XXXX`, `\s1\`).
- **Fees in the text outside Planned Payment**: commission, CPA, bonus.
- **A term or exposure period that differs from the reason text.**
- **An intermediary that disclaims delivery** ("no obligation to publish", "no warranty") while the contract pays 100% with no payment condition.

### Payments (ER: S&M payment; PUKOL: influencer payment)

Match the invoice against the payment data:

- **Entity**: the invoice bill-to = our entity. The issuer = the payee.
- **Amount and currency**, including every amount field on the form. A header custom field such as "Current Payment Amount" must equal the payment amount.
- **Bank**: account name, account number, SWIFT/IBAN. Check that the payee account appears on the invoice or on the vendor master. Never print it.
- **Expense type, reason and invoice description** describe the same service and period.
- **Allocation** fits the business. A US campaign isn't allocated to EU.
- **Linkage**: the payment links the request or contract the amount requires (see thresholds).
- **Duplicates**: compare the invoice number with earlier documents for the same vendor, including drafts. Compare the contract's paid amount with this payment.

### Across documents

- **Split approvals.** Flag several documents to the same vendor on the same day where each sits below an approval threshold and their total is above it.
- **A resubmission of a document flagged earlier.** Say which earlier problems are fixed and which remain.

### When unsure

Flag it and let the user decide. That covers:
- Wrong category and inconsistent checkboxes.
- An unusually large amount that is internally consistent.
- Document types these rules don't cover.

## Thresholds

These are company policy. Replace each placeholder with your own value before the first run:

| Rule | Value |
|---|---|
| A payment must link a request or contract | ≥ `<LINK_ANY_USD>` |
| A payment must link a contract | > `<LINK_CONTRACT_USD>` |
| A contract or payment needs CEO approval | > `<CEO_APPROVAL_USD>` |
| KOL fee-threshold flag on the request form | `<KOL_FLAG_USD>` |
| Contract and payment entity and currency | must match exactly |
