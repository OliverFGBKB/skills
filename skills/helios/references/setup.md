# Setup: dependencies, then the approver's policy

Setup has two parts, and they run in order. **Part 1** installs and checks every dependency for this OS. **Part 2** asks the approver how strict to be and writes their personal policy file.

- Setup approves nothing.
- It is done when every row of the dependency checklist shows ✓ and the policy file exists with no `<…>` left in it.
- If only the dependency check failed, run Part 1 and skip Part 2.

**Asking the approver.** Where this file says **Ask**, use the harness's choice tool (`AskUserQuestion` in Claude Code). It takes at most 4 questions per call and 2–4 options per question, and it adds "Other" for free text by itself. Put the install link or command in each option's description. If the harness has no choice tool, print a numbered list and wait for a reply.

**Installing.**
- **Run the command yourself after the approver picks an option.** Before running a package-manager command, confirm the package exists, e.g. `brew info X`, `winget search X` or `apt-cache policy X`. Never guess a package id.
- **Hand over what needs a human.** A GUI installer, an app's first-run onboarding, or a `sudo` password goes to the approver as a one-line action, e.g. "type `! sudo apt install …`". Then wait for them to confirm.

## Windows desktop agents

On Windows, setup usually runs inside a desktop app: Claude Code in the Claude desktop app, or Codex in the ChatGPT desktop app. The agent's shell there is PowerShell or Git Bash. Settle these points before the checklist, because each one makes a later check fail for a reason that has nothing to do with the dependency.

- **winget may be missing.** Run `winget --version`. If it fails, **Ask** which route to take:
  - *App Installer from Microsoft Store (Recommended)*: link the approver to https://apps.microsoft.com/detail/9NBLGGH4NNS1.
  - *winget-cli release*: https://github.com/microsoft/winget-cli/releases. The approver installs the `.msixbundle`.
  - *Scoop*: https://scoop.sh, a user-level install that needs no admin rights.
  - *Direct installers*: one link per tool, taken from the tool table below.

  Corporate machines often block the Store. In that case, offer Scoop or the direct installers first.
- **Install without prompts.** Add `--accept-package-agreements --accept-source-agreements --silent` to every `winget install`, and `--scope user` where the package supports it. A machine-scope install raises a UAC prompt that only the approver can click. Say so before running it, then wait.
- **The shell won't see new tools until PATH is refreshed.** The agent's shell keeps the PATH it started with. After each install, reload PATH in the same shell before re-checking:
  ```powershell
  $env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')
  ```
  If the check still fails, the approver must fully quit and reopen the desktop app.
- **Portable packages can leave no command on PATH.** Poppler is one of them. Locate the binary, e.g. `Get-ChildItem $env:LOCALAPPDATA\Microsoft\WinGet\Packages -Recurse -Filter pdftotext.exe`, and add its folder to the user PATH.
- **Use `curl.exe` and `py -3`.** In Windows PowerShell 5, `curl` is an alias for `Invoke-WebRequest`. And `python3` can open the Microsoft Store stub instead of running Python.
- **Register MCP servers through the host app**, not through a CLI that may be absent:
  - Claude Code: `claude mcp add …`, or the desktop app's connector settings.
  - Codex: `codex mcp add …`, or `~/.codex/config.toml`.

  Check the exact syntax with the tool's `--help` first.
- **Install the skill in the host app's own skills folder.** Claude Code uses `%USERPROFILE%\.claude\skills\helios`. Codex uses `%USERPROFILE%\.codex\skills\helios`; check that path against the Codex docs before relying on it. Restart the app after copying.
- **ChatGPT desktop without Codex has no shell.** It can't run this skill. Tell the approver, and stop setup.
- **No choice tool.** Codex has no `AskUserQuestion`, so use the numbered-list fallback.

## Part 1: dependencies

### Show the checklist first

Detect the OS (`uname -s`, or `$env:OS` on Windows), the CPU architecture, and the host app (Claude Code, Codex, other). On Windows, go through *Windows desktop agents* above first. Run every check below, then show this table with ✓ / ✗ before installing anything:

| # | Dependency | Check | Required |
|---|---|---|---|
| 1 | Background browser driver | see 1 below | yes |
| 2 | PDF text: `pdftotext`, `pdftoppm` | `pdftotext -v` | yes |
| 3 | OCR for scanned PDFs | `ocrmypdf --version` and `tesseract --list-langs` | recommended |
| 4 | `.docx` text | macOS: `textutil` is built in. Otherwise `pandoc -v` | recommended |
| 5 | `curl`, Python 3 | `curl --version`, `python3 --version` (Windows: `curl.exe --version`, `py -3 --version`) | yes |

Then work through the ✗ rows in order. Re-run each check after its install, and move on only when it shows ✓.

### 1. Browser driver

It must drive a real Chromium that holds the approver's Helios login, through CDP or an extension. It must also leave the OS mouse and keyboard alone. Tools that work from screenshots and move the OS cursor are never offered.

**Check**: `command -v ego-browser`. Also check whether the harness already has a CDP browser tool, such as a Playwright or Chrome DevTools MCP server, or Claude in Chrome.

**Ask** "Which browser driver should Helios approvals use?":

