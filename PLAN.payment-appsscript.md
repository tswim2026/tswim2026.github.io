# TSWIM 2026 Registration & Payment — Plan A (Apps Script + ECPay Payment Link)

> **IMPORTANT**: This plan is the single source of truth for the current state of this work. Update it before and after every task. Context may be cleared at any time.

## Branch

`feat/payment-appsscript` (parallel alternative to `feat/payment-ecpay` which implements Plan B)

## Goal

Deliver end-to-end online registration and payment on `tswim2026.github.io` (bilingual, zh/en) with **maximum automation but minimum custom backend code**, by combining:

- **ECPay one-time Payment Links** (pre-generated in the ECPay vendor backend — no AIO signing code required)
- **Google Apps Script** as the only backend (form handler, callback receiver, mailer, receipt generator, cron)
- **Google Sheet** as the data store

The attendee experience is identical to Plan B (register → pay → confirmation → receipt). The difference is purely internal: we avoid operating a Cloudflare Worker, writing a CheckMacValue generator, and managing production secrets outside Google.

On-site late payment is supported but does **not** go through ECPay — handled by staff via bank transfer or cash.

## Strategy: One Script, Three Triggers

One Apps Script project bound to a Google Sheet, with three entry points:

1. **`onFormSubmit`** — Google Form submission → allocate `OrderID` → email bilingual payment link with `CustomField1=<OrderID>`
2. **`doPost`** (Web App) — ECPay `ReturnURL` callback → verify `CheckMacValue` → reconcile by `OrderID` → mark paid
3. **Time-driven trigger (daily)** — scan unpaid rows past deadline → send reminder / expire

Each trigger is independently testable in the Apps Script editor.

## Related Documents

> When updating, write changes in the correct file — do **not** duplicate content across files.

- `PLAN.payment-appsscript.md` (this file) — scope, decisions, tasks, findings for Plan A
- `PLAN.payment-ecpay.md` — parallel Plan B (full AIO integration via Cloudflare Worker)
- `user-journey-map.md` — user journey flowcharts (includes the Plan A variant diagram)

## Current State

- [x] Plan B (AIO) plan written (`PLAN.payment-ecpay.md`)
- [x] Plan A alternative proposed and scoped (this document)
- [ ] Decision: Plan A vs. Plan B confirmed with stakeholder
- [ ] Open questions resolved (refund policy, collecting-entity name, receipt timing)
- [ ] ECPay vendor backend: payment links generated for each ticket type
- [ ] Apps Script project scaffolded
- [ ] End-to-end staging test passing
- [ ] Production launch

## Key Design Decisions

### Why ECPay Payment Links instead of AIO

- ECPay vendor backend provides **「收款連結管理 / Payment Link Management」**, which produces a reusable URL with a fixed amount and fixed product name. We generate **one link per ticket type** once, by hand.
- Signing (CheckMacValue) is done by ECPay on link creation — we never compute it.
- The link still fires `OrderResultURL` / `ReturnURL` callbacks, so automated reconciliation is preserved.

### Binding attendee ↔ payment

Payment links are shared by all attendees of the same ticket type, so the link alone does not identify who paid. We bind per-attendee data using ECPay's pass-through field:

- On form submit, Apps Script generates a unique `OrderID` of the form `TSWIM2026-<8-hex>` (non-enumerable) and writes a Sheet row with `Status=pending`.
- The email to the attendee contains the payment link with `?CustomField1=<OrderID>` appended.
- ECPay echoes `CustomField1` back in the `ReturnURL` POST. `doPost` looks up the Sheet row by `OrderID`, verifies `TradeAmt` matches the ticket price, verifies `CheckMacValue`, and only then marks `Status=paid`.

Rejected alternatives: keying by email (ECPay callback does not include attendee email), keying by amount (collisions when two people pay the same ticket type concurrently), keying by ECPay `MerchantTradeNo` alone (we do not control that value on pre-generated links).

### Secrets & what is safe to put in the attendee email

**`HashKey` / `HashIV` are backend-only, stored in Apps Script Script Properties.** They never appear in any URL, email, frontend bundle, or git-tracked file. Their only use is verifying the `CheckMacValue` on inbound ECPay callbacks inside `doPost`.

The email sent to each attendee contains **no secrets**, only:

```
https://payment.ecpay.com.tw/SP/SPCheckOut?SPToken=<token>&CustomField1=TSWIM2026-abc12345
```

