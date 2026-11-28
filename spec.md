# Specification (Lab 5: Architecture & Component Diagram)

## 1. Намір та архітектурний підхід (Intent & Architecture Style)

Побудова UML Component Diagram для інтернет-магазину **`Online Store System`** у підході **Варіант A (горизонтальні шари)**: `Presentation → API → Services → Data access → Store DB`, що об'єднує всі моделі попередніх етапів (ER-модель Лаб 1, вимоги та прецеденти Лаб 2, Sequence Лаб 3, Activity Лаб 4).

## 2. Модулі та шари системи (Internal Modules)

- **`Presentation` (браузер: Chrome 90+ / Safari 14+, `UK-ua`):** `Catalog Page` (перегляд товарів за категоріями) та `Checkout Page` (оформлення замовлення й ініціація оплати).
- **`Store Backend` (серверний застосунок, до 500 RPS):**
  - **API (controllers):** `Router`, `AuthMiddleware` (перевірка авторизованого `Customer`, без `Guest`), `CatalogController`, `OrderController`, `PaymentWebhookController`.
  - **Services (business logic):** `CustomerService`, `CatalogService`, `OrderService` (реалізує `Calculate Total` та `Process Payment`).
  - **Data access (repositories):** `CustomerRepository`, `CategoryRepository`, `ProductRepository`, `OrderRepository` (відповідають 4 сутностям Лаб 1).
- **`Store DB`:** окремий процес БД (одна спільна схема для `Customer`, `Category`, `Product`, `Order`).

## 3. Зовнішні інтеграції (External Systems)

- **`PaymentGateway` (`<<external>>`):** єдина зовнішня система поза межею `Online Store System`; надає `Payments API` (шифрування `TLS 1.3`, таймаут `≤ 3.0 с`) та містить `Payment Page` (Card / Apple Pay), на яку перенаправляється клієнт.

## 4. Поза обсягом (Out of Scope)

- Позамодельні зовнішні сервіси (`Email / SMS Provider`, `Identity Provider`, `Delivery API`), яких немає в Лаб 2–4.
- Позамодельні репозиторії чи таблиці БД (`CartRepository`, `PaymentTransactionRepository`, `InvoiceRepository`), що суперечать ER-моделі Лаб 1.

## 5. Acceptance Criteria (Task for AI)

1. **Межа системи та зовнішні інтеграції:** Побудувати UML Component Diagram у Mermaid для `Online Store System` (Варіант A). `<<external>> PaymentGateway` (разом із `Payment Page`) ПОВИНЕН бути розташований суворо поза межею системи; заборонено додавати компоненти з розділу «Поза обсягом».
2. **Порти бекенду та єдиний Router:** На межі `Store Backend` показати порти `HTTP in`, `webhook in`, `PSP out`, `SQL out` із delegation (`-->`) усередину. Єдиний постачальник `REST /api` — `Router` (через `HTTP in`), який маршрутизує до `AuthMiddleware`, `CatalogController`, `OrderController`.
3. **Розмежування інтерфейсів і залежностей:** `Provided/Required` (`lollipop`/`socket`) використовувати ЛИШЕ на межах процесів/систем (`REST /api`, `Payment Webhook`, `Payments API`, `SQL`); усередині `Store Backend` дозволено ВИКЛЮЧНО прямі пунктирні залежності (`..>`).
4. **Наскрізна узгодженість:** Кожен компонент і зв'язок має трасуватися до ER-моделі (Лаб 1), EARS/NFR/UC (Лаб 2), Sequence-діаграм (Лаб 3) та Activity-діаграми (Лаб 4).
5. **Охайність розкладки:** Розташувати шари зверху донизу з мінімальними перетинами зв'язків.
