# UML-диаграммы — «Управление пекарней»

По ТЗ из readme.md. Все диаграммы объединены единой цветовой темой «Пекарня» (тёплые оттенки пшеницы и зелени). Код вставляется в любой Mermaid-редактор (VSCode, GitHub, Mermaid Live Editor).

---

## 1. Варианты использования (Use Case)

```mermaid
%%{init: {"theme":"base", "themeVariables":{"background":"#FFF8EE","primaryColor":"#F5E6C8","primaryTextColor":"#4A3B22","primaryBorderColor":"#C9A95C","lineColor":"#A98A4E","tertiaryColor":"#E8F0E3"}}}%%
flowchart LR
    subgraph ROLES["Роли"]
        Tech["👨‍🍳 Технолог (кондитер)"]
        Klad["📦 Кладовщик"]
        Kas["💳 Кассир-продавец"]
        Adm["🛠 Администратор"]
    end

    subgraph FUNCS["Функции системы"]
        UC1["Вести карточки изделий"]
        UC2["Вести справочник рецептур"]
        UC3["Оформлять заказы на изготовление"]
        UC4["Списывать сырьё в производство"]
        UC5["Вести справочник сырья"]
        UC6["Приход сырья"]
        UC7["Инвентаризация"]
        UC8["Списывать просрочку"]
        UC9["Продажи через кассу"]
        UC10["Отпуск по заказам"]
        UC11["Управление пользователями"]
        UC12["Резервное копирование"]
        UC13["Настройка цен и тарифов"]
        UC14["Отчёт о выпуске продукции"]
        UC15["Отчёт о расходе сырья"]
        UC16["Отчёт о продажах и выручке"]
        UC17["Список позиций, требующих контроля"]
        EX["📊 Экспорт в MS Excel"]
        AU("🔒 Вход по паролю")
    end

    AU --- ROLES
    Tech --> UC1 & UC2 & UC3 & UC4 & UC14
    Klad --> UC5 & UC6 & UC7 & UC8 & UC15
    Kas --> UC9 & UC10 & UC16
    Adm --> UC11 & UC12 & UC13 & UC17
    UC14 & UC15 & UC16 --> EX

    classDef role fill:#C9A95C,stroke:#8A6A32,color:#FFF,font-weight:bold
    classDef func fill:#F5E6C8,stroke:#C9A95C,color:#4A3B22
    class Tech,Klad,Kas,Adm role
    class UC1,UC2,UC3,UC4,UC5,UC6,UC7,UC8,UC9,UC10,UC11,UC12,UC13,UC14,UC15,UC16,UC17,EX func
```

---

## 2. Классы (Class)

```mermaid
classDiagram

    class User {
        +int id
        +string login
        +string passwordHash
        +string role
        +string fullName
    }

    class Product {
        +int id
        +string name
        +string category
        +decimal weight
        +decimal price
        +date expiryDate
    }

    class Recipe {
        +int id
        +string operation
        +decimal outputPerUnit
    }

    class Ingredient {
        +int id
        +string name
        +string unit
        +decimal stock
    }

    class Order {
        +int id
        +string client
        +datetime dateTimeDelivery
        +string status
    }

    class Sale {
        +int id
        +decimal quantity
        +decimal amount
    }

    class WarehouseService {
        +receive(Ingredient, decimal)
        +writeOffProduction(Ingredient, decimal)
        +inventory(Ingredient, decimal)
        +writeOffExpired(Product)
    }

    class OrderService {
        +createOrder(Order)
        +issue(Order)
    }

    class SalesService {
        +sell(Product, decimal)
        +releaseByOrder(Order)
    }

    class ReportService {
        +rawMaterialReport()
        +salesReport()
        +controlPositionsReport()
    }

    Product "1" --> "0..*" Recipe : рецептура
    Recipe "0..*" --> "1..*" Ingredient : нормы закладки
    Product "1" --> "0..*" Sale
    Order "0..*" --> "1..*" Product
    User "1" --> "0..*" Sale

    WarehouseService ..> Ingredient : оперирует
    OrderService ..> Order : оперирует
    SalesService ..> Sale : оперирует
    ReportService ..> User : обслуживает
```

---

## 3. Последовательность — списание сырья в производство (Sequence)

```mermaid
%%{init: {"theme":"base","themeVariables":{"actorBkg":"#F5E6C8","actorBorder":"#C9A95C","actorTextColor":"#4A3B22","signalColor":"#A98A4E","signalTextColor":"#4A3B22","labelBoxBkgColor":"#E8F0E3","labelBoxBorderColor":"#7FA05C","labelTextColor":"#4A3B22","sequenceNumberColor":"#C9A95C"}}}%%
sequenceDiagram
    autonumber
    participant T as Технолог
    participant UI as UI (Склад)
    participant W as WarehouseService
    participant DB as База данных
    participant R as ReportService

    T->>UI: списание сырья (ингредиент, кол-во)
    UI->>W: writeOffProduction(ingredient, qty)
    W->>W: validateQuantity(qty)
    alt количество < 0
        W-->>UI: ошибка «некорректное количество»
    end
    W->>DB: SELECT остаток
    DB-->>W: остаток на складе
    alt остаток < кол-ва
        W-->>UI: ошибка «списание сверх остатка»
    end
    W->>DB: UPDATE остаток (BEGIN)
    W->>DB: INSERT операция склада (COMMIT)
    DB-->>W: ok
    W-->>UI: успех
    UI->>R: отчёт о расходе сырья
    R-->>T: результат
```

