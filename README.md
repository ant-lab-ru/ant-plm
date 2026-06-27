# ant-plm

Веб-приложение для управления жизненным циклом изделий (PLM) — для небольших
студенческих и научных команд, которые ведут НИР и мелкосерийное производство.

Идея: проект ограничен во времени (НИР на разработку изделия или контракт на
выпуск конкретного количества). PLM избавляет от ручного заполнения актов
комплектации и даёт ответ на вопросы: **сколько каких компонентов осталось, по
какому проекту они куплены, хватает ли их на сборку изделия и что делать с
остатками при закрытии проекта.**

## Возможности (MVP)

- Единый справочник номенклатуры (изделия и компоненты) и состав изделия (BOM).
- Закупки и приход по УПД, склад с учётом партий и мест хранения (ledger).
- Заказ под заданное количество изделий: разузлование BOM, расчёт дефицита
  (надо − склад − заказано) и экспорт `order.xlsx` для поставщиков.
- Акты комплектации образцов (автоматическое списание по BOM).
- Проекты со сквозным потоком: закупки → комплектация → передача → сверка
  балансов и решение по остаткам.

## Стек

Django + Django REST Framework · React + TypeScript (Vite) · MySQL/MariaDB.
Рассчитано на shared-хостинг (напр. reg.ru): без Celery/Redis, тяжёлые операции
синхронно, периодика через cron.

## Быстрый старт

> ⚠️ В разработке — раздел будет дополнен, когда сложится рабочая сборка.

```
# backend
# frontend
```

## Модель данных

> Эти диаграммы — источник правды по модели. Любое изменение схемы БД должно
> сразу отражаться здесь.

### Продуктовая схема (как это работает)

```mermaid
flowchart TD
  P["Проект<br/>(НИР / контракт)"] --> D["Потребность:<br/>N изделий"]
  D --> B["Разузлование BOM<br/>→ потребность в компонентах"]
  B --> GAP["Дефицит к заказу:<br/>надо − склад − заказано"]
  GAP --> ORD["Заказ → order.xlsx<br/>(поставщикам, цены оффлайн)"]
  ORD --> UPD["Приход по УПД<br/>→ партии (Lot), закрывают строки заказа"]
  UPD --> W[("Склад:<br/>остатки по партиям и местам")]
  W --> K["Акт комплектации<br/>(списание по BOM)"]
  K --> T["Передача заказчику<br/>(по накладной)"]
  W --> R["Остатки по проекту"]
  T --> CLOSE["Закрытие проекта"]
  R --> CLOSE
  CLOSE --> DISP{"Судьба остатков"}
  DISP --> WO["Списать"]
  DISP --> COMMON["В общее пользование<br/>(проект не указан)"]
  DISP --> CUST["Передать заказчику"]
  DISP --> BAL["На баланс<br/>для будущей комплектации"]
```

### Техническая схема (структура БД)

