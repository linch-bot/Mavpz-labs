# Process Model — Activity Diagram (Lab 4)

Обраний прецедент із Лаб 2: **`Checkout Order`** (`<<include>>` **`Calculate Total`**, `<<extend>>` **`Process Payment`**). Усі 3 Gherkin-сценарії (`Success`, `Declined`, `Timeout`) та повідомлення із Sequence-діаграм Лаб 3 об'єднано в єдину Activity-діаграму.

---

## 1. Activity-діаграма процесу оформлення та оплати замовлення

```mermaid
flowchart TD
    subgraph Customer ["Актор: Customer (Авторизований покупець)"]
        Start((●)) --> A_Init["Ініціювати оформлення замовлення (Checkout Order)"]
        A_SelectMethod["Обрати спосіб оплати (Card / Apple Pay) і підтвердити"]
        E_RecvTimeout["Отримати сповіщення про переривання оплати"]
        E_RecvDeclined["Отримати сповіщення про відмову банку"]
        D_Retry{"Спробувати іншу картку?"}
        E_RecvSuccess["Отримати підтвердження: Оплата успішно завершена"]
    end

    subgraph OrderService ["Система: OrderService (Інтернет-магазин)"]
        A_Calc["Розрахувати загальну суму замовлення (Calculate Total)"]
        M_Retry{" "}
        S_AuthReq[">Надіслати запит на оплату (authorizeTransaction)"]
        A_KeepPendingTimeout["Утримати статус замовлення Pending"]
        S_MsgTimeout[">Надіслати: Оплату перервано"]
        D_Status{"Який статус відповіді шлюзу?"}
        A_KeepPendingDeclined["Утримати статус замовлення Pending"]
        S_MsgDeclined[">Надіслати: Відмова банку. Спробуйте іншу картку"]
        Fork["=== FORK ==="]
        A_SetPaid["Змінити статус замовлення на Paid"]
        S_MsgSuccess[">Надіслати: Оплата успішно завершена"]
        Join["=== JOIN (allSettled) ==="]
        M_End{" "}
        End(((●)))

        Data_Product[("Дані: Product — id, name, price")]
        Data_Order[("Дані: Order — id, customer_id, status")]
    end

    subgraph PaymentGateway ["Зовнішня система: PaymentGateway (Платіжний шлюз)"]
        A_Session["Відкрити платіжну сесію (таймер 3.0 с)"]
        D_Confirm{"Дія клієнта протягом 3.0 с (NFR-3)?"}
        S_TimeoutEvt[">Повідомити про відсутність підтвердження (Timeout)"]
        A_Process["Провести транзакцію (TLS 1.3)"]
        S_RespEvt[">Надіслати результат транзакції (Success / Declined)"]
    end

    A_Init --> A_Calc
    Data_Product -. "читання price (REQ-1)" .-> A_Calc
    A_Calc -. "status = Pending (REQ-3)" .-> Data_Order
    A_Calc --> M_Retry
    M_Retry --> A_SelectMethod
    A_SelectMethod --> S_AuthReq
    S_AuthReq -. "повідомлення (REQ-2)" .-> A_Session
    A_Session --> D_Confirm

    D_Confirm -- "[Клієнт закрив сторінку АБО час > 3.0 с]" --> S_TimeoutEvt
    S_TimeoutEvt -. "немає підтвердження" .-> A_KeepPendingTimeout
    A_KeepPendingTimeout -. "status = Pending" .-> Data_Order
    A_KeepPendingTimeout --> S_MsgTimeout
    S_MsgTimeout -. "повідомлення" .-> E_RecvTimeout
    E_RecvTimeout --> M_End

    D_Confirm -- "[Клієнт підтвердив оплату <= 3.0 с]" --> A_Process
    A_Process --> S_RespEvt
    S_RespEvt -. "відповідь шлюзу" .-> D_Status

    D_Status -- "[gateway_status == Declined]" --> A_KeepPendingDeclined
    A_KeepPendingDeclined -. "status = Pending" .-> Data_Order
    A_KeepPendingDeclined --> S_MsgDeclined
    S_MsgDeclined -. "повідомлення" .-> E_RecvDeclined
    E_RecvDeclined --> D_Retry
    D_Retry -- "[так]" --> M_Retry
    D_Retry -- "[ні]" --> M_End

    D_Status -- "[gateway_status == Success]" --> Fork
    Fork --> A_SetPaid
    A_SetPaid -. "status = Paid (REQ-4)" .-> Data_Order
    Fork --> S_MsgSuccess
    S_MsgSuccess -. "повідомлення" .-> E_RecvSuccess
    A_SetPaid --> Join
    E_RecvSuccess --> Join
    Join --> M_End
    M_End --> End

    style Fork fill:#333,color:#fff,stroke:#333
    style Join fill:#333,color:#fff,stroke:#333
    style Start fill:#000,color:#fff
    style End fill:#000,color:#fff,stroke-width:3px
```

---

## 2. Аудит шляхів (Gherkin Лаб 2 → Шлях на діаграмі)

1. **Сценарій 1 (Успішна обробка платежу, `Success -> Paid`):**
   `Start` → `Ініціювати оформлення` → `Розрахувати загальну суму (Product.price)` → `Обрати спосіб оплати` → `Надіслати запит на оплату` → `Відкрити платіжну сесію` → `[Клієнт підтвердив оплату <= 3.0 с]` → `Провести транзакцію` → `[gateway_status == Success]` → `FORK` (`Змінити статус Order на Paid` ∥ `Надіслати: Оплата успішно завершена` → `Отримати підтвердження`) → `JOIN` → `End`.
2. **Сценарій 2 (Відмова банку, `Declined -> Pending`):**
   `Start` → ... → `[Клієнт підтвердив оплату <= 3.0 с]` → `Провести транзакцію` → `[gateway_status == Declined]` → `Утримати статус замовлення Pending` → `Надіслати: Відмова банку. Спробуйте іншу картку` → `Отримати сповіщення про відмову банку` → `Спробувати іншу картку?` → `[так]` повернення через `M_Retry` на повторну оплату або `[ні]` → `End`.
3. **Сценарій 3 (Переривання оплати / Таймаут `3.0 с`):**
   `Start` → ... → `Відкрити платіжну сесію (таймер 3.0 с)` → `[Клієнт закрив сторінку АБО час > 3.0 с]` → `Повідомити про відсутність підтвердження (Timeout)` → `Утримати статус замовлення Pending` → `Надіслати: Оплату перервано` → `Отримати сповіщення про переривання оплати` → `End`.

---

## 3. Аудит повідомлень (Process ↔ Sequence Лаб 3)

| Повідомлення в процесі | Відправник | Отримувач | Відповідність Sequence (Лаб 3) | Реакція на відповідь або її відсутність |
| :--- | :--- | :--- | :--- | :--- |
| Запит на оплату | `OrderService` | `PaymentGateway` | `authorizeTransaction()` / `createPaymentSession()` | Шлюз відкриває сесію з лімітом `3.0 с` (`NFR-3`); за відсутності підтвердження породжується таймаут (`Order.status = "Pending"`). |
| Результат транзакції | `PaymentGateway` | `OrderService` | `PaymentResponse(Success / Declined)` / `TimeoutException` | `OrderService` розгалужує потік через `D_Status` (`Paid` або повтор/очікування `Pending`). |
| Фінальне сповіщення клієнту | `OrderService` | `Customer` | `CheckoutResult(order_status, message)` | `Customer` отримує підтвердження успіху, пропозицію змінити картку або статус переривання. |
