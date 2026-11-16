# Специфікація компонентної архітектури системи (Завдання 5)

## 1. Мета, архітектурний варіант і межа системи
Метою є побудова компонентної діаграми (**UML Component Diagram**) системи **«Маркетплейс внутрішньоігрових предметів» (`In-Game Marketplace System`)**, яка узгоджує ER-модель (Лаба 1), вимоги `UC-01` (Лаба 2), Sequence-діаграму (Лаба 3) та Activity-діаграму (Лаба 4).
Обрано **Варіант A — горизонтальні шари** консистентно для всього бекенду з напрямом абстракції згори донизу (`Presentation → API (controllers) → Services → Data access (repositories) → Database`).

- **Усередині межі системи (`In-Game Marketplace System`) — три окремі процеси:**
  1. `Marketplace Web UI` (браузерний клієнт: `Catalog & Lot Page`, `Wallet & Top-Up Page`).
  2. `Marketplace Backend` (єдиний серверний процес із портами на межі: `HTTP in`, `webhook in`, `WS out`, `OIDC out`, `PSP out`, `email/push out`, `SQL out`).
  3. `Marketplace DB` (окремий процес реляційної БД зі спільною схемою таблиць `Player`, `InventoryItem`, `MarketTransaction`).
- **Поза межею системи (`<<external>>`):** `PaymentGateway` (платіжний процесор: `Payments API` та зовнішня `Payment Page`), `GameIdentityProvider` (`OIDC / OAuth 2.0`), `NotificationProvider` (`Email / Push API`).

## 2. Внутрішні шари (Packages), порти та контракти
1. **Presentation (браузер):** `Catalog & Lot Page` та `Wallet & Top-Up Page` споживають публічний контракт `REST /api` (через порт `HTTP in`) і підписуються на `WS /events` (через порт `WS out`). При дефіциті коштів (`REQ-05`) гравець перенаправляється (`redirect`) на зовнішню сторінку `PaymentGateway`.
2. **API (controllers):**
   - Порт `HTTP in` надає `lollipop`-інтерфейс `REST /api` і делегує (`-->`) вхідний трафік виключно в `Router` (визначений фреймворком), який маршрутизує запити до `AuthMiddleware`, `AuthController`, `PurchaseController` (`clickBuy`, `promptTopUp`) та `InventoryController`.
   - Порт `webhook in` надає `lollipop`-інтерфейс `Webhook` (його викликає `PaymentGateway` для події `confirmPayment`) і делегує виклик напряму в `PaymentWebhookController` в обхід `Router`.
   - `PaymentTimeoutJob (cron)` — фоновий компонент у шарі API, що відстежує 15-хвилинний тайм-аут очікування оплати (Лаба 4).
3. **Services (business logic):**
   - `TransactionService` — оркеструє `UC-01`, паралельні гілки `fork/join` (`allSettled`) і компенсацію `refundFunds` (`NFR-02`).
   - `InventoryService` — перевіряє статус лота (`REQ-03`) і змінює власника предмета (`updateItemOwnerAndDelist`).
   - `WalletService` — перевіряє баланс, списує/повертає кошти та ініціює платіжну сесію через порт `PSP out` (`Payments API`, `REQ-05`).
   - `AuthService` — перевіряє облікові дані через порт `OIDC out` (`OIDC`).
   - `NotificationService` — надсилає події у `WS out` та сповіщення (`purchaseCompleted` / `purchaseAborted`) через порт `email/push out`.
4. **Data access (repositories) та БД:**
   - `PlayerRepository`, `InventoryRepository`, `TransactionRepository` делегують запити в єдиний вихідний порт `SQL out`, який споживає інтерфейс `SQL` бази `Marketplace DB`.

## 3. Поза обсягом (Out of Scope)
- Адміністративна панель модерації, CDN для статики та окремий брокер повідомлень (Event Bus).

## 4. Критерії прийняття (Acceptance Criteria)
- **AC-01 (Повнота шарів і компонентів):** Відображено `Presentation`, 3 горизонтальні шари всередині `Marketplace Backend` (`API`, `Services`, `Data access`), окремий процес `Marketplace DB` та 3 зовнішні системи `<<external>>`.
- **AC-02 (Правило портів, lollipop та прямих залежностей):** З'єднання через межу процесу `Marketplace Backend` проходять через порти (`in`/`out`), які стикуються з `lollipop/socket`-контрактами; усередині бекенду між контролерами, сервісами й репозиторіями використовуються прямі пунктирні залежності (`..>`) без зайвих `lollipop`.
- **AC-03 (Трасованість до Лаб 1–4):** Усі сутності ER (Лаба 1), вимоги `REQ-03`, `REQ-05`, `NFR-02` (Лаба 2), повідомлення Sequence (Лаба 3) та логіка Activity (Лаба 4, включно з тайм-аутом 15 хв і паралельним оновленням) відображені в компонентах.
- **AC-04 (Відсутність «Lollipop на кожному класі» та обов'язкові порти на межі Backend):** Заборонено створювати `lollipop`-інтерфейси між внутрішніми класами (`Controller → Service → Repository`); усі 7 зовнішніх каналів `Marketplace Backend` (`HTTP in`, `webhook in`, `WS out`, `OIDC out`, `PSP out`, `email/push out`, `SQL out`) оформлюються як порти з delegation (`-->`) усередину та `lollipop/socket` назовні.
- **AC-05 (Чітка межа системи, окремий процес БД та інверсія Webhook):** Системи зі стереотипом `<<external>>` (`PaymentGateway`, `GameIdentityProvider`, `NotificationProvider`) розміщуються строго поза рамкою `In-Game Marketplace System`; `Marketplace DB` розміщується поза `Marketplace Backend` як окремий процес; контракт `Webhook` надається (`provided`) портом `webhook in` бекенду і споживається (`required`) зовнішнім `PaymentGateway`.
- **AC-06 (Маршрутизація через Router та трасування тайм-ауту оплати):** Порт `HTTP in` делегує виклики виключно в `Router`, який розподіляє їх по контролерах (окрім `PaymentWebhookController`, що працює від `webhook in`, та `PaymentTimeoutJob (cron)`, що запускається за розкладом для контролю 15-хвилинного тайм-ауту оплати).