```mermaid
erDiagram
  ITEM ||--o{ BOMLINE : "как parent"
  ITEM ||--o{ BOMLINE : "как component"
  ITEM ||--o{ LOT : "партии"
  ITEM ||--o{ PURCHASELINE : ""
  ITEM ||--o{ PROJECTDEMAND : "target"
  ITEM ||--o{ KITTINGACT : "target"
  ITEM ||--o{ KITTINGACTLINE : "component"
  ITEM ||--o{ TRANSFER : ""

  SUPPLIER ||--o{ RECEIPT : ""

  LOCATION ||--o{ STOCKMOVEMENT : ""
  LOCATION ||--o{ KITTINGACTLINE : ""

  LOT ||--o{ STOCKMOVEMENT : ""
  LOT ||--o{ KITTINGACTLINE : ""

  PROJECT ||--o{ PROJECTDEMAND : ""
  PROJECT ||--o{ STOCKMOVEMENT : "nullable"
  PROJECT ||--o{ PURCHASE : "nullable"
  PROJECT ||--o{ RECEIPT : "nullable"
  PROJECT ||--o{ KITTINGACT : ""
  PROJECT ||--o{ TRANSFER : ""

  PURCHASE ||--o{ PURCHASELINE : ""
  PURCHASE ||--o{ RECEIPT : "nullable"
  RECEIPT ||--o{ LOT : "создаёт партии (поставка)"
  KITTINGACT ||--o{ KITTINGACTLINE : ""
  KITTINGACT ||--o{ LOT : "изготовление"

  ITEM {
    int id PK
    string code "артикул, uniq"
    string name
    string kind "изделие/компонент/материал"
    string uom "ед. изм."
    bool is_manufactured
    bool active
  }
  BOMLINE {
    int id PK
    int parent_id FK
    int component_id FK
    decimal qty
    string position "опц."
  }
  SUPPLIER {
    int id PK
    string name
    string inn
  }
  LOCATION {
    int id PK
    string code
    string name
    string kind
  }
  LOT {
    int id PK
    int item_id FK
    int receipt_id FK "origin: поставка (nullable)"
    int kitting_act_id FK "origin: изготовление (nullable)"
    decimal unit_cost "цена закупки"
    string received_name "название из УПД"
  }
  STOCKMOVEMENT {
    int id PK
    int lot_id FK "item выводится из партии"
    int location_id FK
    int project_id FK "nullable"
    string type "RECEIPT/ISSUE/RETURN/TRANSFER/ADJUSTMENT"
    decimal qty "со знаком"
    string source_type "УПД/акт/передача"
    int source_id
    datetime created_at
  }
  PROJECT {
    int id PK
    string code
    string name
    string status
    date started_at
    date closed_at "nullable"
  }
  PROJECTDEMAND {
    int id PK
    int project_id FK
    int target_item_id FK
    decimal qty
  }
  PURCHASE {
    int id PK
    int project_id FK "nullable"
    string status "draft/sent/partial/received/cancelled"
    date date
    string note
  }
  PURCHASELINE {
    int id PK
    int purchase_id FK
    int item_id FK "uniq в рамках закупки"
    decimal qty "заказано"
  }
  RECEIPT {
    int id PK
    string number "УПД №"
    date date
    int supplier_id FK
    int purchase_id FK "nullable"
    int project_id FK "nullable"
  }
  KITTINGACT {
    int id PK
    int project_id FK
    int target_item_id FK
    decimal qty "кол-во образцов"
    date date
    string status
  }
  KITTINGACTLINE {
    int id PK
    int act_id FK
    int component_id FK
    int lot_id FK
    int location_id FK
    decimal qty
  }
  TRANSFER {
    int id PK
    int project_id FK
    int item_id FK
    decimal qty
    date date
    string number "накладная №"
  }
```

## Ключевые принципы модели

- **Единый `Item`**: изделия и компоненты — одна сущность; изделие может состоять
  из изделий (рекурсивный BOM через `BomLine`).
- **`Lot` — главная учётная единица.** Хранит цену закупки (`unit_cost`) и
  название из УПД (`received_name`); поставщик и дата берутся через `Receipt`.
  Каждый приход = новый `Lot` (заказы уникальны), отдельной строки документа нет.
- **`Lot` не возникает «из воздуха» — всегда есть origin-акт:** поставка
  (`Receipt`), изготовление (`KittingAct` = акт комплектации/изготовления) или
  инвентаризация (акт — планируется). Ровно один origin задан.
- **Склад — неизменяемый ledger** (`StockMovement`): двигается **только по
  `Lot`** (`item` выводится из партии); остаток = сумма движений в разрезе
  партии / места / проекта.
- **Проект — nullable** в движениях и закупках: `project IS NULL` = «общее
  пользование». Решение по остаткам при закрытии проекта (списать / в общее /
  заказчику / на баланс) — это просто новые движения, перепривязывающие или
  списывающие количество.
- **Закупки — лёгкий `Purchase`.** Закупка = список «что купить» (экспорт в
  `order.xlsx`); поставщик и цены — оффлайн, в системе их нет. Закупка
  «разрешается» в приходы: `PurchaseLine` закрывается одним или несколькими `Lot`
  через цепочку `Lot → Receipt → Purchase` (поставка 100 = 60 + 40 = две партии —
  норма). Привязка партии к строке выводится по `(purchase, item)`, поэтому пара
  `(purchase, item)` в `PurchaseLine` уникальна. Статус отражает покрытие
  (`draft → sent → partial → received`).
- **«Что ещё не заказано» и сверка балансов — отчёты поверх ledger**, а не
  отдельные мутабельные таблицы:
  `ещё заказать = надо (BOM×потребность) − склад (Lot) − заказано (открытые PurchaseLine)`.

## Лицензия

См. [LICENSE](LICENSE).
