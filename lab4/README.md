# Лабораторна робота №4: Моделювання бізнес-процесу (Activity Diagram)

## Артефакти роботи
- [`spec.md`](./spec.md) — специфікація бізнес-процесу `UC-01 «Купівля предмета»`, ролі, повідомлення, обґрунтування нотації та критерії прийняття (`AC-01`..`AC-06`).
- [`process.mmd`](./process.mmd) — декларативний опис Activity-діаграми у синтаксисі Mermaid `flowchart`.
- [`adr/0001-activity-diagram-and-payment-timeout.md`](./adr/0001-activity-diagram-and-payment-timeout.md) — архітектурне рішення (MADR) щодо вибору Activity-діаграми, обробки тайм-ауту шлюзу оплати та синхронізації `fork/join`.
- [`ai/`](./ai/) — слід роботи з AI: інтерв'ю та дозбір вимог (`01_initial_generation.md`) і аудит із виправленням топ-3 розбіжностей (`02_audit_and_refinement.md`).
- [`DEFENSE.md`](./DEFENSE.md) — документ захисту роботи за шаблоном DoD §4.

## Рендер діаграми діяльності (UC-01 «Купівля предмета»)

```mermaid
flowchart TD
    subgraph Player ["Гравець — Player"]
        Start((Початок)) --> InitBuy["Ініціювати купівлю предмета clickBuy"]
        DecideTopUp{"Поповнити баланс на суму дефіциту?"}
        SubmitCard["Ввести платіжні дані та підтвердити submitCardPayment"]
        RecvSold["Отримати повідомлення про недоступність лота"]
        RecvCancel["Отримати повідомлення про скасування купівлі"]
        RecvRefund["Отримати повідомлення про скасування та повернення коштів"]
        RecvSuccess["Отримати підтвердження купівлі предмета"]
        End((Кінець))
    end

    subgraph Marketplace ["Система маркетплейсу — Transaction / Inventory / Wallet"]
        CheckLot["Перевірити статус лота checkItemStatus"]
        DataItem[/"Об'єкт даних ER: InventoryItem"/]
        IsLotActive{"Статус лота в інвентарі?"}
        SigLotSold>"Надіслати відмову rejectLotSold"]

        CheckBal["Перевірити баланс гаманця checkBalance"]
        DataPlayer[/"Об'єкт даних ER: Player wallet_balance"/]
        IsBalEnough{"Коштів на балансі достатньо?"}
        SigDeficit>"Надіслати запит поповнення rejectInsufficientFunds / promptTopUp"]
        SigCreateSession>"Надіслати запит сесії createPaymentSession"]

        WaitPayment(("Очікувати подію оплати або тайм-аут 15 хв"))
        AcceptPayEvent>"Прийняти подію від шлюзу або таймера"]
        PayDecision{"Яка подія настала?"}
        SigPayAbort>"Надіслати повідомлення про тайм-аут чи відхилення оплати"]
        CreditBal["Зарахувати кошти на баланс creditWalletBalance"]

        DeductBal["Списати кошти з балансу deductFunds"]

        ForkBar["=== FORK: Паралельне проведення угоди ==="]
        TransferItem["Передати право власності та зняти лот updateItemOwnerAndDelist"]
        CreateTx["Зафіксувати запис угоди createMarketTransaction"]
        DataTx[/"Об'єкт даних ER: MarketTransaction"/]
        JoinBar["=== JOIN: Синхронізація завершення гілок allSettled ==="]

        IsCommitOk{"Обидві операції виконано без збоїв?"}
        RefundBal["Повернути списані кошти на баланс refundFunds"]
        SigRollback>"Надіслати сповіщення про відкат purchaseAborted"]
        SigSuccess>"Надіслати підтвердження угоди purchaseCompleted"]
    end

    subgraph Gateway ["Платіжний шлюз — PaymentGateway"]
        OpenSession["Створити платіжну сесію та форму оплати paymentPageReady"]
        ProcessCard["Обробити транзакцію за карткою гравця"]
        SigConfirmPay>"Надіслати подію результату оплати confirmPayment"]
    end

    InitBuy --> CheckLot
    CheckLot -.-> DataItem
    CheckLot --> IsLotActive

    IsLotActive -->|"[SOLD — Сценарій 3 REQ-03]"| SigLotSold
    SigLotSold --> RecvSold --> End

    IsLotActive -->|"[ACTIVE — лот доступний]"| CheckBal
    CheckBal -.-> DataPlayer
    CheckBal --> IsBalEnough

    IsBalEnough -->|"[Ні — дефіцит коштів Сценарій 2]"| SigDeficit
    SigDeficit --> DecideTopUp
    DecideTopUp -->|"[Ні — гравець відмовився]"| RecvCancel --> End
    DecideTopUp -->|"[Так — поповнити баланс extend]"| SigCreateSession

    SigCreateSession --> OpenSession --> SubmitCard --> ProcessCard --> SigConfirmPay
    SigConfirmPay --> WaitPayment
    WaitPayment --> AcceptPayEvent --> PayDecision

    PayDecision -->|"[Тайм-аут 15 хв або відмова шлюзу]"| SigPayAbort
    SigPayAbort --> RecvCancel

    PayDecision -->|"[Отримано confirmPayment OK REQ-05]"| CreditBal
    CreditBal --> DeductBal

    IsBalEnough -->|"[Так — коштів достатньо Сценарій 1]"| DeductBal
    DeductBal --> ForkBar

    ForkBar --> TransferItem
    ForkBar --> CreateTx
    TransferItem -.-> DataItem
    CreateTx -.-> DataTx
    TransferItem --> JoinBar
    CreateTx --> JoinBar

    JoinBar --> IsCommitOk
    IsCommitOk -->|"[Ні — збій оновлення NFR-02]"| RefundBal --> SigRollback --> RecvRefund --> End
    IsCommitOk -->|"[Так — успішно]"| SigSuccess --> RecvSuccess --> End
```