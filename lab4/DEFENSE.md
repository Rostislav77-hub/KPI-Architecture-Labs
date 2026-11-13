# DEFENSE
Завдання №4: Процес / поведінка (Activity, BPMN або машина станів)

**1. Що я просив у AI** (1–2 речення).
На основі власноруч написаної специфікації `lab4/spec.md` поставив ШІ задачу проаналізувати процес `UC-01 «Купівля предмета»`, знайти прогалини в переходах, синхронізації паралельних гілок і обробці повідомлень та поставити уточнювальні питання перед генерацією Activity-діаграми. Промпти та відповіді: `lab4/ai/01_initial_generation.md`, аудит і рев'ю: `lab4/ai/02_audit_and_refinement.md`.

**2. Топ-3 розбіжності, які я знайшов і виправив** (spec ↔ вихід AI):

| # | Що було не так (проти якого пункту spec) | Мій слід (промпт / фідбек / дод. spec) | Виправлення (коміт) |
|---|---|---|---|
| 1 | **Змішування нотацій, зайва доріжка UI та рівень БД (проти `AC-04`):** у первинній генерації ШІ використав іменники-стани («Статус: Продано», «Статус: Оплачено») замість дієслів, виділив проміжний `MarketplaceUI` в окрему доріжку без власної бізнес-логіки та згадав фізичну БД. | Аудит у `lab4/ai/02_audit_and_refinement.md` + додано критерій `AC-04` у `lab4/spec.md` | [`9032a7c`](https://github.com/Rostislav77-hub/KPI-Architecture-Labs/commit/9032a7c) |
| 2 | **Повідомлення без реакції на невідповідь і розбіжність назв (проти `AC-03`):** після надсилання запиту в `PaymentGateway` не було показано очікування події та гілки тайм-ауту (15 хв) за відсутності відповіді; назви сигналів не збігалися з `lab3/sequence.mmd`. | Аудит у `lab4/ai/02_audit_and_refinement.md` + уточнено критерій `AC-03` у `lab4/spec.md` | [`9032a7c`](https://github.com/Rostislav77-hub/KPI-Architecture-Labs/commit/9032a7c) |
| 3 | **Достроковий вихід усередині `fork/join` та брак вузлів даних ER (проти `AC-05`, `AC-06`):** гілка помилки виходила на `EndNode` в обхід `JoinNode`, ламаючи синхронізацію `Promise.allSettled`; були відсутні об'єкти даних із ER-моделі Лаби 1. | Аудит у `lab4/ai/02_audit_and_refinement.md` + додано критерії `AC-05`, `AC-06` у `lab4/spec.md` | [`9032a7c`](https://github.com/Rostislav77-hub/KPI-Architecture-Labs/commit/9032a7c) |

**3. Ключове рішення:**
Зафіксовано в `lab4/adr/0001-activity-diagram-and-payment-timeout.md`. Обрано **Activity-діаграму (`Mermaid flowchart`) із 3 доріжками (`Player`, `Marketplace System`, `PaymentGateway`)**, оскільки процес вимагає розмежування відповідальності між актором, системою та зовнішнім шлюзом (у машині станів `stateDiagram-v2` ролі та паралельні дії зникають, а BPMN 2.0 XML надлишковий через відсутність фізичних `Manual Task` поза системою).
**Як вирішено спірні місця структури кроків:**
- **Що злито:** внутрішні сервіси маркетплейсу (`TransactionService`, `InventoryService`, `WalletService`) об'єднано в одну доріжку `Система маркетплейсу` на рівні бізнес-процесу, а клієнтський UI опущено (він лише передає дані); потік після успішного поповнення (`creditWalletBalance`) циклом зливається з основним кроком списання коштів (`deductFunds`).
- **Що паралельне (`fork / join`):** після списання коштів паралельно виконуються передача предмета зі зняттям з лотів (`updateItemOwnerAndDelist`) та створення запису `MarketTransaction` (`createMarketTransaction`). Обидві гілки сходяться у `join` за семантикою `allSettled` без дострокових виходів, а перевірка успішності та відкат `refundFunds` (`NFR-02`) виконуються після синхронізації.
- **Що опціональне та як оброблено невідповідь:** поповнення балансу (`<<extend>>`, `REQ-05`) виконується опціонально за рішенням гравця; при зверненні до `PaymentGateway` (`createPaymentSession`) змодельовано як прийом підтвердження (`confirmPayment`), так і подію тайм-ауту 15 хв зі скасуванням операції.

**4. Перевірка:**
- **Узгодженість із ER-моделлю (`lab1/model.mmd`):** усі операції працюють із вузлами даних `InventoryItem` (перевірка статусу, зміна власника), `Player` (`wallet_balance`) та `MarketTransaction` (фіксація угоди).
- **Узгодженість із UC та Gherkin (`lab2/spec.md`):** кожен із 3 сценаріїв `UC-01` має повний досяжний шлях від `Початок` до `Кінець` з вичерпними умовами в `[...]` (Сценарій 1 — `[ACTIVE]` → достатньо коштів → `fork/join` → успіх або `refundFunds`; Сценарій 2 — дефіцит коштів → поповнення через шлюз або тайм-аут/відмова; Сценарій 3 — `[SOLD]` → негайна відмова `rejectLotSold` за `REQ-03`).
- **Узгодженість із Sequence-діаграмою (`lab3/sequence.mmd`):** усі повідомлення процесу (`clickBuy`, `checkItemStatus`, `rejectLotSold`, `checkBalance`, `rejectInsufficientFunds` / `promptTopUp`, `createPaymentSession`, `submitCardPayment`, `confirmPayment`, `creditWalletBalance`, `deductFunds`, `updateItemOwnerAndDelist`, `createMarketTransaction`, `refundFunds`, `purchaseAborted`, `purchaseCompleted`) повністю відповідають викликам Sequence-діаграми.

**Здача:**
- **(а) Посилання на GitHub PR:** [https://github.com/Rostislav77-hub/KPI-Architecture-Labs/pull/4](https://github.com/Rostislav77-hub/KPI-Architecture-Labs/pull/4)
- **(б) Цей заповнений DEFENSE.**