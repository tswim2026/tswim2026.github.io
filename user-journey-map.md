## User Journey Map

> **Legend**: nodes with a dashed red border are steps whose design is still **pending discussion** and may change before launch.
>
> Two plans are on the table. Attendee-facing flows are nearly identical; the internal / organizer flows differ. Sections below cover:
>
> 1. Main flow (attendee) — shared by both plans
> 2. On-site fallback — shared by both plans
> 3. Organizer back-office — Plan B (full AIO integration, `PLAN.payment-ecpay.md`)
> 4. **Plan A variant** — Apps Script + ECPay Payment Link (`PLAN.payment-appsscript.md`): internal flow and per-attendee binding

### Main flow — online registration & payment

```mermaid
flowchart TD
    A["Visit TSWIM 2026 website"] --> B["Click Register (報名)"]
    C0["Fill in bilingual form:<br/>name, email, affiliation,<br/>receipt title (收據抬頭), tax ID (統一編號, optional),<br/>address (地址, optional)"]
    B --> C0
    C0 --> C1{"Select ticket type (票種)"}
    C1 -->|Student + banquet 學生含晚宴| P1["NT$1,200"]
    C1 -->|Faculty + banquet 教師含晚宴| P2["NT$2,000"]
    C1 -->|No banquet 不含晚宴| P3["Free — no payment needed"]
    P1 --> D["Submit → total is fixed and sent to ECPay"]
    P2 --> D
    P3 --> J["Receive registration confirmation email<br/>(skip payment)"]
    D --> E{"Choose payment method (付款方式)"}
    E -->|Credit Card 信用卡| F["Enter card details"]
    E -->|Virtual ATM 網路 ATM| G["Get virtual account number (虛擬帳號)"]
    F --> H["Payment confirmed"]
    G --> I["Transfer at ATM / online banking<br/>within the deadline"]
    I --> H
    H --> J["Receive payment success email"]
    J --> K["Receive ordinary receipt (一般收據) by email"]:::tbd
    K --> L["Receive pre-event check-in instructions"]
    L --> M["On-site QR check-in (現場報到) → get badge"]

    classDef tbd stroke-dasharray: 5 5,stroke:#c0392b,color:#c0392b;
```

**Still pending discussion:**

- `K` — when the receipt arrives (shortly after payment, or after the event) and exactly what the receipt looks like.

### Fallback — on-site late payment

```mermaid
flowchart LR
    A["Arrive at check-in desk (現場報到櫃台)"] --> B{"Already paid? (是否已繳費)"}
    B -->|Yes| C["Scan QR → get badge (發名牌)"]
    B -->|No| D["Pay on-site (現場補繳)<br/>bank transfer (匯款) / cash (現金)"]:::tbd
    D --> E["Receive paper receipt (紙本收據)<br/>with your title (抬頭) / tax ID (統編) if any"]:::tbd
    E --> C

    classDef tbd stroke-dasharray: 5 5,stroke:#c0392b,color:#c0392b;
```

**Still pending discussion:**

- `D` — which on-site payment methods are accepted.
- `E` — paper receipt template / who signs it.

### Organizer back-office view

```mermaid
flowchart TD
    S["Review registration list daily"] --> T{"Payment overdue? (逾期未繳)"}
    T -->|Yes| U["Send reminder email (催款信)"]
    T -->|No| V["No action"]
    S --> W["Reconcile payments (對帳)"]
    W --> X["Issue ordinary receipts (一般收據)<br/>and email attendees"]:::tbd
    X --> Y["A week before the event:<br/>export check-in list (報到名單) + QR codes"]

    classDef tbd stroke-dasharray: 5 5,stroke:#c0392b,color:#c0392b;
```

**Still pending discussion:**

- `X` — whether receipts go out **right after each payment** or **as one batch after the event**.

### Plan A variant — Apps Script + ECPay Payment Link (internal flow)

> See `PLAN.payment-appsscript.md`. The attendee still experiences the Main flow above; this diagram shows what happens **inside the system** for Plan A, and how one specific attendee is identified across the payment round-trip.

```mermaid
flowchart TD
    subgraph Attendee
      A1["Submit registration form<br/>(register.html)"]
      A2["Receive bilingual email<br/>with personalized payment link"]
      A3["Click link → ECPay Cashier<br/>(Chinese UI; EN guide linked)"]
      A4["Pay by credit card or virtual ATM"]
      A5["Receive payment success email<br/>+ receipt (一般收據) PDF"]
    end

    subgraph AppsScript["Google Apps Script (single project)"]
      B1["onFormSubmit trigger"]
      B2["Generate OrderID<br/>TSWIM2026-&lt;8-hex&gt;"]
      B3["Write Sheet row<br/>Status = pending"]
      B4["Email payment link with<br/>?CustomField1=&lt;OrderID&gt;"]
      B5["doPost (Web App)<br/>= ECPay ReturnURL"]
      B6{"Verify<br/>CheckMacValue?"}
      B7{"OrderID found<br/>in Sheet?"}
      B8{"TradeAmt matches<br/>ticket-type price?"}
      B9["Update row:<br/>Status = paid, PaidAt = now"]
      B10["Respond '1|OK'"]
      B11["Flag row:<br/>amount_mismatch / unknown"]:::tbd
      B12["Daily cron:<br/>expire / remind / reissue"]
    end

    subgraph ECPay
      C1["Pre-generated Payment Link<br/>per ticket type (fixed amount)"]
      C2["Cashier collects payment"]
      C3["Server-to-server POST<br/>to ReturnURL"]
    end

    A1 --> B1 --> B2 --> B3 --> B4 --> A2
    A2 --> A3 --> C1 --> C2 --> A4
    C2 --> C3 --> B5 --> B6
    B6 -- no --> B11
    B6 -- yes --> B7
    B7 -- no --> B11
    B7 -- yes --> B8
    B8 -- no --> B11
    B8 -- yes --> B9 --> B10 --> C3
    B9 --> A5
    B12 -.-> A2

    classDef tbd stroke-dasharray: 5 5,stroke:#c0392b,color:#c0392b;
```

**Key points:**

- **Binding**: `OrderID` is generated by Apps Script at form submit and passed to ECPay via `CustomField1`, which ECPay echoes back unchanged in the callback. This is the only reliable key — neither email, name, nor amount alone can identify the attendee.
- **Three defenses** against spoofed / tampered callbacks, all required: (1) `CheckMacValue` matches `HashKey` / `HashIV`; (2) `OrderID` exists in Sheet with `Status=pending`; (3) `TradeAmt` equals the server-side price for that ticket type. Any failure routes to `amount_mismatch` / `unknown` rather than `paid`.
- **Idempotency**: a valid callback for an already-`paid` row is a no-op; ECPay retries are safe.
- **No AIO signing code** — ECPay generates the payment link; we only verify the callback signature.

**Still pending discussion:**

- `B11` — exact operator workflow for anomalous callbacks (email alert vs. Sheet filter view).
- Receipt arrival (`A5`) timing — same open question as the shared back-office view: per-payment or post-event batch.
