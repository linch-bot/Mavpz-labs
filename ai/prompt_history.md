# Prompts and Generations (Lab 3)

## 1: Сценарій 1 — Успішна обробка платежу

### Промпт 1 (на основі файлу spec.md)

> На основі файлу `spec.md` побудуй детальну Sequence-діаграму у форматі Mermaid (`sequenceDiagram`) для Сценарію 1 (Успішна обробка платежу). Покажи взаємодію між учасниками для розрахунку суми замовлення та проведення оплати через платіжний шлюз. Обов'язково використай комбінований фрагмент (`loop` або `opt`) та блоки активації (`activate`/`deactivate`).

### Відповідь ШІ (Версія 1)

```mermaid
sequenceDiagram
    actor Customer
    participant ShoppingCart
    participant OrderService
    participant PaymentTransactionDB
    participant PaymentGateway

    Customer->>OrderService: checkout()
    activate OrderService
    OrderService->>ShoppingCart: getCartItems()
    activate ShoppingCart
    ShoppingCart-->>OrderService: itemsList
    deactivate ShoppingCart

    loop Для кожного товару в кошику
        OrderService->>OrderService: sumPrice(item)
    end

    OrderService->>PaymentGateway: pay(totalAmount)
    activate PaymentGateway
    PaymentGateway-->>OrderService: Success
    deactivate PaymentGateway

    opt Статус транзакції Success
        OrderService->>PaymentTransactionDB: saveTransaction(orderId, "Success")
        activate PaymentTransactionDB
        PaymentTransactionDB-->>OrderService: saved
        deactivate PaymentTransactionDB
    end

    OrderService-->>Customer: "Оплата успішно завершена"
    deactivate OrderService
```

## 2: Сценарій 2 (Відмова банку) та Сценарій 3 (Переривання оплати / Таймаут)

### Промпт 2 (на основі файлу spec.md)

> Згенеруй окремі Sequence-діаграми у форматі Mermaid (`sequenceDiagram`) для Сценарію 2 (Неуспішна обробка платежу — статус `Declined`) та Сценарію 3 (Переривання оплати користувачем / Таймаут). У кожній діаграмі використай фрагмент `alt` або `opt` та лінії активації.

### Відповідь ШІ (Версія 1)

**Сценарій 2: Відмова банку**

```mermaid
sequenceDiagram
    actor Customer
    participant OrderService
    participant Order
    participant PaymentGateway

    Customer->>OrderService: submitPayment(orderId)
    activate OrderService
    OrderService->>PaymentGateway: processPayment(orderId, amount)
    activate PaymentGateway
    PaymentGateway-->>OrderService: Declined
    deactivate PaymentGateway

    alt Статус Success
        OrderService->>Order: updateStatus("Paid")
        OrderService-->>Customer: "Оплата успішно завершена"
        deactivate OrderService
    else Статус Declined
        OrderService->>Order: keepStatus("Pending")
        OrderService-->>Customer: "Відмова банку. Спробуйте іншу картку"
    end
```

**Сценарій 3: Переривання оплати користувачем (Таймаут)**

```mermaid
sequenceDiagram
    actor Customer
    participant OrderService
    participant Order
    participant PaymentGateway

    Customer->>OrderService: просить оплатити замовлення
    activate OrderService
    OrderService->>PaymentGateway: відкриває сторінку оплати
    activate PaymentGateway

    opt Користувач передумав
        Customer->>PaymentGateway: закриває вікно до завершення
        PaymentGateway-->>OrderService: немає відповіді (таймаут)
        deactivate PaymentGateway
        OrderService->>Order: змінити статус на "Cancelled"
    end

    OrderService-->>Customer: замовлення скасовано
    deactivate OrderService
```