- `SPToken` — opaque reference to the payment-link record we created once in the ECPay vendor backend (ticket type + fixed amount). Not a signing key; cannot be used to impersonate the merchant.
- `CustomField1` — our `OrderID`. Non-secret by design; just the join key between callback and Sheet row.

**Threat model for a leaked payment link** (e.g. forwarded, screenshotted, scraped):

| Action by attacker | Outcome |
|---|---|
| Pay using the link with their own card | Our order gets marked `paid`; no loss to us. |
| Swap `CustomField1` to another valid `OrderID` | They pay for a stranger's registration; log-worthy but not a breach. |
| Invent a fake `OrderID` | Callback hits `doPost`; row lookup fails → flagged `unknown`, not `paid`. |
| Alter the URL to change amount | ECPay ignores URL-level amount tampering (`SPToken` is the source of truth); callback `TradeAmt` is cross-checked server-side anyway. |
| Try to forge a `ReturnURL` POST | Blocked by `CheckMacValue` verification — requires `HashKey`, which never left Script Properties. |

Worst-case outcome of a leaked link is reputational confusion, **not** financial loss or account takeover. This is why Plan A can safely run out of Apps Script with only a verification-side secret.

### Receipt handling

Same legal posture as Plan B: **no 統一發票**. ECPay is collection agent only; the organizing institute issues an ordinary receipt (一般收據) later. Receipt fields (`抬頭`, `統一編號`, `地址`) are collected on the form and stored in the Sheet — never sent to ECPay.

## Key Findings (ECPay Payment Link specifics)

- Payment Link URL format: `https://payment.ecpay.com.tw/SP/SPCheckOut?SPToken=<token>`
- Appending `&CustomField1=<value>` is supported; ECPay URL-decodes and echoes it in the callback POST body.
- `ReturnURL` (server-to-server) must respond with plain text `1|OK` after valid signature verification; otherwise ECPay retries for ~72 hours.
- `CheckMacValue` on the callback is still computed with the merchant's `HashKey` / `HashIV` — **we do need to verify it** (SHA256 is available in Apps Script via `Utilities.computeDigest(SHA_256, ...)`).
- Amount tampering defense: the callback carries `TradeAmt`. Even if the user hand-edits the link's amount, the callback reports what was actually charged — we compare against the ticket-type price stored server-side.
- Link expiry is configurable in the ECPay backend (recommend 7 days); expired links redirect the user to a page we cannot style, so we proactively re-issue via the daily cron before expiry.

## Questions

> Cross off once resolved and note the decision made.

### Decided

- [x] ~~Which plan?~~ **Candidate Plan A** — pending final confirmation vs. Plan B
- [x] ~~Support English payment flow?~~ **Yes** — bilingual on our own pages; ECPay cashier is Chinese-only, English walkthrough provided with screenshots
- [x] ~~Accept on-site late payment?~~ **Yes, but not via ECPay** — bank transfer / cash, recorded manually
- [x] ~~How do we bind an ECPay callback to a specific attendee?~~ **`OrderID` passed via `CustomField1`; verified against Sheet row + `TradeAmt` + `CheckMacValue`**

### Open

- [ ] Final choice: Plan A (this) vs. Plan B (`PLAN.payment-ecpay.md`)
- [ ] Legal: collecting entity qualifies for 代收代付免開統一發票 — requires accounting sign-off
- [ ] Collecting entity / issuing body: "Taiwan Net Zero Resilient Supply Chain Alliance" vs. NTHU unit
- [ ] Receipt template (serial format, chop, signer, bilingual labels)
- [ ] Receipt issue timing: per-payment vs. post-event batch
- [ ] Refund policy (deadline, ratios)
- [ ] Student verification: student ID upload vs. institutional email check
- [ ] Apps Script Gmail daily quota (100/day for consumer accounts, 1,500/day for Workspace) — confirm which account owns the project

## Scope

### In scope

- `tswim2026.github.io` Registration section redesign (bilingual, pricing, Register button)
- Google Form (embedded in `register.html` via iframe) or custom HTML form POSTing to an Apps Script Web App
- One Apps Script project with three triggers (`onFormSubmit`, `doPost`, daily cron)
- ECPay payment links (one per paid ticket type), configured once in the vendor backend
- `CheckMacValue` verification in Apps Script (SHA256)
- Reconciliation by `OrderID` via `CustomField1`
- Bilingual automated emails (registration received, payment link, payment success, reminder, D-2 check-in)
- Ordinary receipt (一般收據) generation from a Google Docs template → PDF → Gmail
- English payment walkthrough page (with screenshots)
- On-site check-in roster export (with QR codes)

