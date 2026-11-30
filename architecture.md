# Architecture — UML Component Diagram (Lab 5)

Обраний підхід декомпозиції: **Варіант A (горизонтальні шари)** (`Presentation → API → Services → Data access → Store DB`).  
Додатково створено інтерактивну HTML-діаграму зі справжніми UML 2.5 портами та `lollipop`/`socket`: [`architecture.html`](architecture.html).

---

## 1. Діаграма компонентів (`Online Store System`)

```mermaid
flowchart TB
    Customer((Користувач<br/>Customer))

    subgraph System ["Online Store System (Межа системи)"]
        direction TB

        subgraph Presentation ["Presentation (браузер: Chrome 90+ / Safari 14+, UK-ua)"]
            direction LR
            CatalogPage["«component»<br/>Catalog Page"]
            CheckoutPage["«component»<br/>Checkout Page"]
        end

        I_REST(("«provided»<br/>REST /api"))

        subgraph Backend ["«component» Store Backend (один серверний процес)"]
            direction TB

            pHttp["[port in] HTTP in"]
            pWh["[port in] webhook in"]

            subgraph API ["package: API (controllers)"]
                direction LR
                Router["«component»<br/>Router"]
                AuthMW["«component»<br/>AuthMiddleware"]
                CatalogC["«component»<br/>CatalogController"]
                OrderC["«component»<br/>OrderController"]
                WebhookC["«component»<br/>PaymentWebhookController"]
            end

            subgraph Services ["package: Services (business logic)"]
                direction LR
                CustomerS["«component»<br/>CustomerService"]
                CatalogS["«component»<br/>CatalogService"]
                OrderS["«component»<br/>OrderService"]
            end

            subgraph DAL ["package: Data access (repositories)"]
                direction LR
                CustomerR["«component»<br/>CustomerRepository"]
                CategoryR["«component»<br/>CategoryRepository"]
                ProductR["«component»<br/>ProductRepository"]
                OrderR["«component»<br/>OrderRepository"]
            end

            pPsp["[port out] PSP out"]
            pSql["[port out] SQL out"]
        end

        I_SQL(("«provided»<br/>SQL"))
        StoreDB[("«database»<br/>Store DB<br/>(Customer, Category, Product, Order)")]
    end

    I_Wh(("«provided»<br/>Payment Webhook"))
    I_PSP(("«provided»<br/>Payments API<br/>TLS 1.3, ≤ 3.0s"))

    subgraph ExtPG ["«external» PaymentGateway"]
        direction TB
        ExtAPI["«external»<br/>Payments API"]
        ExtPage["«external»<br/>Payment Page (Card / Apple Pay)"]
    end

    %% Користувач
    Customer --> CatalogPage
    Customer --> CheckoutPage
    Customer -. "redirect → оплата" .-> ExtPage

    %% UI -> REST /api -> HTTP in -> Router
    CatalogPage -- "«required»" --> I_REST
    CheckoutPage -- "«required»" --> I_REST
    pHttp --- I_REST
    pHttp -- "delegation" --> Router

    %% Router -> Controllers (пунктирні залежності всередині процесу)
    Router -.-> AuthMW
    Router -.-> CatalogC
    Router -.-> OrderC

    %% Webhook in -> PaymentWebhookController
    ExtAPI -- "«required»" --> I_Wh
    pWh --- I_Wh
    pWh -- "delegation" --> WebhookC

    %% API -> Services
    AuthMW -.-> CustomerS
    CatalogC -.-> CatalogS
    OrderC -.-> OrderS
    WebhookC -.-> OrderS

    %% Services -> Data access (суворо вниз, відповідно до Лаб 1 та Лаб 3)
    CustomerS -.-> CustomerR
    CatalogS -.-> CategoryR
    CatalogS -.-> ProductR
    OrderS -.-> CustomerR
    OrderS -.-> ProductR
    OrderS -.-> OrderR

    %% OrderService -> PSP out -> Payments API
    OrderS -- "delegation" --> pPsp
    pPsp -- "«required»" --> I_PSP
    ExtAPI --- I_PSP

    %% Repositories -> SQL out -> SQL <- Store DB
    CustomerR -- "delegation" --> pSql
    CategoryR -- "delegation" --> pSql
    ProductR -- "delegation" --> pSql
    OrderR -- "delegation" --> pSql
    pSql -- "«required»" --> I_SQL
    StoreDB --- I_SQL
```

