# Первинна генерація Sequence-діаграми (AI Trace 01)

## Prompt
На основі специфікації `lab3/spec.md` та прецеденту UC-01 «Купівля предмета» побудуй Sequence-діаграму у форматі Mermaid (`sequenceDiagram`). Учасники: `Player`, `MarketplaceUI`, `TransactionService`, `WalletService`, `InventoryService`, `PaymentGateway`. Покажи перевірку балансу через `alt` та опціональне поповнення через `opt`.

## Дослівна відповідь AI
Нижче наведено Sequence-діаграму у форматі Mermaid для сценарію купівлі предмета на маркетплейсі з використанням блоків `alt` та `opt`:

```mermaid
sequenceDiagram
    autonumber
    actor Player
    participant UI as MarketplaceUI
    participant TS as TransactionService
    participant WS as WalletService
    participant IS as InventoryService
    participant PG as PaymentGateway

    Player->>UI: initiatePurchase(itemId, itemPrice)
    UI->>TS: processPurchase(playerId, itemId, itemPrice)
    TS->>WS: checkBalance(playerId, itemPrice)
    WS-->>TS: balanceStatus(isEnough)

    alt isEnough == true (Достатньо коштів)
        TS->>WS: deductFunds(playerId, itemPrice)
        WS-->>TS: fundsDeducted(true)
        TS->>IS: transferOwnership(itemId, playerId)
        IS-->>TS: ownershipTransferred(true)
        TS-->>UI: purchaseSuccess(transactionDetails)
        UI-->>Player: showSuccessMessage("Предмет успішно придбано!")
    else isEnough == false (Недостатньо коштів)
        TS-->>UI: purchaseFailed("Insufficient funds")
        UI-->>Player: showErrorMessage("Недостатньо коштів на балансі")

        opt Гравець вирішує поповнити баланс (extend)
            Player->>UI: selectTopUp(missingAmount)
            UI->>PG: processPayment(playerId, missingAmount)
            PG-->>UI: paymentConfirmed(token)
            UI->>WS: addFunds(playerId, missingAmount)
            WS-->>UI: balanceUpdated(newBalance)
            UI-->>Player: showTopUpSuccess("Баланс поповнено, спробуйте ще раз")
        end
    end
```