| Option | Description to show | Then do |
|---|---|---|
| ego lite (Recommended) | Agent-first Chromium that works in the background and can import your Chrome login. https://lite.ego.app/ | macOS: if the `ego-browser` skill is installed, run its `scripts/install.sh`. Otherwise download the build for this architecture from https://lite.ego.app/ and install it. Windows: open https://lite.ego.app/ for the approver to run the installer. Both: the approver finishes onboarding in the app, which puts `ego-browser` on PATH. Wait for them, then run `ego-browser skill` to verify. |
| Playwright MCP, extension mode | Drives your own Chrome tabs through the Playwright MCP Bridge extension. https://github.com/microsoft/playwright-mcp | Confirm the flag with `npx @playwright/mcp@latest --help`. Register it with the harness, e.g. `claude mcp add playwright -- npx @playwright/mcp@latest --extension`. The approver installs the bridge extension in Chrome. |
| Chrome DevTools MCP | Attaches to a Chrome you start with remote debugging. https://github.com/ChromeDevTools/chrome-devtools-mcp | Confirm the flags with `--help`. Register it with `--browser-url=http://127.0.0.1:9222`. The approver starts Chrome with `--remote-debugging-port=9222` on a profile that holds the Helios login. |
| A tool this harness already has | Use it only if it can run page JS, fetch with page cookies, read request headers and response bodies, and click a CSS selector. | Test those four capabilities on any page. If one fails, ask again. |

**Verify**: open the Helios host (Part 2 collects it, so ask for it now if needed) and read the page title. A login page means the driver works. Ask the approver to sign in once in that browser.

### 2–5. Command-line tools

Use the row for this OS. Install everything that is missing in one command, after one **Ask**: "Install these missing tools now?", with the options *Install all (Recommended)* / *Pick per tool* / *Skip optional ones*.

| Dependency | macOS (Homebrew) | Windows (winget; see *Windows desktop agents* if it is missing) | Debian / Ubuntu |
|---|---|---|---|
| poppler | `brew install poppler` | `winget install oschwartz10612.Poppler` (portable: add its `bin` to PATH, see above). Scoop: `scoop install poppler` | `sudo apt install poppler-utils` |
| OCR | `brew install ocrmypdf tesseract-lang` | `winget install UB-Mannheim.TesseractOCR` + `winget install ArtifexSoftware.GhostScript` + `py -3 -m pip install --user ocrmypdf`. Tick Chinese in the Tesseract language list, or add `chi_sim.traineddata` | `sudo apt install ocrmypdf tesseract-ocr-chi-sim` |
| pandoc | not needed: `textutil` is built in | `winget install JohnMacFarlane.Pandoc`. Scoop: `scoop install pandoc` | `sudo apt install pandoc` |
| curl, python3 | built in; `brew install python` if missing | `winget install Python.Python.3.12`. `curl.exe` ships with Windows 10 1803+ | `sudo apt install curl python3` |

- **No Homebrew on macOS**: link https://brew.sh. Installing it needs the approver's password, so hand them the one-line installer from that page.
- **Other Linux distros**: use the native package manager. Search for the same package names first.
- **OCR languages**: Chinese invoices need `chi_sim` in `tesseract --list-langs`. Add other languages the approver's vendors invoice in.

## Part 2: the approver's policy

Write to `~/.config/helios/policy.md` (or `$HELIOS_POLICY`). If the file already exists, show its current values. Ask only about the keys the approver wants to change, and keep the rest.

Before the first question, tell the approver: "The thresholds are your company's approval policy. Take them from the policy document, not from these options."

**Ask, round 0** (plain text, one question): "What is your Helios console host? Copy it from the address bar after logging in, e.g. `console-xx.huilianyi.com`." → `host`

**Ask, round 1: how the run behaves**

| Key | Question | Options (first = recommended) |
|---|---|---|
| `approve_mode` | After review, how should clean documents be handled? | `auto`: approve them right away / `confirm_batch`: show the list, approve on my yes / `review_only`: never approve |
| `report_language` | Report language? | 中文 / English |
| `carryover` | A document reviewed in an earlier run and unchanged since? | `reuse`: keep the earlier verdict / `re-review`: review it again |
| `minor_issues` | Minor issues, e.g. a wrong category or a narrow invoice description with the right amount? | `flag`: leave the document for me / `approve_and_note`: approve it and note the issue |

**Ask, round 2: company thresholds, in USD.** Every question also takes free text through "Other".

| Key | Question | Options |
|---|---|---|
| `link_any_usd` | From what amount must a payment link a request or contract? | 0 (always) / 100 / 500 |
| `link_contract_usd` | Above what amount must a payment link a contract? | 1000 / 2500 / 5000 |
| `ceo_approval_usd` | Above what amount does a contract or payment need top-level (CEO) approval? | 5000 / 10000 / 25000 |
| `kol_flag_usd` | What fee threshold does the KOL request form's "exceeds" flag use? | 1000 / 2000 / 5000 |

**Ask, round 3: how strict to be**

| Key | Question | Options |
|---|---|---|
| `cpv_outlier_multiple` | Flag a creator request when its CPV is at least how many times the batch median? | 3 / 2 / 5 / off |
| `views_above_median` | Estimated views clearly above the creator's median views? | `flag` / `ignore` |
| `unsigned_docusign` | Contract unsigned, but the form says Docusign follows approval? | `flag` / `allow` |
| `fx_tolerance_pct` | A contract's implied FX rate is off market by more than? | 5 / 2 / 10 / off |

**Ask, round 4: cross-document checks**

| Key | Question | Options |
|---|---|---|
| `split_window_days` | Treat documents to one vendor as a possible split when submitted within? | 0 (same day) / 7 / 30 / off |
| `duplicate_lookback_months` | How far back to search for a duplicate invoice? | 12 / 6 / 24 |

### Write and confirm

1. Fill [policy-template.md](policy-template.md) with the answers, `mkdir -p ~/.config/helios`, and write the file there.
2. Show the approver the result as one table (key, value, what it means).
3. **Ask**: "Save this policy?" with the options *Save (Recommended)* / *Change some answers*. On *Change*, re-ask only those keys.
4. Finish with a one-line next step. Suggest `/helios review only` for a dry run before the first real run.