### Out of scope

- Dynamic or per-attendee pricing (only 3 fixed ticket types)
- Cloudflare Worker / any non-Google backend
- AIO CheckMacValue **generation** (we still verify callbacks; we never generate)
- Automated refunds (handled manually in ECPay backend)
- On-site late payment through ECPay (bank transfer / cash → manually recorded in Sheet)
- Multi-currency / overseas card optimization
- ECPay 統一發票 / e-invoice integration

### Backend changes (Apps Script)

Single project, bound to the registrations Sheet. Files:

- `Config.gs` — ticket-type price table, ECPay `HashKey` / `HashIV` (Script Properties), sender addresses, sheet names
- `OrderId.gs` — `generateOrderId()` returns `TSWIM2026-<8-hex>`
- `OnSubmit.gs` — Form trigger: validate row, allocate `OrderID`, write status=`pending`, email payment link with `CustomField1=<OrderID>`
- `Callback.gs` — `doPost(e)`: parse `e.parameter`, verify `CheckMacValue`, look up row by `OrderID`, assert `TradeAmt` equals expected price, set `Status=paid` / `PaidAt`, return `ContentService.createTextOutput('1|OK')`
- `CheckMac.gs` — `verify(params, hashKey, hashIV)`; sort keys, URL-encode per ECPay spec, SHA256 uppercase
- `Cron.gs` — daily: expire `pending` rows past N days, send reminder emails, re-issue payment links where appropriate
- `Receipts.gs` — render Google Docs template → PDF → email; write `ReceiptSerial` / `ReceiptIssuedAt` back to Sheet
- `Tests.gs` — offline unit tests callable from the editor (fixtures include the ECPay official sample values)

Script Properties: `ECPAY_HASH_KEY`, `ECPAY_HASH_IV`, `ECPAY_MERCHANT_ID`, `PAYMENT_LINK_STUDENT`, `PAYMENT_LINK_FACULTY`, `RECEIPT_DOC_TEMPLATE_ID`, `SENDER_NAME`.

### Frontend changes

- `index.html:283-320`:
  - Change "To be announced" → "Open"; Register button links to `/register.html`
  - Remove `d-none` from pricing block; render bilingual labels
- New `register.html`: bilingual form. Options:
  - **Option 1 (fastest)**: embed a Google Form via iframe with pre-filled bilingual labels. Apps Script's `onFormSubmit` picks up rows directly from the bound Sheet.
  - **Option 2 (nicer UX)**: custom HTML form → `fetch` POST to Apps Script Web App `doPost` with a distinct route (e.g. `?action=register`); same server code returns a JSON OK and redirects to a "check your email" page.
  - Fields: contact (name, email, affiliation, notes), ticket type (radio; price server-side), receipt block (`Individual` vs `Organization`, optional `統一編號`, `抬頭`, `地址`, receipt email).
- New `payment-guide-en.html`: English walkthrough derived from the 8 slides of the prior SOP deck.
- New `payment-result.html`: shown after returning from ECPay; explains that the confirmation email is authoritative and that virtual-ATM payments take up to 1 business day.

## Tasks

> **Test-first**: write failing tests (1a, 1b…) before the implementation that turns them green. Apps Script tests live in `Tests.gs` and are run manually from the editor.

### Slice 1 — Apps Script backend core

- [ ] 1.1a Unit test: `generateOrderId()` format `TSWIM2026-[0-9a-f]{8}`, no collisions over 10k iterations
- [ ] 1.1b Unit test: `CheckMac.verify` accepts ECPay official sample payload, rejects one-byte mutation
- [ ] 1.2 Implement `CheckMac.gs` and `OrderId.gs`
- [ ] 1.3a `doPost` tests: invalid signature → response is not `1|OK`; valid but unknown `OrderID` → not `1|OK` + logged; valid + known `OrderID` but `TradeAmt` mismatch → row flagged `amount_mismatch`, not `paid`; valid + matching → row updated, response is `1|OK`; duplicate valid callback → idempotent (no double-update)
- [ ] 1.3b `onFormSubmit` tests: well-formed row → `pending` row created, single email sent, link contains `CustomField1=<OrderID>`; free-ticket row → confirmation email only, no payment link, status `confirmed`; unknown ticket type → row flagged `invalid`, admin alert
- [ ] 1.4 Implement `OnSubmit.gs` and `Callback.gs`
- [ ] 1.5 Deploy Web App (execute as: me; access: anyone, even anonymous) — URL goes into ECPay `ReturnURL` / `OrderResultURL`
- [ ] 1.6 Generate the 2 paid payment links in ECPay backend (student 1200, faculty 2000), store URLs in Script Properties

