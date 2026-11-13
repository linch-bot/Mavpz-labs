# Prompts and Generations (Lab 4)

## 1: Перша генерація Activity-діаграми бізнес-процесу оплати

### Промпт 1 (на основі файлу spec.md)

> На основі файлу `spec.md` побудуй єдину Activity-діаграму у форматі Mermaid (`flowchart TD` зі swimlanes) для бізнес-процесу оформлення та оплати замовлення (`Checkout Order` + `Calculate Total` + `Process Payment`). Покажи початок і кінець, розгалуження, цикл повторної спроби оплати, паралельну гілку (`fork`/`join`) та обмін повідомленнями з `PaymentGateway`.

### Відповідь ШІ (Версія 1)

```mermaid
flowchart TD
    subgraph Customer ["Актор: Customer"]
        Start((●)) --> A_Init["Замовлення сформовано"]
        A_Select["Обрати спосіб оплати"]
        A_Retry{"Повторити оплату?"}
    end

    subgraph OrderService ["Система: OrderService"]
        A_Calc["Розрахувати суму"]
        S_Pay[">Надіслати запит на оплату"]
        D_Res{"Статус оплати?"}
        Fork["=== FORK ==="]
        A_Notify[">Надіслати підтвердження успіху"]
        Join["=== JOIN ==="]
        End1(((●)))
        End2(((●)))
    end

    subgraph OrderDB ["База даних: OrderDB"]
        A_Pending["Pending"]
        A_Paid["Paid"]
    end

    subgraph PaymentGateway ["Зовнішня система: PaymentGateway"]
        A_Proc["Обробити платіж"]
    end

    A_Init --> A_Calc
    A_Calc --> A_Pending
    A_Pending --> A_Select
    A_Select --> S_Pay
    S_Pay -. "authorizeTransaction" .-> A_Proc
    A_Proc -. "Success / Declined" .-> D_Res

    D_Res -- "[Success]" --> Fork
    Fork --> A_Paid
    A_Paid --> Join
    Fork --> A_Notify
    A_Notify --> End1

    Join --> End2

    D_Res -- "[Declined]" --> A_Retry
    A_Retry -- "[так]" --> A_Select
    A_Retry -- "[ні]" --> End2
```
