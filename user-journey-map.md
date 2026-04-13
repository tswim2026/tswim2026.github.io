## User Journey Map

> **Legend**: nodes with a dashed red border are steps whose design is still **pending discussion** and may change before launch.

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
