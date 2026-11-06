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
