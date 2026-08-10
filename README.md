## US-GFECD Architecture

---

## Русский

### US-GFECD

US-GFECD — это архитектурный подход для realtime-приложений с разделением ответственности.

**Клиент (US):**
- **UI** — отвечает за отображение. Содержит элементы (атомарные компоненты), которые читают или изменяют данные.
- **Store** — управляет состоянием. Отвечает за хранение данных и отправку запросов на сервер.

**Сервер (GFECD):**
- **Gate** — входной адаптер. Принимает запросы от клиента и вызывает Flow.
- **Flow** — оркестратор. Координирует выполнение задач, вызывая Core, Db или Emit.
- **Core** — бизнес-логика. Чистые функции с правилами и расчётами.
- **Db** — доступ к данным. Работает с базой данных.
- **Emit** — исходящие события. Отправляет уведомления клиентам.

---

### Ключевые принципы

Слои изолированы друг от друга. Верхний слой не знает о внутреннем устройстве нижнего, а нижний — не знает о существовании верхнего. Это позволяет изменять реализацию любого слоя без влияния на другие.

---

### Общая схема

![Схема US-GFECD](./image.png)

Схема разделена на две части: клиент (US) и сервер (GFECD). Серые стрелки показывают внутренние связи между слоями. Зелёные стрелки обозначают прямые клиент-серверные взаимодействия: Entity и Invoke отправляют запросы в Gate, Emit отправляет события в Entity.

---

### Слои сервера (GFECD)

#### Gate

**Зачем:** отделить протокол (WebSocket/HTTP) от бизнес-логики.

**Что делает:** принимает запросы, проверяет авторизацию, валидирует входные данные, вызывает Flow.

**Что не делает:** не содержит бизнес-логики, не работает с БД, не отправляет события.

---

#### Flow

**Зачем:** координировать сложные бизнес-процессы.

**Что делает:** вызывает Core (логику), Db (данные), Emit (события). Управляет транзакциями.

**Что не делает:** не содержит бизнес-правил, не импортирует Gate.

---

#### Core

**Зачем:** изолировать правила и расчёты.

**Что делает:** содержит бизнес-правила, выполняет расчёты и валидацию. Работает с данными без побочных эффектов.

**Что не делает:** не импортирует Gate, Flow, Db, Emit, не работает с БД.

---

#### Db

**Зачем:** инкапсулировать работу с БД.

**Что делает:** читает и записывает данные. Отвечает на вопросы (например, `isUserInRoom`).

**Что не делает:** не содержит бизнес-логику, не оркестрирует запросы.

---

#### Emit

**Зачем:** отправлять события клиентам.

**Что делает:** отправляет события через WebSocket, поддерживает комнаты.

**Что не делает:** не содержит бизнес-логику, не импортирует Gate, Flow, Core, Db.

---

### Слои клиента (US)

#### UI

**Зачем:** отделить представление от данных и логики.

**Что делает:** отображает данные из Store, принимает ввод пользователя, передаёт команды в Store.

**Что не делает:** не содержит бизнес-логику, не работает с сервером напрямую, не хранит состояние (всё состояние в Store).

**Состав:**
- **Elements** — атомарные компоненты без логики (кнопки, карточки, поля ввода). Ничего не знают о Store.
- **View** — читают данные из Store. Только отображение. Не изменяют данные. Могут использовать Entity для инициализации (подписка на данные).
- **Edit** — изменяют данные через Store (вызывают Invoke). Не читают данные.
- **Page** — собирают View, Edit и Elements в страницу.

---

#### Store

**Зачем:** управлять состоянием и синхронизировать его с сервером.

**Что делает:** хранит данные, обрабатывает запросы, обновляет состояние при ответах от сервера.

**Что не делает:** не содержит бизнес-логику, не знает о UI.

**Состав:**
- **State** — данные и редьюсеры для их изменения. (Рекомендуется выносить редьюсеры в State, но это не обязательно.)
- **Entity** — объединяет State, инициализацию (загрузка данных) и подписки на обновления от сервера. Управляет жизненным циклом данных (init/clean).
- **Invoke** — отправка запросов на сервер. Получает ответ, обновляет State через редьюсеры.

