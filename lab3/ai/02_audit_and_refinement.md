# Аудит та виправлення діаграми (AI Trace 02)

## Prompt
Первинна діаграма не відповідає вимогам Завдання 2 та ER-моделі (`lab1/model.mmd`).
Виконай виправлення:
1. Розбий діаграму на три окремі (для кожного Gherkin-сценарію UC-01: успішна купівля, нестача коштів, предмет уже продано).
2. Додай лінії `activate`/`deactivate` для всіх системних сервісів, що охоплюють фрагменти `alt`, `opt`, `break`. Актор `Player` без активації.
3. Узгодь із ER-моделлю: використовуй `wallet_balance` у `Player`, оновлюй `player_id` у `InventoryItem` разом із вилученням предмета з активних лотів (`updateItemOwnerAndDelist`), створюй запис `MarketTransaction(sellerId, buyerId, itemId, price)`.
4. Додай компенсацію `refundFunds` у Сценарій 1 для забезпечення атомарності (NFR-02).
5. У Сценарії 2 оновлюй баланс лише через синхронний виклик `confirmPayment` від `PaymentGateway` до `WalletService` (REQ-05).
6. У Сценарії 3 використай фрагмент `break` при статусі `SOLD` без списання коштів (REQ-03).

## Дослівна відповідь AI
Нижче наведено три окремі Sequence-діаграми для кожного сценарію UC-01 з урахуванням смуг активації, моделі даних та вимог NFR-02, REQ-05 і REQ-03:

```mermaid
%% Сценарій 1: Успішна купівля предмета та атомарність транзакції (NFR-02)
sequenceDiagram
    autonumber
    actor Player
    participant UI as MarketplaceUI
    participant TS as TransactionService
    participant IS as InventoryService
    participant WS as WalletService

    Player->>UI: clickBuy(itemId, price)
    UI->>TS: processPurchase(buyerId, itemId, price)
    activate TS

    TS->>IS: checkItemStatus(itemId)
    activate IS
    IS-->>TS: status(ACTIVE, sellerId)
    deactivate IS

    TS->>WS: checkBalance(buyerId, price)
    activate WS
    WS-->>TS: balanceOk(true)
    deactivate WS

    TS->>WS: deductFunds(buyerId, price)
    activate WS
    WS-->>TS: fundsDeducted(true)
    deactivate WS

    TS->>IS: updateItemOwnerAndDelist(itemId, buyerId)
    activate IS
    alt Передача player_id успішна та предмет зникає з активних лотів
        IS-->>TS: ownerUpdatedAndDelisted(true)
        TS->>TS: createMarketTransaction(sellerId, buyerId, itemId, price)
        TS-->>UI: purchaseCompleted(transactionId)
        UI-->>Player: showSuccess("Предмет придбано та знято з лотів")
    else Збій оновлення InventoryItem (NFR-02)
        IS-->>TS: updateFailed(error)
        TS->>WS: refundFunds(buyerId, price)
        activate WS
        WS-->>TS: refundConfirmed(true)
        deactivate WS
        TS-->>UI: purchaseAborted("Помилка, кошти повернуто")
        UI-->>Player: showError("Купівлю скасовано")
    end
    deactivate IS

    deactivate TS

%% Сценарій 2: Недостатньо коштів та поповнення через шлюз (REQ-05)
sequenceDiagram
    autonumber
    actor Player
    participant UI as MarketplaceUI
    participant TS as TransactionService
    participant IS as InventoryService
    participant WS as WalletService
    participant PG as PaymentGateway

    Player->>UI: clickBuy(itemId, price)
    UI->>TS: processPurchase(buyerId, itemId, price)
    activate TS

    TS->>IS: checkItemStatus(itemId)
    activate IS
    IS-->>TS: status(ACTIVE, sellerId)
    deactivate IS

    TS->>WS: checkBalance(buyerId, price)
    activate WS
    WS-->>TS: balanceInsufficient(deficit)
    deactivate WS

    TS-->>UI: rejectInsufficientFunds(deficit)
    deactivate TS

    opt Поповнення балансу (extend)
        UI-->>Player: promptTopUp(deficit)
        Player->>UI: requestTopUp(deficit)
        UI->>PG: createPaymentSession(buyerId, deficit)
        activate PG
        PG-->>UI: paymentPageReady(sessionId)
        Player->>PG: submitCardPayment()
        PG->>WS: confirmPayment(buyerId, deficit)
        activate WS
        WS->>WS: credit_wallet_balance(buyerId, deficit)
        WS-->>PG: paymentAck(OK)
        deactivate WS
        PG-->>UI: paymentSucceeded()
        deactivate PG
        UI-->>Player: showBalanceUpdated("Баланс поповнено")
    end

%% Сценарій 3: Предмет вже продано (REQ-03, break)
sequenceDiagram
    autonumber
    actor Player
    participant UI as MarketplaceUI
    participant TS as TransactionService
    participant IS as InventoryService

    Player->>UI: clickBuy(itemId, price)
    UI->>TS: processPurchase(buyerId, itemId, price)
    activate TS

    TS->>IS: checkItemStatus(itemId)
    activate IS
    IS-->>TS: status(SOLD, sellerId)
    deactivate IS

    break Статус лота == SOLD
        TS-->>UI: rejectLotSold("Предмет вже продано")
        UI-->>Player: showError("Предмет недоступний")
    end

    deactivate TS
```