---

## 2. Аудит трасування до попередніх моделей (Лаб 1–4)

| Компонент / Шар | Дані (Лаб 1: ER) | Вимоги та UC (Лаб 2) | Взаємодія (Лаб 3: Sequence) | Процес (Лаб 4: Activity) |
| :--- | :--- | :--- | :--- | :--- |
| **`Catalog Page`, `Checkout Page`** | Оперують `Category`, `Product`, `Order` | `Customer`, Chrome 90+ / Safari 14+ (`NFR-1`), `UK-ua` (`NFR-5`), `UC_Checkout` | Ініціюють `checkoutOrder()`, `submitPayment()`, `initiatePaymentSession()` | Доріжка `Customer`: ініціація оформлення, вибір способу оплати, перегляд результату |
| **`Router`, `AuthMiddleware`** | Валідує приналежність `Customer` (`UUID id`) | Заборона `Guest` (`Out of Scope` Лаб 2), пікове навантаження `500 RPS` (`NFR-2`) | Точка входу запитів `Customer → OrderService` | Розподіл HTTP-запитів із браузера у контролери |
| **`CatalogController` / `CatalogService`** | `Category`, `Product` (`name`, `price`, `category_id`) | Каталог товарів магазину (`README.md`) | Джерело даних про товари для замовлення | Підготовка переліку `Product` перед оформленням |
| **`OrderController`, `PaymentWebhookController`, `OrderService`** | `Order` (`status`), `Product` (`price`), `Customer` (`customer_id`) | `REQ-1` (`Calculate Total`), `REQ-2` (`Process Payment`), `REQ-3` (`Pending`), `REQ-4` (`Paid`) | Повний цикл `OrderService`: `calculateTotal()`, `authorizeTransaction()`, прийом `PaymentWebhook` | Доріжка `OrderService`: розрахунок, `FORK`/`JOIN`, утримання `Pending` чи перехід у `Paid` |
| **`Customer/Category/Product/OrderRepository` + `Store DB`** | 4 таблиці спільної схеми: `Customer`, `Category`, `Product`, `Order` | Без збереження карток і транзакцій на боці магазину | `getOrderById()`, `findProductsByOrderId()`, `updateOrderStatus()` | Вузли даних `Product` і `Order` (БД не є окремим лейном процесу) |
| **`PaymentGateway` (`<<external>>`)** | Поза межею БД магазину | Зовнішній актор `PaymentGateway` (`ADR-002`), `TLS 1.3` (`NFR-4`), `≤ 3.0 с` (`NFR-3`), `Card` / `Apple Pay` | Учасник `PaymentGateway`: `authorizeTransaction()`, `createPaymentSession()`, `cancelPaymentSession()` | Доріжка `PaymentGateway`: платіжна сесія `3.0 с`, транзакція, повернення результату або таймаут |

---

## 3. Специфікація портів і контрактів (`provided` / `required`)

| Контракт / Порт | Напрям на межі | Постачальник (`provided`) | Споживач (`required`) | Призначення |
| :--- | :--- | :--- | :--- | :--- |
| **`REST /api` (`HTTP in`)** | Вхідний (`Store Backend`) | `Store Backend` (`Router`) | `Catalog Page`, `Checkout Page` | Єдиний HTTP-контракт UI → backend; `HTTP in` делегує виключно в `Router`. |
| **`Payment Webhook` (`webhook in`)** | Вхідний (`Store Backend`) | `Store Backend` (`PaymentWebhookController`) | `PaymentGateway` (`Payments API`) | Асинхронне сповіщення результату транзакції (`Success` / `Declined` / `Timeout`) у `OrderService`. |
| **`Payments API` (`PSP out`)** | Вихідний (`Store Backend`) | `PaymentGateway` | `Store Backend` (`OrderService`) | Авторизація платежу й створення платіжної сесії по `TLS 1.3` із таймаутом `≤ 3.0 с`. |
| **`SQL` (`SQL out`)** | Вихідний (`Store Backend`) | `Store DB` | Репозиторії `Data access` | Запити до спільної схеми (`Customer`, `Category`, `Product`, `Order`). |
