---
name: helios
description: Review and approve the documents waiting for you in Helios (Huilianyi / 汇联易 expense system) → My Approvals. Opens every pending request, contract and payment, checks each one against its attachments under your personal approval policy, approves the clean ones, and reports the rest with reasons. First run walks you through installing dependencies and setting that policy. Use for "/helios", "/helios setup", "approve my Helios documents", "审批 Helios / My Approvals", or reviewing a single Helios EA / ER / PUBCA / PUKOL document.
disable-model-invocation: true
argument-hint: "[setup | review only | document numbers]"
---

# Helios approvals

You review the approver's **My Approvals** queue one document at a time:

- **Clean** → approve it.
- **Anything wrong** → leave it untouched, and list it in the report at the end.

How a document counts as clean is set by the approver's **policy**: a file of `key: value` lines at `~/.config/helios/policy.md`, or at the path in `$HELIOS_POLICY` when that is set. `policy.<key>` below means a value from that file.

## Route the run

| Situation | Do |
|---|---|
| Argument is `setup`, the policy file is missing, or it still contains a `<…>` placeholder | Follow [references/setup.md](references/setup.md) to the end, then stop. Approve nothing in a setup run. |
| A dependency check from setup fails now (driver or `pdftotext` missing) | Run only the dependency part of setup. |
| Argument is `review only`, or `policy.approve_mode: review_only` | Steps 1–3 and 6. Approve nothing. |
| Argument is document numbers | All steps, limited to those documents. |
| No argument | All steps, whole queue. |

Read the policy file before the first browser call. Interfaces, selectors and ready-to-run scripts are in [references/helios-ui.md](references/helios-ui.md); read it before the first browser call too.

## Guardrails

- **The only write action** is the detail panel's **Approve**, then **OK** in its dialog, with the comment left empty.
- **Problem documents stay exactly as they are.** No Reject, Return, comment, edit or message. The reasons go only into the report.
- **Login or 2FA screen** → stop and ask the user to sign in. Never type credentials.
- **The report contains no bank account numbers, IBANs or credentials.** If a document's text holds a plaintext password, write only "reason field contains plaintext login credentials — rotate and scrub".
- **Attachments stay on this machine.** Download them to a per-document folder under the session scratch directory and read them locally.
- **The browser runs in the background.** Use only a driver that leaves the user's mouse and keyboard alone; setup lists the supported ones.

## Visible-only

Base every finding on what the document page **actually shows**.

- API JSON is a fast way to locate data. Each field a finding cites must also appear in the detail panel's rendered text.
- Fields hidden by form linkage rules count as absent. Don't cite their stale values, and don't flag them as "missing". Example: on the KOL request form, the fee-threshold flag (`policy.kol_flag_usd`) set to NO hides Audience, Reasons for Collab, Collaboration Breakdown, Usage Rights, Expected Live Month and Campaign.

## Steps

1. **List.** Capture the pending list and record each document's number, applicant, form, entity and amount. Mark carry-overs: documents already reviewed in an earlier run.
   - Done when the list count equals the page's `Pending - All (N)`.
2. **Capture.** For every document, save the detail panel text, its API JSON, its invoice JSON, and every attachment.
   - Done when every document has a panel text file, and every contract or payment has its attachments on disk. Attachments can fail to download because of an expired signed URL. Retry the other URL variants until each file is on disk.
3. **Review.** Apply the [Rules](#rules) by document type. Open every attachment; never judge a file by its name.
   - Carry-overs, with `policy.carryover: reuse`: compare the panel text and the attachment hashes with the earlier run. If both are unchanged, reuse the earlier verdict and say so. With `re-review`, review them again from scratch.
   - Done when every document is either **clean** or **problem + reasons**, and every reason points at a visible field or an attachment line.
4. **Approve the clean ones.** With `policy.approve_mode: confirm_batch`, first show the clean list and wait for a yes. For each one, open its detail panel and confirm the panel's `Document No.` is the target. Then click the panel's Approve and confirm with OK.
   - Never use the row-level quick Approve.
   - Skip, and record, any document whose panel number does not match or that has left the queue.
5. **Verify.** Reload the list. Every approved document must be gone, and the remaining list must equal the problem list.
   - Documents that arrived during the run are out of scope. Report them, and ask whether to run again.
6. **Report** in `policy.report_language`, conclusion first:
   - One line: how many were pending, approved, and left.
   - A table of approved documents: number, applicant, one-line basis.
   - A table of problem documents: number, applicant, reasons. Each reason names the field and the attachment that disagree.
   - Skipped documents and new arrivals, if any.
   - Any policy value this run suggests changing, as an offer. Change the policy file only when the user says yes.

## Rules

A rule whose policy value is `off` is skipped.

### Requests (EA: KOL collab, sales & marketing expense)

Judge whether the request itself makes sense. A request does not need attachments.

- Creator type, market, content and purpose agree with each other. For example, the audience geography fits the market.
- The fee fits the data: followers, median views, estimated views, CPV.
  - With `policy.views_above_median: flag`, flag estimated views clearly above median views.
  - Flag CPV at `policy.cpv_outlier_multiple` × the batch reference or more. The reference is the median CPV of comparable requests in the same batch; state it in the report.
- The currency, the "in USD" amount field, and the fee-threshold flag (`policy.kol_flag_usd`) agree with each other.
- Flag one creator with two requests in the same period, whether a duplicate or a split to stay under a threshold.

### Contracts (PUBCA)

Apply the request checks, then check the contract text in the attachment:

- **Entity**: our party on the contract = the document's signing entity.
- **Amount and currency**: the contract amount and payment terms (installments, amounts, dates) = Planned Payment Info. When the currencies differ, compute the implied FX rate. Flag it if it is more than `policy.fx_tolerance_pct`% off market.
- **Counterparty**: = Counterparty Info. An influencer contract may use Payee-Influencer, but the payee account must belong to that counterparty.
- **Payee**: = the party the reason text says gets paid. For example, a reason that says "paid via Impact" needs Impact as payee.

Also flag:
- **Unsigned attachment.** If the form says Docusign will be initiated after approval, `policy.unsigned_docusign` decides:
  - `flag` → flag it as "unsigned — Docusign pending".
  - `allow` → the missing signature alone doesn't block approval.

  An unsigned contract without Docusign is always flagged.
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
- **Linkage**: a payment of `policy.link_any_usd` or more links a request or contract. Above `policy.link_contract_usd` it links a contract.
- **Duplicates**: compare the invoice number with earlier documents for the same vendor, including drafts, over the last `policy.duplicate_lookback_months` months. Compare the contract's paid amount with this payment.
- **Entity and currency**: they match exactly across the contract, invoice and payment.

### Across documents

- **Split approvals.** Flag documents to one vendor within `policy.split_window_days` days where each is below `policy.ceo_approval_usd` and their total is above it. A value of 0 means the same day.
- **A resubmission of a document flagged earlier.** Say which earlier problems are fixed and which remain.

### Minor issues

Examples: a wrong category, inconsistent checkboxes, or an invoice description that is narrower than the contract while the amount still matches. `policy.minor_issues` decides:
- `flag` → treat the document as a problem.
- `approve_and_note` → approve it, and list the issue under the approved row.

### When unsure

Always flag, whatever the policy says:
- An unusually large amount that is internally consistent.
- Document types these rules don't cover.
