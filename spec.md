# Specification (Lab 3: Interaction & Sequence Diagrams)

## 1. Intent and Selected Use Case

Деталізація внутрішньої взаємодії компонентів системи для прецеденту **`Process Payment`** (у контексті оформлення замовлення **`Checkout Order`** та обов'язкового розрахунку суми **`Calculate Total`** із Лабораторної роботи №2) на основі затвердженої ER-моделі даних із Лабораторної роботи №1 (`Customer`, `Order`, `Product`, `Category`).

## 2. Scenarios (from Lab 2 Gherkin)

1. **Сценарій 1 (Успішна обробка платежу):** `Customer` переходить до оплати сформованого `Order` (`Order.status = "Pending"`, `REQ-3`). Система отримує товари замовлення (`Product`), розраховує загальну суму на основі `Product.price` (`REQ-1`), надсилає запит до `PaymentGateway` (`REQ-2`), отримує статус транзакції `Success`, змінює `Order.status` на `"Paid"` (`REQ-4`) і повертає клієнту повідомлення `"Оплата успішно завершена"`.
2. **Сценарій 2 (Неуспішна обробка платежу / Відмова банку):** Після звернення до `PaymentGateway` шлюз повертає статус транзакції `Declined`. Система утримує `Order.status` як `"Pending"` і повертає клієнту повідомлення `"Відмова банку. Спробуйте іншу картку"`.
3. **Сценарій 3 (Переривання оплати користувачем / Таймаут):** `Customer` закриває сторінку платіжного шлюзу до завершення операції (або вичерпується ліміт часу `3.0 с` згідно з `NFR-3`). `PaymentGateway` не надсилає підтвердження транзакції, і система залишає `Order.status` рівним `"Pending"`.

## 3. Participants

- **Customer (Actor):** Авторизований покупець із моделі даних Лаб 1 (`UUID id`, `email`, `full_name`).
- **OrderService (Service):** Внутрішній компонент системи, який оркеструє оформлення замовлення (`Checkout Order`), розрахунок вартості (`Calculate Total`) та взаємодію з платіжним шлюзом (`Process Payment`).
- **Product (Entity / Repository):** Доступ до сутності `Product` із Лаб 1 (`UUID id`, `String name`, `Decimal price`) для отримання переліку товарів у замовленні та їхніх цін.
- **Order (Entity / Repository):** Доступ до сутності `Order` із Лаб 1 (`UUID id`, `UUID customer_id`, `Timestamp order_date`, `String status`) для читання й оновлення стану замовлення.
- **PaymentGateway (External System):** Зовнішній платіжний сервіс із Лаб 2 (`ADR-002`), який обробляє платіж (`Pay by Card` / `Pay by Apple Pay`) за протоколом TLS 1.3 (`NFR-4`).

## 4. Поза обсягом (Out of Scope)

- Введення нових сутностей бази даних, яких не існує в ER-моделі Лаб 1 (зокрема `ShoppingCart`, `PaymentTransactionDB`, `Invoice` або `Warehouse`).
- Збереження окремих записів про платіжні транзакції на боці магазину (відповідно до Лаб 2 фіксується лише поле `Order.status`).
- Додаткові сервіси сповіщень (`EmailService`, `SMSService`) або актор `Manager`/`Guest`.

## 5. Acceptance Criteria (Task for AI)

1. **Покриття сценаріїв:** Створити 3 окремі детальні Sequence-діаграми у синтаксисі Mermaid (`sequenceDiagram`) — по одній на кожен сценарій обраного UC із Лаб 2.
2. **Комбіновані фрагменти:** У кожній діаграмі обов'язково використати щонайменше один фрагмент керування потоком (`alt`, `opt` або `loop`).
3. **Цілісність Activation Bars:** Лінія активації (`activate` / `deactivate`) учасника, що працює поза фрагментом або в кількох гілках умовного блоку (`alt` / `opt` / `loop`), ПОВИННА починатися до входу у фрагмент і закінчуватися строго після ключового слова `end` — ЗАБОРОНЕНО обривати `deactivate` всередині однієї гілки `alt`.
4. **Сувора відповідність ER-моделі Лаб 1:** Використовувати ВИКЛЮЧНО затверджених учасників із Розділу 3 (`Customer`, `OrderService`, `Product`, `Order`, `PaymentGateway`).
5. **Реалістичні назви методів та узгодженість статусів із Gherkin (Лаб 2):** Усі виклики мають бути записані як сигнатури методів (`checkoutOrder(orderId: UUID)`, `findProductsByOrderId(orderId: UUID)`, `calculateTotal(products)`, `authorizeTransaction(orderId: UUID, amount: Decimal)`, `updateOrderStatus(orderId: UUID, status: String)`). ЗАБОРОНЕНО використовувати розмовний текст або статус `"Cancelled"` — при відмові банку чи таймауті (`3.0 с`) статус `Order.status` ПОВИНЕН залишатися `"Pending"`.
