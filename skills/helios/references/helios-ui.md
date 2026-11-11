# Helios UI and API reference

Host: `https://<HELIOS_HOST>`, where `<HELIOS_HOST>` is `policy.host` from the approver's policy file. Substitute it into every script below. The My Approvals list is at `/main/approve`.

The session is the browser's logged-in cookie plus a bearer token sent in request headers. Never store the token anywhere. Capture it live: enable `Network`, load the list page, and reuse the headers of the page's own `pendingApproval` request for every call made through `page.fetch`.

The scripts below use the ego-browser dialect: `taskSpace`, `page.goto`, `page.evaluate`, `page.fetch`, `page.cdp`, `page.events`, `page.click`. The [last section](#porting-to-another-driver) maps them to other drivers. Set `D` to a per-run scratch folder, with an absolute path.

## Endpoints

| Purpose | Request |
|---|---|
| Pending list | `POST /api/approvals/pendingApproval?roleType=TENANT&page=0&size=200&tableViewClient=web`. Copy the body from the page's request; it includes `"iHaveHandled":false` |
| Request (EA) | `GET /api/application/{OID}` |
| Payment (ER / PUKOL) | `GET /api/v3/expense/reports/{OID}` → `rows` |
| Contract header / lines | `GET /contract/api/contract/header/v2/{OID}`, `GET /contract/api/contract/line/v2/herder/{OID}` (`herder` is the API's own spelling) |
| Expense line: allocation and attachments | `GET /api/invoices/{invoiceOID}?roleType=TENANT&isDateCombinedUTC=false&searchApportion=true&businessCode={docNo}` → `expenseApportion[].costCenterItems`, `receiptList[]` (`title` = bill-to, `payee`, `billingNo`, `fee` in cents, `pdfUrl`) |
| Payment schedule: payee, bank, invoiceOID | `GET /api/corp/payment/schedule/list/v2?roleType=TENANT&expId={reportId}&isPermissionFilter=false` |
| Vendor bank master | `GET /supplier/api/ven/bank/forms/values/{id}`. The panel loads it by itself |
| History search (duplicate invoices) | `POST /api/approvals/filters/get`. The created-date window must be ≤ 12 months; a longer window returns 400, so split the range |

Status codes seen: `1001` draft/editing, `1002` in approval, `1005` paid.

## List: capture the headers and the pending list

```js
const fs = await import("node:fs/promises");
const D = "/ABS/PATH/run";              // per-run folder
await fs.mkdir(D, { recursive: true });
const task = await taskSpace("helios-approve");
const page = task.page("p1");
await page.cdp("Network.enable");
await page.goto("https://<HELIOS_HOST>/main/approve", { timeout: 45000 }).catch(() => {});
await page.waitForTimeout(8000);
const req = (await page.events()).find(e => e.method === "Network.requestWillBeSent" && /pendingApproval/.test(e.params?.request?.url || ""));
if (!req) throw new Error("no pendingApproval request: login page? stop and ask the user");
const r = req.params.request;
await fs.writeFile(`${D}/headers.json`, JSON.stringify(r.headers));   // contains the bearer token: scratch only
const url = r.url.replace(/size=\d+/, "size=200").replace(/page=\d+/, "page=0");
const res = await page.fetch(url, { method: "POST", headers: r.headers, body: r.postData });
await fs.writeFile(`${D}/pending.json`, res.body);
console.log(JSON.parse(res.body).map(x => [x.businessCode, x.applicantName, x.formName].join(" | ")).join("\n"));
console.log("page says:", await page.evaluate(() => (document.body.innerText.match(/Pending - All \((\d+)\)/) || [])[1]));
```

## Capture: the panel text and API JSON for each document

Clicking a row's `td` opens the right-hand detail drawer; the URL does not change. `document.body.innerText` then holds the list *and* the drawer, so trim the text from the `Document No.:` line, three lines back.

```js
const fs = await import("node:fs/promises");
const D = "/ABS/PATH/run";
const CODES = ["ER00000000", "PUBCA0000000000"];
const task = await taskSpace("helios-approve");
const page = task.page("p1");
await page.cdp("Network.enable");
for (const code of CODES) {
  await page.goto("https://<HELIOS_HOST>/main/approve", { timeout: 45000 }).catch(() => {});
  await page.waitForTimeout(5000);
  await page.events();                                   // drop the list page's events
  const ok = await page.evaluate(c => { const td = [...document.querySelectorAll("td")].find(t => t.innerText.trim() === c); td?.closest("tr").click(); return !!td; }, code);
  if (!ok) { console.log(code, "NOT IN LIST"); continue; }
  await page.waitForTimeout(9000);
  const txt = await page.evaluate(() => document.body.innerText);
  const lines = txt.split("\n"); const i = lines.findIndex(l => l.startsWith("Document No.:"));
  await fs.mkdir(`${D}/${code}`, { recursive: true });
  await fs.writeFile(`${D}/${code}/panel.txt`, lines.slice(Math.max(0, i - 3)).join("\n"));
  let n = 0;
  for (const e of await page.events()) {
    if (e.method !== "Network.responseReceived" || !/json/.test(e.params.response.mimeType) || !/\/api\//.test(e.params.response.url)) continue;
    try {
      const b = await page.cdp("Network.getResponseBody", { requestId: e.params.requestId });
      await fs.writeFile(`${D}/${code}/api_${String(n++).padStart(2, "0")}.json`, JSON.stringify({ url: e.params.response.url.replace(/^https:\/\/[^/]+/, ""), body: b.body }));
    } catch {}
  }
  console.log(code, "panel lines", lines.length - i, "apis", n);
}
```

The drawer does **not** load the invoice detail. For payments, collect the `invoiceOID` values from the `expense/reports` and `schedule/list` JSON. Then fetch each one with the captured headers:

```js
const H = JSON.parse(await fs.readFile(`${D}/headers.json`, "utf8"));
const r = await page.fetch(`/api/invoices/${oid}?roleType=TENANT&isDateCombinedUTC=false&searchApportion=true&businessCode=${code}`, { headers: H });
await fs.writeFile(`${D}/${code}/inv_${oid}.json`, r.body);
```

## Capture: download the attachments

Attachments are OSS signed URLs that expire after about an hour. The same file usually appears under several URL variants, and some of them return 403. Collect every variant for each filename and try them until one returns 200. Run this outside the browser; it only needs `curl`:

```python
import json, glob, re, os, subprocess, urllib.parse, collections, sys
D = sys.argv[1]
for d in sorted(glob.glob(f"{D}/*/")):
    code = d.rstrip("/").split("/")[-1]; cand = collections.defaultdict(list)
    for f in glob.glob(d + "*.json"):
        b = open(f).read()
        if os.path.basename(f).startswith("api_"): b = json.loads(b)["body"]
        b = b.replace("\\u0026", "&").replace("\\/", "/")
        for u in re.findall(r'https://[^"\s]+?(?:Signature|Expires|x-oss|OSSAccessKeyId)[^"\s]*', b):
            m = re.search(r"filename(?:\*=UTF-8''|%3D%22|=\")([^\"&;]+)", urllib.parse.unquote(u)) or re.search(r"/([^/?]+\.\w{3,4})\?", u)
            fn = urllib.parse.unquote(m.group(1)).strip('"') if m else None
            if fn and re.search(r"\.(pdf|png|jpe?g|docx|xlsx|zip|ofd)$", fn, re.I): cand[fn].append(u)
    for fn, us in cand.items():
        out = f"{d}att/{fn}"; os.makedirs(os.path.dirname(out), exist_ok=True)
        ok = any(subprocess.run(["curl", "-s", "-o", out, "-w", "%{http_code}", u], capture_output=True, text=True).stdout == "200" for u in dict.fromkeys(us))
        print(code, fn, "OK" if ok else "FAIL")
```

Reading attachments:
- PDF: `pdftotext -layout`. If the text comes back empty, it is a scan, so OCR it.
- `.docx`: `textutil -convert txt -stdout` on macOS, or `pandoc`.
- Signatures are images. Render the signature page (`pdftoppm -r 60 -f N -l N`) and look at it. Text extraction shows an empty `Signature:` even when a signature is present.

Check bank details without printing them. Test whether the panel's payee account string appears in the whitespace-stripped invoice text or in the vendor-bank JSON. Report only true or false.

## Approve

Each document goes through four checks before the click. If any check fails, the document is skipped:
1. It is still in the list.
2. The open panel's `Document No.` equals the target.
3. The Approve button sits inside that panel.
4. The confirmation dialog says Approve.

```js
const task = await taskSpace("helios-approve");
const page = task.page("p1");
const todo = ["EA00000000"];
for (const code of todo) {
  await page.goto("https://<HELIOS_HOST>/main/approve", { timeout: 45000 }).catch(() => {});
  await page.waitForTimeout(5000);
  const ok = await page.evaluate(c => { const td = [...document.querySelectorAll("td")].find(t => t.innerText.trim() === c); if (!td) return false; td.closest("tr").click(); return true; }, code);
  if (!ok) { console.log(code, "NOT IN LIST"); continue; }
  await page.waitForTimeout(7000);
  const found = await page.evaluate(c => {
    let hit = null;
    for (const b of [...document.querySelectorAll("button")].filter(b => b.innerText.trim() === "Approve" && b.offsetParent !== null)) {
      let a = b;
      for (let i = 0; i < 15 && a; i++) {
        a = a.parentElement;
        if (a?.innerText?.includes("Document No.:")) {
          const f = (a.innerText.match(/Document No\.:\s*(\S+)/) || [])[1];
          if (f === c) { b.setAttribute("data-agent-approve", f); hit = f; }
          break;
        }
      }
    }
    return hit;
  }, code);
  if (found !== code) { console.log(code, "panel mismatch, skipped:", found); continue; }
  await page.click(`button[data-agent-approve="${code}"]`, { label: `Approve ${code}` });
  await page.waitForTimeout(2500);
  const dlg = await page.evaluate(() => [...document.querySelectorAll(".ant-modal, [role=dialog]")].map(d => d.innerText).join(" ").slice(0, 80));
  if (!/Approve/.test(dlg)) { console.log(code, "no approve dialog:", dlg); continue; }
  await page.click('loc=role:button[name="OK"]', { label: `confirm approve ${code}` });
  await page.waitForTimeout(5000);
  console.log(code, "-> pending now", await page.evaluate(() => (document.body.innerText.match(/Pending - All \((\d+)\)/) || [])[1]));
}
```

## Verify

```js
await page.goto("https://<HELIOS_HOST>/main/approve", { timeout: 45000 }).catch(() => {});
await page.waitForTimeout(6000);
const left = await page.evaluate(() => [...document.querySelectorAll("tbody tr")]
  .map(r => [...r.querySelectorAll("td")].map(t => t.innerText.trim()).find(t => /^[A-Z]{2,6}\d{8,}$/.test(t))).filter(Boolean));
console.log(JSON.stringify(left));
await task.finish({ keep: [] });   // ego-browser: close the task space
```

## Pitfalls

- **Every list row also has an Approve button.** Tag the panel's button (`data-agent-approve`) and click that tag. Never click by button text alone.
- **Playwright selector chains such as `>> nth=-1` are not supported in ego-browser.**
- **Header amount fields can be 0 or stale.** Approval notifications show "USD 0.00" on some forms. Take the amount from the panel's payment block and the invoice.
- **The first signed URL for a file is often the expired one.** Always retry the other variants.

## Porting to another driver

| Need | ego-browser | Playwright-style / other CDP tools |
|---|---|---|
| Run JS in the page | `page.evaluate(fn, arg)` | `page.evaluate(fn, arg)` |
| Fetch with the page's cookies | `page.fetch(url, opts)` | `page.evaluate(() => fetch(...).then(r => r.text()))` |
| Read request headers | `page.cdp("Network.enable")` + `page.events()` | `page.on("request", r => r.headers())` |
| Response bodies | `page.cdp("Network.getResponseBody")` | `page.on("response", r => r.json())` |
| Click a tagged element | `page.click('button[data-agent-approve="X"]')` | same CSS selector |

A driver that only takes screenshots and moves the OS cursor cannot run these scripts. It also takes the user's mouse away while it works. Don't use one.
