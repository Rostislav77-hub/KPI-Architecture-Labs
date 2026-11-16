# DEFENSE
Завдання №5: Архітектура (Component Diagram)

**1. Що я просив у AI** (1–2 речення).
На основі власноруч написаної специфікації `lab5/spec.md` поставив ШІ задачу проаналізувати архітектуру системи маркетплейсу за Варіантом A (горизонтальні шари), перевірити трасованість до Лаб 1–4 та правила розмежування портів, контрактів `lollipop/socket` і прямих залежностей перед генерацією діаграми. Промпти та відповіді: `lab5/ai/01_initial_generation.md`, аудит і рев'ю: `lab5/ai/02_audit_and_refinement.md`.

**2. Топ-3 розбіжності, які я знайшов і виправив** (spec ↔ вихід AI):

| # | Що було не так (проти якого пункту spec) | Мій слід (промпт / фідбек / дод. spec) | Виправлення (коміт) |
|---|---|---|---|
| 1 | **Антипатерн «Lollipop на кожному класі» та відсутність портів на межі Backend (проти `AC-04`):** ШІ намалював `lollipop`-інтерфейси між внутрішніми сервісами й репозиторіями, але пропустив порти (`HTTP in`, `webhook in`, `WS out`, `OIDC out`, `PSP out`, `email/push out`, `SQL out`) на межі процесу `Marketplace Backend`. | Аудит у `lab5/ai/02_audit_and_refinement.md` + додано критерій `AC-04` у `lab5/spec.md` | `4786aa0` |
| 2 | **Зовнішні системи й БД всередині Backend та інверсія Webhook (проти `AC-05`):** провайдери `<<external>>` і `Marketplace DB` були поміщені всередину бекенду, а `Webhook` викликався з UI замість `PaymentGateway` через порт `webhook in`. | Аудит у `lab5/ai/02_audit_and_refinement.md` + додано критерій `AC-05` у `lab5/spec.md` | `4786aa0` |
| 3 | **Відсутність `Router` перед контролерами та `PaymentTimeoutJob (cron)` (проти `AC-06`):** `WebUI` звертався до контролерів напряму в обхід єдиного `Router` (`REST /api`), і був відсутній `PaymentTimeoutJob` для контролю 15-хвилинного тайм-ауту оплати з Лаби 4. | Аудит у `lab5/ai/02_audit_and_refinement.md` + додано критерій `AC-06` у `lab5/spec.md` | `4786aa0` |

**3. Ключове рішення:**
Зафіксовано в `lab5/adr/0001-layered-component-architecture.md`. Обрано **Варіант A — горизонтальні шари** (`Presentation → API (controllers) → Services → Data access (repositories) → Marketplace DB`) консистентно для всієї системи. `Marketplace Backend` оформлено як один процес із 7 портами на межі: зовнішні з'єднання проходять через порти у зв'язці з `lollipop/socket` (`REST /api`, `WS /events`, `Webhook`, `Payments API`, `OIDC`, `Email / Push API`, `SQL`), тоді як усередині бекенду між `Router`, контролерами, сервісами та репозиторіями використовуються прямі пунктирні залежності (`..>`).

**4. Перевірка — узгодженість із усім ланцюгом моделей:**
- **Лаба 1 (Дані — `lab1/model.mmd`):** окремий процес `Marketplace DB` (одна спільна схема за Варіантом A) та три репозиторії (`PlayerRepository`, `InventoryRepository`, `TransactionRepository`) через порт `SQL out` обслуговують сутності `Player`, `InventoryItem`, `MarketTransaction`.
- **Лаба 2 (Вимоги та UC — `lab2/spec.md`):** `InventoryService` реалізує перевірку активності лота (`REQ-03`), `WalletService` через порт `PSP out` і `PaymentGateway` забезпечує доплату на суму дефіциту (`REQ-05`), а `TransactionService` виконує компенсацію `refundFunds` (`NFR-02`).
- **Лаба 3 (Взаємодія — `lab3/sequence.mmd`):** усі учасники Sequence-діаграми відображені у компоненти; виклик `createPaymentSession` іде через `PSP out` до `Payments API`, а зворотна подія `confirmPayment` надходить від `PaymentGateway` у контракт `Webhook` порту `webhook in` → `PaymentWebhookController`.
- **Лаба 4 (Процес — `lab4/process.mmd`):** доріжка `Система маркетплейсу` розкладена на `Marketplace Web UI`, `Marketplace Backend` (3 шари) і `Marketplace DB`; паралельний блок `fork/join` (`allSettled`) координується `TransactionService`, а гілка 15-хвилинного тайм-ауту оплати реалізована через `PaymentTimeoutJob (cron)`.

**Здача:**
- **(а) Посилання на GitHub PR:** [https://github.com/Rostislav77-hub/KPI-Architecture-Labs/pull/5](https://github.com/Rostislav77-hub/KPI-Architecture-Labs/pull/5)
- **(б) Цей заповнений DEFENSE.**