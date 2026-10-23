## Сценарій 1: Успішна купівля предмета та атомарність транзакції (NFR-02)
```mermaid
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
```

## Сценарій 2: Недостатньо коштів та поповнення через шлюз (REQ-05)
```mermaid
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
```

## Сценарій 3: Предмет уже продано (REQ-03, break)
```mermaid
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
        TS-->>UI: rejectLotSold("Предмет уже продано")
        UI-->>Player: showError("Предмет недоступний")
    end

    deactivate TS
```