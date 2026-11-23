# UML-диаграммы — «Управление пекарней»

По ТЗ из readme.md репозитория. Диаграммы написаны на Mermaid (rendering в VSCode, GitHub, Mermaid Live Editor и т.п.).

---

## 1. Диаграмма вариантов использования (Use Case)

```mermaid
flowchart TD
    Tech[Технолог (кондитер)] --> UC1[Вести карточки изделий]
    Tech --> UC2[Вести справочник рецептур]
    Tech --> UC3[Оформлять заказы на изготовление]
    Tech --> UC4[Списывать сырьё в производство]

    Klad[Кладовщик] --> UC5[Вести справочник сырья и ингредиентов]
    Klad --> UC6[Учитывать приход сырья]
    Klad --> UC7[Проводить инвентаризацию]
    Klad --> UC8[Списывать просроченную продукцию]

    Kas[Кассир-продавец] --> UC9[Учитывать продажи через кассу]
    Kas --> UC10[Отпуск продукции по заказам]

    Adm[Администратор системы] --> UC11[Управление пользователями и правами]
    Adm --> UC12[Резервное копирование и восстановление БД]
    Adm --> UC13[Настройка цен и тарифов]

    Tech --> UC14[Формировать отчёт о выпуске продукции]
    Klad --> UC15[Формировать отчёт о расходе и остатках сырья]
    Kas --> UC16[Формировать отчёт о продажах и выручке]
    Adm --> UC17[Список позиций, требующих контроля]

    UC14 --> EX1[Экспорт в MS Excel]
    UC15 --> EX1
    UC16 --> EX1

    UC1 --> SEARCH[Поиск: наименование, категория]
    UC2 --> SEARCH
    UC5 --> SEARCH

    subgraph "Аутентификация"
        AUTH[Вход по паролю]
    end
    Tech --> AUTH
    Klad --> AUTH
    Kas --> AUTH
    Adm --> AUTH
```

---

## 2. Диаграмма классов (Class)

```mermaid
classDiagram
    class User {
        +id: int
        +login: string
        +passwordHash: string
        +role: Role
        +fullName: string
    }

    class Role {
        <<enumeration>>
        TECHNOLOG
        WAREHOUSEMAN
        CASHIER
        ADMIN
    }

    class Product {
        +id: int
        +name: string
        +category: string
        +weight: decimal
        +price: decimal
        +expiryDate: date
        +composition: string
    }

    class Recipe {
        +id: int
        +product: Product
        +operation: string
        +outputPerUnit: decimal
    }

    class Ingredient {
        +id: int
        +name: string
        +unit: string
        +price: decimal
        +supplier: string
        +stock: decimal
    }

    class RawMaterialOperation {
        +id: int
        +dateTime: datetime
        +type: MatType
        +quantity: decimal
    }

    class MatType {
        <<enumeration>>
        INCOME
        WRITE_OFF_PRODUCTION
        INVENTORY
        WRITE_OFF_EXPIRED
    }

    class Order {
        +id: int
        +client: string
        +dateTimeDelivery: datetime
        +status: OrderStatus
        +items: string
    }

    class OrderStatus {
        <<enumeration>>
        NEW
        IN_PRODUCTION
        READY
        ISSUED
        CANCELLED
    }

    class Sale {
        +id: int
        +dateTime: datetime
        +quantity: decimal
        +amount: decimal
    }

    class Report {
        +id: int
        +periodFrom: date
        +periodTo: date
        +createdAt: datetime
        +generate(): void
        +exportToExcel(): void
    }

    Product "1" --> "0..*" Recipe : recipe
    Recipe "0..*" --> "1..*" Ingredient : uses (norm)
    Product "1" --> "0..*" Sale : sold
    Order "0..*" --> "1..*" Product : contains
    User "1" --> "0..*" Report : generates

    class WarehouseService {
        +receive(ingredient: Ingredient, qty: decimal): void
        +writeOffProduction(ingredient: Ingredient, qty: decimal): void
        +inventory(ingredient: Ingredient, actualQty: decimal): void
        +writeOffExpired(product: Product): void
        -validateQuantity(qty: decimal): bool
        -checkStock(ingredient: Ingredient, qty: decimal): bool
    }

    class OrderService {
        +createOrder(order: Order): void
        +updateStatus(order: Order, status: OrderStatus): void
        +issue(order: Order): void
    }

    class SalesService {
        +sell(product: Product, qty: decimal): void
        +releaseByOrder(order: Order): void
    }

    class ReportService {
        +productionReport(from: date, to: date): Report
        +rawMaterialReport(withRemains: bool): Report
        +salesReport(from: date, to: date): Report
        +orderRegister(from: date, to: date): Report
        +controlPositionsReport(): Report
    }

    User --> WarehouseService : uses
    User --> OrderService : uses
    User --> SalesService : uses
    User --> ReportService : uses
```