---

## 4. Деятельность — оформление заказа на изготовление (Activity)

```mermaid
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#F5E6C8","primaryBorderColor":"#C9A95C","primaryTextColor":"#4A3B22","lineColor":"#A98A4E","tertiaryColor":"#E8F0E3","noteBkgColor":"#FFF3D6","noteBorderColor":"#C9A95C"}}}%%
flowchart TD
    A(["Начало"]) --> B[Аутентификация]
    B --> C{Пароль верен?}
    C -->|нет| B
    C -->|да| D[Выбор изделия по рецептуре]
    D --> E[Ввод клиента, состава, даты выдачи]
    E --> F{Данные корректны?}
    F -->|нет| E
    F -->|да| G{Сырья достаточно?}
    G -->|нет| H[Сообщение: недостача сырья]
    H --> E
    G -->|да| I[Создать заказ: статус NEW]
    I --> J[Списать сырьё по нормам закладки]
    J --> K[Перевести заказ в IN_PRODUCTION → READY]
    K --> L{Продажа сразу?}
    L -->|да| M[Отразить отпуск через кассу]
    L -->|нет| N
    M --> N[Экспорт накладной/отчёта в Excel]
    N --> O(["Конец"])

    style A fill:#7FA05C,stroke:#4E6B33,color:#FFF
    style O fill:#7FA05C,stroke:#4E6B33,color:#FFF
    style C fill:#FFF3D6,stroke:#C9A95C,color:#4A3B22
    style F fill:#FFF3D6,stroke:#C9A95C,color:#4A3B22
    style G fill:#FFF3D6,stroke:#C9A95C,color:#4A3B22
    style L fill:#FFF3D6,stroke:#C9A95C,color:#4A3B22
    style H fill:#F2D0C9,stroke:#B3574E,color:#5A2320
```

---

## 5. Компоненты (Component)

```mermaid
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#E8F0E3","primaryBorderColor":"#7FA05C","primaryTextColor":"#33421F","lineColor":"#A98A4E","tertiaryColor":"#F5E6C8"}}}%%
flowchart TB
    subgraph UI["Пользовательский интерфейс"]
        U1[Продукция и рецептуры]
        U2[Складской учёт]
        U3[Заказы и продажи]
        U4[Отчёты]
        U5[Администрирование]
    end

    subgraph APP["Прикладная логика"]
        A1[Сервисы продукции и склада]
        A2[Сервис заказов и продаж]
        A3[Сервис отчётности]
        A4[Аутентификация и права]
        A5[Экспорт в Excel]
        A6[Торговое оборудование]
    end

    subgraph DATA["Данные"]
        D1[("База данных до 1 млн записей")]
        D2[Резервное копирование]
    end

    U1 & U2 --> A1
    U3 --> A2
    U4 --> A3
    U5 --> A4
    A3 --> A5
    A2 --> A6
    A1 --> D1
    A2 --> D1
    A4 --> D1
    D1 --> D2

    style UI fill:#F5E6C8,stroke:#C9A95C,color:#4A3B22
    style APP fill:#E8F0E3,stroke:#7FA05C,color:#33421F
    style DATA fill:#FFF3D6,stroke:#C9A95C,color:#4A3B22
```

---

## 6. Развёртывание (Deployment)

```mermaid
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#E8F0E3","primaryBorderColor":"#7FA05C","primaryTextColor":"#33421F","lineColor":"#A98A4E","tertiaryColor":"#F5E6C8"}}}%%
flowchart TB
    subgraph PCR["Пекарня (локальная сеть)"]
        PC["🖥 Рабочие места пользователей<br/>(Windows 7/10/11)"]
        SRV["🖥 Сервер БД"]
        DB[("База данных<br/>до 1 000 000 записей")]
        BK[/"Система резервного<br/>копирования"/]
        PRN["🖨 Принтер накладных"]
        FR["🧾 Фискальный регистратор / ККМ"]
        BS["📷 Сканер штрихкодов"]
        EXL["📊 MS Excel 2010+"]
        APP["ПО «Управление пекарней»"]
    end

    PC --> APP
    APP --> SRV
    SRV --> DB
    SRV --> BK
    APP --> PRN
    APP --> FR
    APP --> BS
    APP --> EXL

    style PCR fill:#FFF8EE,stroke:#C9A95C,color:#4A3B22
    style APP fill:#F5E6C8,stroke:#C9A95C,color:#4A3B22,font-weight:bold
```

---

### Замечания
- Это визуально другой стиль по сравнению с прошлым репозиторием: единая тема, `direction LR`, стилизация узлов и группировка по цвету. Набор типов диаграмм тот же.
- Диаграммы отражают логику ТЗ; точные атрибуты сущностей уточняются по образцам документов (карточка изделия, справочник рецептур, приходная накладная).