> Подробнее о реализации Store в клиентской библиотеке: [@us-gfecd/client](https://npmjs.com/package/@us-gfecd/client)

---

### Потоки данных

#### Запрос

1. **UI** → **Store** (команда).
2. **Store** → **Gate** (запрос по сети через Socket.IO v4).
3. **Gate** → **Flow** (вызов).
4. **Flow** → **Core** (логика) и **Db** (данные).
5. **Flow** → **Emit** (опционально, если нужно уведомить других).
6. Ответ: **Flow** → **Gate** → **Store** → **UI**.

#### Событие

1. **Серверное событие** → **Flow**.
2. **Flow** → **Emit**.
3. **Emit** → **Entity** (на клиенте) через Socket.IO v4.
4. **Entity** → **State** (обновление).
5. **State** → **UI** (перерисовка).

---

## English

### US-GFECD Architecture

US-GFECD is an architectural approach for realtime applications with clear separation of responsibilities.

**Client (US):**
- **UI** — handles display. Contains elements (atomic components) that read or modify data.
- **Store** — manages state. Responsible for storing data and sending requests to the server.

**Server (GFECD):**
- **Gate** — entry adapter. Accepts requests from the client and calls Flow.
- **Flow** — orchestrator. Coordinates tasks by calling Core, Db, or Emit.
- **Core** — business logic. Pure functions with rules and calculations.
- **Db** — data access. Works with the database.
- **Emit** — outgoing events. Sends notifications to clients.

---

### Key Principles

Layers are isolated from each other. The upper layer does not know about the internal structure of the lower layer, and the lower layer does not know about the existence of the upper layer. This allows changing the implementation of any layer without affecting others.

---

### General Scheme

![US-GFECD Scheme](./image.png)

The scheme is divided into two parts: client (US) and server (GFECD). Gray arrows show internal connections between layers. Green arrows indicate direct client-server interactions: Entity and Invoke send requests to Gate, Emit sends events to Entity.

---

### Server Layers (GFECD)

#### Gate

**Why:** separate the protocol (WebSocket/HTTP) from business logic.

**What it does:** accepts requests, checks authorization, validates input data, calls Flow.

**What it doesn't do:** does not contain business logic, does not work with the database, does not send events.

---

#### Flow

**Why:** coordinate complex business processes.

**What it does:** calls Core (logic), Db (data), Emit (events). Manages transactions.

**What it doesn't do:** does not contain business rules, does not import Gate.

---

#### Core

**Why:** isolate rules and calculations.

**What it does:** contains business rules, performs calculations and validation. Works with data without side effects.

**What it doesn't do:** does not import Gate, Flow, Db, Emit, does not work with the database.

---

#### Db

**Why:** encapsulate database access.

**What it does:** reads and writes data. Answers questions (e.g., `isUserInRoom`).

**What it doesn't do:** does not contain business logic, does not orchestrate queries.

---

#### Emit

**Why:** send events to clients.

**What it does:** sends events via WebSocket, supports rooms.

**What it doesn't do:** does not contain business logic, does not import Gate, Flow, Core, Db.

---

### Client Layers (US)

#### UI

**Why:** separate presentation from data and logic.

**What it does:** displays data from Store, accepts user input, passes commands to Store.

**What it doesn't do:** does not contain business logic, does not work with the server directly, does not store state (all state is in Store).

**Composition:**
- **Elements** — atomic components without logic (buttons, cards, input fields). Know nothing about Store.
- **View** — read data from Store. Display only. Do not modify data. May use Entity for initialization (data subscription).
- **Edit** — modify data through Store (call Invoke). Do not read data.
- **Page** — assemble View, Edit, and Elements into a page.

---

#### Store

**Why:** manage state and synchronize it with the server.

**What it does:** stores data, processes requests, updates state upon server responses.

**What it doesn't do:** does not contain business logic, does not know about UI.

**Composition:**
- **State** — data and reducers for modifying it. (It's recommended to keep reducers in State, but not required.)
- **Entity** — combines State, initialization (data loading), and subscriptions to server updates. Manages data lifecycle (init/clean).
- **Invoke** — sends requests to the server. Receives responses, updates State through reducers.

> More details about Store implementation in the client library: [@us-gfecd/client](https://npmjs.com/package/@us-gfecd/client)

---

### Data Flows

#### Request

1. **UI** → **Store** (command).
2. **Store** → **Gate** (network request via Socket.IO v4).
3. **Gate** → **Flow** (call).
4. **Flow** → **Core** (logic) and **Db** (data).
5. **Flow** → **Emit** (optional, if other clients need to be notified).
6. Response: **Flow** → **Gate** → **Store** → **UI**.

#### Event

1. **Server event** → **Flow**.
2. **Flow** → **Emit**.
3. **Emit** → **Entity** (on the client) via Socket.IO v4.
4. **Entity** → **State** (update).
5. **State** → **UI** (re-render).