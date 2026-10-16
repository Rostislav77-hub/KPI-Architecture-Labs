## 3. EARS Вимоги

### Функціональні вимоги (по сценаріях купівлі та поповнення):
1. **REQ-01 (Успішна купівля):** WHEN the Player clicks "Buy", IF `wallet_balance` >= `price` AND the item status is "Available", the system SHALL deduct the price from `wallet_balance`, change item ownership to the Player, and remove the listing from the market.
2. **REQ-02 (Недостатньо коштів):** WHEN the Player clicks "Buy", IF `wallet_balance` < `price`, the system SHALL reject the transaction, display an "Insufficient funds" error, and offer a wallet top-up option.
3. **REQ-03 (Предмет вже продано):** WHEN the Player clicks "Buy", IF the item status is no longer "Available" (bought by another player), the system SHALL cancel the order and notify the Player that the item is unavailable.
4. **REQ-04 (Перегляд каталогу):** WHILE the Player is browsing the market, the system SHALL display only items with the status "Available".
5. **REQ-05 (Поповнення балансу):** WHEN the Player confirms a deposit amount, the system SHALL redirect the request to the `PaymentGateway` and update `wallet_balance` only after receiving a confirmation callback.
6. **REQ-06 (UI-вимога):** WHERE the skin rarity is "Legendary", the system SHALL highlight its border in gold.

### Нефункціональні вимоги (NFR):
7. **NFR-01 (Продуктивність):** WHEN the Player initiates an item purchase, the system SHALL complete the balance check and ownership transfer within 500 milliseconds.
8. **NFR-02 (Надійність / Атомарність):** WHILE processing a purchase transaction, the system SHALL ensure ACID compliance so that funds are never deducted without transferring the item (preventing race conditions when two players buy the same item simultaneously).
9. **NFR-03 (Безпека):** The system SHALL NOT store raw credit card details locally, delegating all PCI DSS sensitive operations to the external `PaymentGateway`.

## 4. User Stories
- **US-01:** Як Гравець, я хочу купувати предмети на маркеті (та отримувати чіткий зворотний зв'язок у разі нестачі коштів або недоступності лота), щоб безпечно поповнювати свій інвентар.
- **US-02:** Як Гравець, я хочу поповнювати баланс через захищену платіжну систему, щоб мати змогу купувати скіни.
- **US-03:** Як Гравець, я хочу виставляти свої предмети на продаж у каталозі, щоб заробляти кошти на внутрішній баланс.

## 5. Gherkin Сценарії (Feature: Купівля предмета на маркетплейсі)

**Feature:** Купівля предмета (UC-01)
  Як Гравець, я хочу придбати вибраний скін, щоб використовувати його в грі.

  **Scenario 1: Успішна купівля предмета (Happy Path)**
    Given Гравець авторизований і має `wallet_balance` 500 грн
    And Предмет "AK-47 | Slate" має статус "Available" і коштує 300 грн
    When Гравець натискає кнопку "Купити"
    Then Система списує 300 грн, і `wallet_balance` стає 200 грн
    And Предмет "AK-47 | Slate" переходить в інвентар Гравця
    And Предмет зникає зі списку активних лотів маркету

  **Scenario 2: Відмова у купівлі через нестачу коштів (Insufficient Funds)**
    Given Гравець авторизований і має `wallet_balance` 100 грн
    And Предмет "AWP | Asiimov" має статус "Available" і коштує 1500 грн
    When Гравець натискає кнопку "Купити"
    Then Система блокує транзакцію, а `wallet_balance` залишається 100 грн
    And Гравець бачить повідомлення про помилку "Недостатньо коштів на балансі"
    And Система пропонує кнопку "Поповнити баланс" (перехід до UC-03)

  **Scenario 3: Помилка купівлі, якщо предмет щойно купив інший гравець (Race Condition / Item Unavailable)**
    Given Гравець відкрив сторінку предмета "Butterfly Knife", який мав статус "Available"
    And Інший користувач встиг викупити цей предмет секундою раніше (статус змінився на "Sold")
    When Гравець натискає кнопку "Купити"
    Then Система скасовує операцію без списання коштів з `wallet_balance`
    And Гравець бачить сповіщення "На жаль, цей предмет вже продано"

## 6. Матриця трасування (Traceability Matrix)
| Вимога (EARS / NFR) | US-01 (Купівля) | US-02 (Поповнення) | US-03 (Продаж) | Gherkin Сценарій |
| :--- | :---: | :---: | :---: | :--- |
| **REQ-01 (Успіх)** | X | | | Scenario 1 |
| **REQ-02 (Мало коштів)** | X | X | | Scenario 2 |
| **REQ-03 (Вже продано)** | X | | | Scenario 3 |
| **REQ-04 (Каталог)** | X | | X | — |
| **REQ-05 (Депозит)** | | X | | — |
| **REQ-06 (UI Золотий)** | | | | — |
| **NFR-01 (Швидкість <=500ms)**| X | | | Scenario 1, 2, 3 |
| **NFR-02 (Атомарність ACID)** | X | | X | Scenario 1, 3 |
| **NFR-03 (PCI DSS Безпека)** | | X | | — |

*Примітка: REQ-06 залишається непокритою бізнес-сценаріями (User Stories / Use Cases), оскільки це суто візуальна UI-вимога фронтенду (підсвічування рамки скіна кольором), яка не впливає на серверну логіку чи стан бази даних.*