### Slice 2 — Frontend form & result pages

- [ ] 2.1 Decide Option 1 (Google Form iframe) vs. Option 2 (custom form + Web App)
- [ ] 2.2 Build `register.html` (bilingual, responsive)
- [ ] 2.3 Update `index.html` Registration section (bilingual, show pricing)
- [ ] 2.4 Build `payment-result.html` + `payment-guide-en.html`
- [ ] 2.5 Add `#registration` anchor to nav

### Slice 3 — Data & notifications

- [ ] 3.1 Sheet schema: `OrderID / Name / Email / Affiliation / Role / Banquet / Amount / Status / CreatedAt / PaidAt / ReceiptType / TaxID / ReceiptTitle / ReceiptAddr / ReceiptEmail / ReceiptSerial / ReceiptIssuedAt / Notes`
- [ ] 3.2 Email templates (zh/en): registration received / payment link / payment success / reminder / D-2 check-in
- [ ] 3.3 Daily cron: flag unpaid orders past deadline, send reminder, expire past N days
- [ ] 3.4 Admin alert email on `amount_mismatch` / `invalid` rows

### Slice 4 — Ordinary receipt (一般收據) generation

- [ ] 4.0 Unit test: receipt-data validation — 統編 must be 8 digits if present; individual orders allow blank 統編
- [ ] 4.1 Design Google Docs receipt template (institute chop, serial `TSWIM2026-0001`, bilingual labels, amount in 大寫)
- [ ] 4.2 Implement `Receipts.renderAndSend(orderId)` — copy template → replace placeholders → export PDF → email → stamp `ReceiptSerial` / `ReceiptIssuedAt`
- [ ] 4.3 Post-event batch runner (menu item in the Sheet): allocate serials sequentially for all `Status=paid` rows with empty `ReceiptSerial`
- [ ] 4.4 On-site paper receipt workflow (print from same template by OrderID)

### Slice 5 — Testing & launch

- [ ] 5.1 Staging: point Web App at ECPay staging MerchantID `2000132`; run credit card + virtual ATM end-to-end; verify `CustomField1` round-trip
- [ ] 5.2 UAT: one order per role type × (individual receipt / org receipt with 統編)
- [ ] 5.3 Swap in production `HashKey` / `HashIV` / payment link URLs in Script Properties
- [ ] 5.4 Announcement + open registration

## Trade-offs vs. Plan B

| Concern | Plan A (this) | Plan B (`PLAN.payment-ecpay.md`) |
|---|---|---|
| Where code lives | One Apps Script project (~200 lines) | Cloudflare Worker + TS + tests (~500+ lines) |
| Signing | ECPay generates links; we only verify callbacks | We sign every request and verify callbacks |
| Price flexibility | Fixed set of pre-generated links | Arbitrary amounts per order |
| Secrets surface | Script Properties (Google account) | Worker env + Google Sheet webhook |
| Estimated build time | 1–2 days | 1–2 weeks |
| Failure modes | Gmail quota, Apps Script execution limits (6 min / 30 min) | Worker CPU limits, KV/Sheets rate limits |
| Observability | Apps Script execution log | Worker `wrangler tail`, custom logging |
| Reversibility | High — delete the Apps Script project | Medium — requires decommissioning Worker + DNS |

## References

- Prior flow: `../韌性與管理國際研討會線上繳費SOP.pptx` (link → click → fill → pay → success email)
- ECPay vendor backend: <https://vendor.ecpay.com.tw/> (credentials in `../todo/payment.md`)
- ECPay developer portal: <https://developers.ecpay.com.tw/>
- Test information: <https://developers.ecpay.com.tw/?p=2856>
- CheckMacValue algorithm references (verification only in this plan):
  - [ECPay CheckMacValue generator (Gist)](https://gist.github.com/liaosankai/7aada599848ad599529fd5fdfa7926e6)
- Apps Script references:
  - [Web Apps (`doPost`)](https://developers.google.com/apps-script/guides/web)
  - [Simple / installable triggers](https://developers.google.com/apps-script/guides/triggers)
  - [`Utilities.computeDigest` (SHA-256)](https://developers.google.com/apps-script/reference/utilities/utilities#computedigestalgorithm,-value)
  - [Quotas](https://developers.google.com/apps-script/guides/services/quotas)
