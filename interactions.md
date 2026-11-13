# Interaction Models — Sequence Diagrams (Lab 3)

Обраний прецедент із Лаб 2: **`Process Payment`** (розширює **`Checkout Order`** через `<<extend>>` та спирається на розрахунок вартості **`Calculate Total`** через `<<include>>`).

---

## 1. Сценарій 1: Успішна обробка платежу (`Success -> Paid`)

**Відповідність Лаб 2 (Gherkin + EARS):**
- Розрахунок загальної суми замовлення (`Order`) на основі ціни (`Product.price`) обраних товарів (`REQ-1`, `UC_Calculate`).
- Утримання `Order.status = "Pending"` під час обробки транзакції (`REQ-3`) та відправка запиту до `PaymentGateway` (`REQ-2`, `UC_Payment`).
- При отриманні `gateway_status = "Success"` зміна статусу замовлення на `"Paid"` (`REQ-4`) і повернення клієнту повідомлення `"Оплата успішно завершена"`.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant OrderService
    participant Order
    participant Product
    participant PaymentGateway

    Customer->>OrderService: checkoutOrder(orderId: UUID, paymentMethod: String)
    activate OrderService

    OrderService->>Order: getOrderById(orderId: UUID)
    activate Order
    Order-->>OrderService: Order(id, customer_id, status="Pending")
    deactivate Order

    OrderService->>Product: findProductsByOrderId(orderId: UUID)
    activate Product
    Product-->>OrderService: List<Product(id, name, price)>
    deactivate Product

    loop Для кожного Product у замовленні (UC_Calculate / REQ-1)
        OrderService->>OrderService: calculateTotal(product.price: Decimal)
    end

    OrderService->>PaymentGateway: authorizeTransaction(orderId: UUID, totalAmount: Decimal, paymentMethod: String)
    activate PaymentGateway
    PaymentGateway-->>OrderService: PaymentResponse(gateway_status="Success")
    deactivate PaymentGateway

    opt gateway_status == "Success" (REQ-4)
        OrderService->>Order: updateOrderStatus(orderId: UUID, status="Paid")
        activate Order
        Order-->>OrderService: Order(id, status="Paid")
        deactivate Order
    end

    OrderService-->>Customer: CheckoutResult(order_status="Paid", message="Оплата успішно завершена")
    deactivate OrderService
```

---

## 2. Сценарій 2: Неуспішна обробка платежу / Відмова банку (`Declined -> Pending`)

**Відповідність Лаб 2 (Gherkin + EARS):**
- Відправка запиту на оплату (`Pay by Card` / `Pay by Apple Pay`) до `PaymentGateway` (`REQ-2`).
- Обробка розгалуження за відповіддю шлюзу у фрагменті `alt`: при `gateway_status = "Declined"` статус `Order.status` залишається `"Pending"` (`REQ-3`), а клієнту повертається повідомлення `"Відмова банку. Спробуйте іншу картку"`.
- Лінія активації `OrderService` починається до блоку `alt` і завершується після `end`.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant OrderService
    participant Order
    participant Product
    participant PaymentGateway

    Customer->>OrderService: submitPayment(orderId: UUID, paymentMethod: String)
    activate OrderService

    OrderService->>Order: getOrderById(orderId: UUID)
    activate Order
    Order-->>OrderService: Order(id, customer_id, status="Pending")
    deactivate Order

    OrderService->>Product: findProductsByOrderId(orderId: UUID)
    activate Product
    Product-->>OrderService: List<Product(id, name, price)>
    deactivate Product

    OrderService->>OrderService: calculateTotal(products: List<Product>)

    OrderService->>PaymentGateway: authorizeTransaction(orderId: UUID, totalAmount: Decimal, paymentMethod: String)
    activate PaymentGateway
    PaymentGateway-->>OrderService: PaymentResponse(gateway_status="Declined")
    deactivate PaymentGateway

    alt gateway_status == "Success"
        OrderService->>Order: updateOrderStatus(orderId: UUID, status="Paid")
        activate Order
        Order-->>OrderService: Order(id, status="Paid")
        deactivate Order
        OrderService-->>Customer: CheckoutResult(order_status="Paid", message="Оплата успішно завершена")
    else gateway_status == "Declined"
        OrderService->>Order: updateOrderStatus(orderId: UUID, status="Pending")
        activate Order
        Order-->>OrderService: Order(id, status="Pending")
        deactivate Order
        OrderService-->>Customer: CheckoutResult(order_status="Pending", message="Відмова банку. Спробуйте іншу картку")
    end

    deactivate OrderService
```

---

## 3. Сценарій 3: Переривання оплати користувачем / Таймаут (`Timeout -> Pending`)

**Відповідність Лаб 2 (Gherkin + EARS + NFR):**
- Поки транзакція обробляється, `Order.status` утримується як `"Pending"` (`REQ-3`).
- Коли `Customer` закриває сторінку платіжного шлюзу до завершення операції, `PaymentGateway` не надсилає підтвердження транзакції (спрацьовує верхня межа очікування `3.0 с` згідно з `NFR-3`).
- Статус `Order.status` залишається `"Pending"`.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant OrderService
    participant Order
    participant PaymentGateway

    Customer->>OrderService: initiatePaymentSession(orderId: UUID, paymentMethod: String)
    activate OrderService

    OrderService->>Order: getOrderById(orderId: UUID)
    activate Order
    Order-->>OrderService: Order(id, customer_id, status="Pending")
    deactivate Order

    OrderService->>PaymentGateway: createPaymentSession(orderId: UUID, timeoutSec=3.0)
    activate PaymentGateway
    PaymentGateway-->>OrderService: SessionToken(sessionUrl)

    Customer->>PaymentGateway: cancelPaymentSession()

    alt Підтвердження надійшло до 3.0 с (NFR-3)
        PaymentGateway-->>OrderService: PaymentWebhook(gateway_status="Success")
        OrderService->>Order: updateOrderStatus(orderId: UUID, status="Paid")
        activate Order
        Order-->>OrderService: Order(id, status="Paid")
        deactivate Order
    else Клієнт закрив сторінку шлюзу / Таймаут > 3.0 с (REQ-3)
        PaymentGateway-->>OrderService: TimeoutException(confirmed=false)
        OrderService->>Order: updateOrderStatus(orderId: UUID, status="Pending")
        activate Order
        Order-->>OrderService: Order(id, status="Pending")
        deactivate Order
    end
    deactivate PaymentGateway

    OrderService-->>Customer: CheckoutResult(order_status="Pending", message="Оплату перервано. Замовлення очікує оплати")
    deactivate OrderService
```