---

## 3. Диаграмма последовательности — списание сырья в производство (Sequence)

```mermaid
sequenceDiagram
    participant Tech as Технолог
    participant UI as UI-слой (Склад)
    participant WS as WarehouseService
    participant DB as База данных
    participant Rep as ReportService

    Tech->>UI: списание сырья в производство (ингредиент, количество)
    UI->>WS: writeOffProduction(ingredient, qty)
    WS->>WS: validateQuantity(qty)
    alt количество отрицательное или 0
        WS-->>UI: ошибка «некорректное количество»
    end
    WS->>DB: SELECT остаток ингредиента
    DB-->>WS: остаток
    alt остаток < количества
        WS-->>UI: ошибка «списание сверх остатка» (недостача)
    end
    WS->>DB: UPDATE остаток (списание) — BEGIN TRANSACTION
    WS->>DB: INSERT операция склада — COMMIT
    DB-->>WS: ok
    WS-->>UI: успех
    UI->>Rep: отчёт о расходе сырья / остатках
    Rep-->>Tech: результат
```

---

## 4. Диаграмма деятельности — оформление заказа на изготовление (Activity)

```mermaid
flowchart TD
    A([Начало]) --> B[Аутентификация пользователя]
    B --> C{Пароль верен?}
    C -- нет --> B
    C -- да --> D[Выбор изделия по рецептуре]
    D --> E[Ввод клиента, состава, даты выдачи]
    E --> F{Проверка входных данных}
    F -- ошибка --> E
    F -- ок --> G{Сырья достаточно?}
    G -- нет --> H[Ошибка: недостача сырья] --> E
    G -- да --> I[Создать заказ со статусом NEW]
    I --> J[Списать сырьё по нормам закладки]
    J --> K[Перевести заказ в IN_PRODUCTION / READY]
    K --> L{Оформить продажу?}
    L -- да --> M[Отразить отпуск/продажу через кассу]
    L -- нет --> N
    M --> N[Экспорт накладной / отчёта в Excel]
    N --> O([Конец])
```

---

## 5. Диаграмма компонентов (Component)

```mermaid
flowchart LR
    subgraph UI_SOFT[UI-слой]
        UP[Модуль учёта продукции и рецептур]
        UW[Модуль складского учёта]
        UO[Модуль заказов и продаж]
        UR[Модуль формирования отчётов]
        UA[Модуль администрирования]
    end

    subgraph APP[Прикладной слой]
        SVC[Сервисы: продукция, рецептуры, склад]
        ORD[Сервис заказов и продаж]
        REP[Сервис отчётности]
        AUTH[Аутентификация и права]
        EXC[Экспорт в MS Excel]
        TTE[Интеграция с торговым оборудованием]
    end

    subgraph DATA[Слой данных]
        DB[(База данных до 1 млн записей)]
        BK[Резервное копирование / восстановление]
    end

    UP --> SVC
    UW --> SVC
    UO --> ORD
    UR --> REP
    UA --> AUTH
    REP --> EXC
    ORD --> TTE
    SVC --> DB
    ORD --> DB
    DB --> BK
    AUTH --> DB
```

---

## 6. Диаграмма развёртывания (Deployment)

```mermaid
flowchart TD
    subgraph Office[Локальная сеть пекарни]
        PC[Рабочие места пользователей]
        PC --> |"Windows 7/10/11, GUI"| APP["ПО «Управление пекарней»"]
        APP --> |сетевой доступ| SRV[Сервер БД]
        SRV --> DB[(База данных<br/>до 1 000 000 записей)]
        SRV --> BK[Система резервного копирования]
        APP --> PRN[Принтер<br/>печать накладных и отчётов]
        APP --> FR[Фискальный регистратор / ККМ]
        APP --> BS[Сканер штрихкодов]
        APP --> EXL[MS Excel 2010+<br/>экспорт отчётов]
    end
```

---

### Замечания
- Диаграммы построены по тексту ТЗ и описывают **логику из документа**, без привязки к фактическому коду.
- Точные атрибуты сущностей (поля классов) уточняются по образцам документов из Приложений 1–3 ТЗ (карточка изделия, справочник рецептур, приходная накладная на сырьё).