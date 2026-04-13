# TSWIM 2026 Registration & Payment Integration (ECPay AIO)

> **IMPORTANT**: This plan is the single source of truth for the current state of this work. Update it before and after every task. Context may be cleared at any time.

## Branch

`feat/payment-ecpay`

## Goal

Deliver end-to-end online registration and payment on `tswim2026.github.io` (bilingual, zh/en) using **Plan B — full ECPay AIO integration**. Paid attendees flow directly from the site to ECPay and back, with results auto-recorded to a shared Google Sheet for the organizing team. On-site late payment is supported but **does not go through ECPay** — handled by staff via bank transfer or cash.

## Strategy: Vertical Slice

One branch = one complete, testable path (registration form → ECPay payment → callback recorded → confirmation email):

1. **Backend (Cloudflare Worker) tests** — mock ECPay POSTs; validate CheckMacValue generation and verification
2. **Backend implementation** — `/create-order` and `/ecpay-callback` endpoints
3. **Frontend integration** — update `index.html` Registration section; add bilingual forms and result pages
4. **End-to-end verification** — run credit card + virtual ATM flows against ECPay staging (MerchantID `2000132`)

## Related Documents

> When updating, write changes in the correct file — do **not** duplicate content across files.

- `PLAN.payment-ecpay.md` (this file) — scope, decisions, tasks, findings
- `user-journey-map.md` — all user journey Mermaid flowcharts (main flow, on-site fallback, organizer back-office). Add / edit flowcharts there, not here.

## Current State

- [x] Requirements interview (2026-04-13)
- [x] Prior SOP reviewed (`韌性與管理國際研討會線上繳費SOP.pptx`)
- [x] Plan created and moved into site repo on branch `feat/payment-ecpay`
- [ ] Open questions resolved (refund policy, collecting-entity name)
- [ ] Production ECPay HashKey / HashIV obtained
- [ ] Backend prototype
- [ ] Frontend prototype
- [ ] End-to-end staging test passing
- [ ] Production launch

## Key Findings

- **Site**: `tswim2026.github.io` is a static GitHub Pages site. `index.html:283-289` already has a Registration card (status: "To be announced"). `index.html:291-320` has a hidden pricing block:
  - Student (banquet included) — NT$1,200
  - Faculty (banquet included) — NT$2,000
  - No banquet — Free
- **Prior SOP (semi-automated)**: email payment link → ECPay cashier page → credit card / virtual ATM → confirmation email. Collecting entity was "Taiwan Net Zero Resilient Supply Chain Alliance". *Whether to continue using this entity is an open question.*
- **ECPay staging** ([ECPay Developers — Test Information](https://developers.ecpay.com.tw/?p=2856))
  - Staging endpoint: `https://payment-stage.ecpay.com.tw/Cashier/AioCheckOut/V5`
  - Production endpoint: `https://payment.ecpay.com.tw/Cashier/AioCheckOut/V5`
  - Test MerchantID: `2000132`, HashKey: `5294y06JbISpM5x9`, HashIV: `v77hoKGq4kWxNNIS`
- **CheckMacValue algorithm**: sort params A–Z → prepend `HashKey=…&` and append `&HashIV=…` → URL-encode (preserve `-_.!*()`, lowercase) → SHA256 → uppercase; `EncryptType=1`.
- **ReturnURL (server-to-server)**: must respond with plain text `1|OK` after valid signature check, otherwise ECPay will retry.
- **ECPay account already activated** (see `../todo/payment.md`) — only missing real HashKey/HashIV and callback URL configuration.
- **No tax invoices (統一發票) — ordinary receipt (一般收據) only**. The organizing institute (non-profit / academic unit) is not a 營業人 and therefore does **not** issue 統一發票 via ECPay. Implications for the AIO integration:
  - Set **`InvoiceMark=N`** on the AIO CheckOut V5 form (or omit all invoice extension params); ECPay will act as a collection agent only and will not auto-issue any tax invoice.
  - We still collect each attendee's **統一編號 (optional, 8 digits)**, **抬頭**, **地址** on the registration form — these are used by the organizing institute to produce its own 一般收據 (PDF / paper), not sent to ECPay.
  - Legal framing: see 財政部「代收代付免開統一發票」規定 — the collecting entity (e.g., the supply-chain alliance or NTHU unit) must meet the exemption conditions, otherwise legally it still needs to issue 發票. **Confirmation with finance/accounting is required before launch.**
  - Receipt delivery: PDF emailed post-event by the organizing institute; on-site paper receipt available on request.
  - Source: [財政部 — 代收代付免開統一發票](https://www.mof.gov.tw/singlehtml/384fb3077bb349ea973e7fc6f13b6974?cntId=c89c8997fa9c43e4a1a6dffb8e7ed4da), [ECPay 技術問題 FAQ](https://www.ecpay.com.tw/CascadeFAQ/CascadeFAQ_Qa?nID=4372)

## Questions

> Cross off once resolved and note the decision made.

### Decided

- [x] ~~Which plan?~~ **Plan B (full AIO integration)**
- [x] ~~Support English payment flow?~~ **Yes** — bilingual on our own pages; ECPay cashier is Chinese-only, so we provide an English walkthrough with screenshots
- [x] ~~Accept on-site late payment?~~ **Yes, but not via ECPay** — handled as bank transfer / cash by staff, recorded manually

### Receipt (一般收據)

- [x] ~~Invoice vs receipt?~~ **Decision**: no 統一發票. ECPay acts as collection agent only (`InvoiceMark=N`); the organizing institute issues a **一般收據 (ordinary receipt)** to each paid attendee (PDF by email; optional paper on-site).
- [ ] **Legal check**: confirm the collecting entity qualifies for 代收代付免開統一發票 (財政部). Needs sign-off from accounting before launch — **open**
- [ ] **Collecting entity / issuing body**: continue "Taiwan Net Zero Resilient Supply Chain Alliance" or issue under NTHU name? Affects both reconciliation and whose chop / 抬頭 appears on the 收據 — **open**
- [ ] **Receipt template**: signer, institute chop, bilingual labels, amount-in-國字大寫, serial numbering scheme (e.g. `TSWIM2026-0001`) — **open**
- [ ] **Issue timing**: per-payment auto-issue (Worker triggers PDF + email within minutes of ECPay callback) **vs** post-event batch (one serial run after the event). Per-payment gives better UX but needs serial-allocation concurrency handling — **open**
- [ ] **Confirm**: 統編 / 抬頭 / 地址 fields collected on the form are used only for our 收據 and never sent to ECPay — **open (implementation check)**

### Operational / scope

- [ ] Refund policy (post-early-bird changes / cancellation ratios / deadline) — **open**
- [ ] Student verification: upload student ID online / institutional email verification?

### Technical / infrastructure

- [ ] Backend hosting: Cloudflare Workers (recommended, free tier sufficient) vs. Vercel Functions vs. Google Apps Script
- [ ] Data store: Google Sheet (via Apps Script webhook) vs. Notion DB vs. Supabase

## Scope

### In scope

- `tswim2026.github.io` Registration section redesign (bilingual, pricing, Register button)
- Cloudflare Worker (or equivalent): `/create-order`, `/ecpay-callback`, `/order-status/:id`
- ECPay AIO V5 integration (credit card + virtual ATM)
- Registration form and data landing (Google Sheet for v1)
- Bilingual automated emails (registration received, payment success, virtual ATM details, D-2 reminder)
- Ordinary receipt (一般收據) generation: PDF template filled from Sheet row (name, 抬頭, 統編, amount, date, serial), emailed post-event
- English payment walkthrough page (with screenshots)
- On-site check-in roster export (with QR codes)

### Out of scope

- Automated refunds (handled manually in ECPay backend)
- On-site late payment through ECPay (bank transfer / cash → manually recorded in Sheet)
- Multi-currency / overseas card optimization (stay on ECPay TWD)
- **ECPay 統一發票 / e-invoice integration** (explicitly not used; legally we issue 一般收據 only)

### Backend changes

- New project `tswim-payment-worker/` (Cloudflare Worker + TypeScript)
  - `POST /create-order`: validate fields → generate `MerchantTradeNo` (`TSWIM` + yyyymmdd + sequence) → write Sheet (status=pending) → compute CheckMacValue → return auto-submit HTML form (action → ECPay)
  - `POST /ecpay-callback`: verify CheckMacValue → update Sheet (status=paid / failed) → trigger email → respond `1|OK`
  - `GET /order-status/:id`: polled by frontend to show result
  - Secrets: `ECPAY_MERCHANT_ID` / `ECPAY_HASH_KEY` / `ECPAY_HASH_IV` / `GSHEET_WEBHOOK_URL` / `MAIL_API_KEY`

### Frontend changes

- `index.html:283-320`:
  - Change "To be announced" → "Open"; Register button links to `/register.html`
  - Remove `d-none` from pricing block; render bilingual labels
- New `register.html`: bilingual form → submits to Worker `/create-order`. Fields:
  - Contact: name, email, affiliation, notes
  - **Ticket type** (radio, price computed server-side from this choice alone):
    - Student + banquet — NT$1,200
    - Faculty + banquet — NT$2,000
    - No banquet — Free (bypass ECPay; registration confirmed by email only)
  - Pricing table is the single source of truth in the Worker. The frontend only sends the ticket type code; the Worker looks up `TotalAmount` and passes it to ECPay as a fixed signed field. Users cannot edit the amount anywhere.
  - **Receipt (一般收據) block** — data used by us only, not sent to ECPay:
    - Radio: `個人 (Individual)` vs `公司 / 機構 (Organization)`
    - If `公司 / 機構`: `統一編號 (optional, 8 digits regex `^\d{8}$`)`, `收據抬頭 / Receipt Title`, `地址 / Address (optional)`
    - If `個人`: `收據抬頭` defaults to contact name (editable)
    - Receipt email defaults to the contact email (editable)
- New `payment-guide-en.html`: English walkthrough derived from the 8 slides of the prior SOP deck
- New `payment-result.html`: post-payment page (reads `?MerchantTradeNo=` and polls `/order-status/:id` for ~5s)

## Tasks

> **Test-first**: write failing tests (1a, 1b…) before the implementation that turns them green.

### Slice 1 — Backend core

- [ ] 1.1a Unit test for CheckMacValue generation (using official sample values)
- [ ] 1.1b Unit test for CheckMacValue verification (reject tampered params)
- [ ] 1.2 Implement `checkMacValue.ts`
- [ ] 1.3a `/create-order` tests: valid input returns auto-submit HTML; invalid fields return 400; **unknown ticket type → 400**; **Free ticket → skip ECPay and respond with a registration-only success path**; **client cannot override `TotalAmount` (ignored if sent)**
- [ ] 1.3b `/ecpay-callback` tests: invalid signature returns 400; valid returns `1|OK` and calls Sheet webhook (mocked)
- [ ] 1.4 Implement both endpoints
- [ ] 1.5 Deploy to Cloudflare Workers (staging)
- [ ] 1.6 Configure `ReturnURL` / `OrderResultURL` / `ClientBackURL` in ECPay backend

### Slice 2 — Frontend form & result pages

- [ ] 2.1 Build `register.html` (bilingual, responsive)
- [ ] 2.2 Update `index.html` Registration section (bilingual, show pricing)
- [ ] 2.3 Build `payment-result.html` + `payment-guide-en.html`
- [ ] 2.4 Add `#registration` anchor to nav

### Slice 3 — Data landing & notifications

- [ ] 3.1 Create Google Sheet schema: `OrderID / Name / Email / Affiliation / Role / Banquet / Amount / Status / CreatedAt / PaidAt / ReceiptType (individual|org) / TaxID / ReceiptTitle / ReceiptAddr / ReceiptEmail / ReceiptSerial / ReceiptIssuedAt`
- [ ] 3.2 Apps Script webhook endpoint (called from Worker)
- [ ] 3.3 Email templates (zh/en): registration received / payment success / virtual ATM details / D-2 reminder
- [ ] 3.4 Daily cron: flag unpaid virtual-ATM orders and send reminder

### Slice 4 — Ordinary receipt (一般收據) generation

- [ ] 4.0a Unit test: `/create-order` sends **`InvoiceMark=N`** and omits all ECPay invoice extension params (assert no `CustomerIdentifier` etc. leak into the signed payload)
- [ ] 4.0b Unit test: receipt-data validation — 統編 must be 8 digits if present; individual orders allow blank 統編
- [ ] 4.1 Design 收據 PDF template (institute chop, serial format e.g. `TSWIM2026-0001`, bilingual labels, amount in 大寫)
- [ ] 4.2 Implement receipt generator (Apps Script or Worker → PDF) driven by Sheet rows where `Status=paid` and `ReceiptSerial` is empty
- [ ] 4.3 Post-event batch: allocate serial numbers, render PDFs, email to `ReceiptEmail`, write `ReceiptSerial` + `ReceiptIssuedAt` back to Sheet
- [ ] 4.4 On-site paper receipt workflow (print from same template by OrderID)

### Slice 5 — Testing & launch

- [ ] 5.1 Run credit card + virtual ATM end-to-end on staging MerchantID `2000132`
- [ ] 5.2 Internal UAT: one order per role type × (individual receipt / org receipt with 統編)
- [ ] 5.3 Swap in production credentials (env vars)
- [ ] 5.4 Announcement + open registration


## References

- Prior flow: `../韌性與管理國際研討會線上繳費SOP.pptx` (link → click → fill → pay → success email)
- ECPay vendor backend: <https://vendor.ecpay.com.tw/> (credentials in `../todo/payment.md`)
- ECPay developer portal: <https://developers.ecpay.com.tw/>
- Test information: <https://developers.ecpay.com.tw/?p=2856>
- CheckMacValue algorithm and implementation references:
  - [ECPay CheckMacValue generator (Gist)](https://gist.github.com/liaosankai/7aada599848ad599529fd5fdfa7926e6)
  - [Express/Node.js integration from scratch](https://medium.com/@94jillian/%E7%B6%A0%E7%95%8C%E9%87%91%E6%B5%81%E4%B8%B2%E6%8E%A5-express-node-js-mongodb-atlas-%E4%B8%8D%E7%94%A8%E7%B6%A0%E7%95%8Csdk-%E5%9C%9F%E6%B3%95%E7%85%89%E9%8B%BC%E6%8E%A5%E8%B5%B7%E4%BE%86-731ac2d475af)
  - [Rails integration (Postman tests)](https://medium.com/%E5%AE%B8-%E5%AD%B8%E7%BF%92%E7%AD%86%E8%A8%98/rails-%E4%B8%B2%E6%8E%A5-ecpay-%E7%B6%A0%E7%95%8C%E9%87%91%E6%B5%81-%E4%B8%80-postman-%E6%B8%AC%E8%A9%A6-api-37a34f0a4b6)
  - [AIO full example walkthrough](https://medium.com/@roan6903/ecpay-aioexampple-37073ceeb853)
  - [Flask + Python credit card integration](https://www.maxlist.xyz/2020/02/14/python-ecpay/)
