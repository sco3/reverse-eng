# Требования к ре-имплементации Rust MCP Runtime

**Версия:** 1.0
**Дата:** 2026-03-17
**Язык:** Русский

---

## Что это за модуль?

### Описание простым языком

**Одним предложением:** Этот Rust модуль - это **высокопроизводительный HTTP шлюз**, который находится между MCP клиентами (такими как AI ассистенты, IDE или LLM приложения) и Python бэкендом, обрабатывая трафик протокола MCP быстрее, пока Python занимается безопасностью.

**Что он на самом деле делает, шаг за шагом:**

1. **Слушает HTTP запросы** на порту 8787 (по умолчанию) от MCP клиентов
2. **Получает сообщения протокола MCP** - это JSON-RPC запросы типа:
   - "Дай мне список доступных инструментов" (`tools/list`)
   - "Вызови этот инструмент с этими аргументами" (`tools/call`)
   - "Дай мне этот ресурс" (`resources/read`)
   - "Дай мне этот промпт" (`prompts/get`)
   - "Инициализируй новую сессию" (`initialize`)
3. **Проверяет, разрешен ли запрос** вызывая эндпоинт аутентификации Python
4. **Решает, куда отправить запрос:**
   - **Обработать локально** (без сетевого вызова): `ping` → возвращает `{}` немедленно
   - **Запросить базу данных напрямую** (быстрый путь): `tools/list`, `resources/list`, `prompts/list` → запрашивает PostgreSQL, возвращает результаты
   - **Вызвать upstream сервер** (прямое выполнение): `tools/call` → вызывает фактический MCP сервер (например, Git сервер, файловый сервер и т.д.)
   - **Проксировать на Python** (резервный путь): Всё остальное → перенаправляет на Python бэкенд
5. **Управляет сессиями** - помнит, какой пользователь владеет какой сессией, переиспользует аутентификацию для скорости
6. **Транслирует ответы обратно** клиентам используя SSE (Server-Sent Events) для длительных запросов
7. **Собирает метрики** - считает попадания в кэш, ошибки аутентификации, отказы сессий и т.д.

**Чего он НЕ делает:**
- НЕ аутентифицирует пользователей сам (Python делает это)
- НЕ решает, что пользователям разрешено делать (Python делает это через RBAC)
- НЕ хранит инструменты/ресурсы/промпты (они в PostgreSQL)
- НЕ заменяет Python бэкенд (это ускоритель, а не замена)

**Аналогия:** Думайте об этом как о **CDN для MCP трафика**. Так же, как CDN кэширует статический контент, чтобы не обращаться к вашему веб-серверу, этот Rust runtime обрабатывает распространенные MCP запросы напрямую, чтобы не обращаться к Python бэкенду. Но для всего, что связано с безопасностью, он всё ещё обращается к Python.

---

## Как он решает, куда отправить запрос?

### Логика принятия решений о маршрутизации

Когда запрос поступает, Rust runtime проходит через **дерево решений** для определения обработчика. Вот точная логика:

```mermaid
flowchart TD
    A["📥 Запрос поступает<br/>POST /mcp<br/>JSON-RPC тело"] --> B{"ШАГ 1:<br/>method == ping?"}

    B -->|ДА| C["🏠 ОБРАБОТАТЬ ЛОКАЛЬНО<br/>Вернуть {}<br/>Статус: 200"]
    B -->|НЕТ| D{"ШАГ 2:<br/>notifications/*?"}

    D -->|"ДА - initialized/message/cancelled"| E["🔙 ПРОКСИ НА PYTHON<br/>Вызов бэкенда"]
    D -->|"ДА - другие"| F["🏠 ОБРАБОТАТЬ ЛОКАЛЬНО<br/>Catch-all<br/>Вернуть {}"]
    D -->|НЕТ| G{"ШАГ 3:<br/>Catch-all список?<br/>sampling/*, completion/*<br/>logging/*, elicitation/*"}

    G -->|ДА| H["🏠 ОБРАБОТАТЬ ЛОКАЛЬНО<br/>Вернуть {}"]
    G -->|НЕТ| I{"ШАГ 4:<br/>Direct DB возможен?<br/>1. server-scoped?<br/>2. DB настроен?<br/>3. Метод в списке?"}

    I -->|"ДА - tools/list"| J["💾 ПРЯМОЙ DB ЗАПРОС<br/>Запрос PostgreSQL<br/>Вернуть результаты"]
    I -->|"ДА - resources/list"| J
    I -->|"ДА - resources/read"| J
    I -->|"ДА - resource_templates/list"| J
    I -->|"ДА - prompts/list"| J
    I -->|"ДА - prompts/get"| J
    I -->|НЕТ| K{"ШАГ 5:<br/>method == tools/call?"}

    K -->|ДА| L["🔧 ПЛАН ВЫПОЛНЕНИЯ<br/>Вызов Python для плана<br/>Вызов upstream сервера"]
    K -->|НЕТ| M{"ШАГ 6:<br/>method == initialize?"}

    M -->|ДА| N["📋 SESSION CORE<br/>Создать запись сессии<br/>Прокси на Python"]
    M -->|НЕТ| O["🔙 ПО УМОЛЧАНИЮ: ПРОКСИ НА PYTHON<br/>POST /_internal/mcp/{method}"]

    style C fill:#90EE90
    style F fill:#90EE90
    style H fill:#90EE90
    style J fill:#87CEEB
    style L fill:#FFB6C1
    style N fill:#DDA0DD
    style E fill:#FFE4B5
    style O fill:#FFE4B5
```

### Таблица решений

| Метод | Условие | Обработчик | Почему |
|-------|---------|-----------|--------|
| `ping` | Всегда | **Локальный** | Нет состояния, нет аутентификации, возвращает `{}` |
| `initialize` | Всегда | **Session Core + Backend** | Создает запись сессии, затем проксирует на Python |
| `tools/list` | server-scoped + DB настроен | **Direct DB** | Может запросить PostgreSQL напрямую с фильтрацией по команде |
| `tools/list` | Не server-scoped | **Backend** | Нужна логика агрегации Python |
| `tools/call` | Всегда | **Upstream (через план Python)** | Python разрешает аутентификацию, Rust вызывает upstream сервер |
| `resources/list` | server-scoped + DB настроен | **Direct DB** | Может запросить PostgreSQL напрямую |
| `resources/read` | server-scoped + простые параметры + DB | **Direct DB** | Простой поиск по URI |
| `resources/read` | Сложные параметры или нет DB | **Backend** | Нужны хуки плагинов Python |
| `prompts/list` | server-scoped + DB настроен | **Direct DB** | Может запросить PostgreSQL напрямую |
| `prompts/get` | server-scoped + простые параметры + DB | **Direct DB** | Простой поиск по имени |
| `notifications/initialized` | Всегда | **Backend** | Нужны обработчики уведомлений Python |
| `notifications/message` | Всегда | **Backend** | Нужны обработчики уведомлений Python |
| `notifications/cancelled` | Всегда | **Backend** | Нужны обработчики уведомлений Python |
| `notifications/*` | Другие | **Локальный (catch-all)** | Возвращает пустой успех |
| `sampling/*`, `completion/*`, `logging/*`, `elicitation/*` | Большинство | **Локальный (catch-all)** | Возвращает пустой успех |
| Всё остальное | По умолчанию | **Backend** | Прокси на Python для безопасности |

### Типы обработчиков

```mermaid
flowchart LR
    subgraph Handlers["Типы обработчиков"]
        direction TB
        H1["🏠 Локальный<br/>Без внешних вызовов<br/>ping, catch-alls"]
        H2["💾 Direct DB<br/>PostgreSQL запрос<br/>tools/list, resources/*"]
        H3["🔧 Upstream вызов<br/>Напрямую к MCP серверу<br/>tools/call"]
        H4["🔙 Backend Proxy<br/>На Python internal<br/>initialize, notifications"]
        H5["📋 Session Core<br/>Создание сессии + Backend<br/>initialize"]
    end

    H1 --> R1["✅ Быстрее всего<br/>~1ms"]
    H2 --> R2["⚡ Быстро<br/>~10-50ms"]
    H3 --> R3["🚀 Средне<br/>~50-200ms"]
    H4 --> R4["🐌 Медленнее<br/>~100-500ms"]
    H5 --> R5["📊 Средне<br/>~50-100ms"]

    style H1 fill:#90EE90
    style H2 fill:#87CEEB
    style H3 fill:#FFB6C1
    style H4 fill:#FFE4B5
    style H5 fill:#DDA0DD
```

---

## Содержание

1. [Обзор](#1-обзор)
2. [Архитектура](#2-архитектура)
3. [Конечные точки и потоки запросов](#3-конечные-точки-и-потоки-запросов)
4. [Детальный поток обработки запросов](#4-детальный-поток-обработки-запросов)
5. [Операции с базой данных](#5-операции-с-базой-данных)
6. [Операции кэширования](#6-операции-кэширования)
7. [Сбор метрик](#7-сбор-метрик)
8. [Модели данных](#8-модели-данных)
9. [Конфигурация](#9-конфигурация)
10. [Обработка ошибок](#10-обработка-ошибок)

---

## 1. Обзор

**Назначение:** Этот документ специфицирует требования для ре-имплементации ContextForge Rust MCP Runtime на другом языке программирования. Rust MCP Runtime - это опциональный высокопроизводительный сайдкар, который обрабатывает публичный HTTP-трафик MCP (Model Context Protocol), делегируя аутентификацию, токенизацию и RBAC Python бэкенду.

**Что делает этот модуль:**

Rust MCP Runtime служит **высокопроизводительным HTTP edge** для трафика протокола MCP. Это НЕ полноценный MCP сервер - это **умный прокси и менеджер сессий**, который:

1. **Принимает публичные MCP запросы** (`GET/POST/DELETE /mcp`) от клиентов
2. **Аутентифицирует через Python** бэкенд (Python остается авторитетом аутентификации)
3. **Управляет сессиями** - отслеживает владение сессией, контекст аутентификации, область сервера
4. **Маршрутизирует запросы** к соответствующим обработчикам:
   - **Локальная обработка**: `ping`, catch-all уведомления
   - **Прямые DB запросы**: `tools/list`, `resources/list`, `prompts/list` (PostgreSQL)
   - **Бэкенд прокси**: `initialize`, `notifications/*`, `roots/*`
   - **Upstream вызовы**: `tools/call` (напрямую к upstream MCP серверам)
5. **Владеет опциональными ядрами** в режиме `full`:
   - Управление метаданными сессий
   - Хранилище событий на Redis
   - Live SSE стриминг
   - Возобновляемые GET потоки
   - Межворкерная affinity переадресация

**Ключевые проектные решения:**

- **Python - авторитет аутентификации**: Rust вызывает `POST /_internal/mcp/authenticate` Python для всех публичных запросов
- **Повторное использование аутентификации сессии**: После начальной аутентификации Rust может переиспользовать контекст аутентификации для той же сессии (ограничено TTL)
- **Развертывание на основе режимов**: `off` → `shadow` → `edge` → `full` контролирует владение Rust
- **Быстрый путь прямого выполнения**: `tools/call` может обходить Python для подходящих вызовов
- **Строгая изоляция сессий**: Попытки перехвата сессии отклоняются с детальной метрикой

---

## 2. Архитектура

### 2.1 Диаграмма архитектуры высокого уровня

```mermaid
flowchart TB
    subgraph Client["🖥️ Клиентский слой"]
        C1["MCP клиенты<br/>LLMs, IDE плагины"]
    end

    subgraph Nginx["⚖️ Nginx Reverse Proxy"]
        N1["Маршрутизирует /mcp на<br/>Rust edge|full или<br/>Python off|shadow"]
    end

    subgraph Rust["🦀 Rust MCP Runtime"]
        direction TB

        subgraph HTTP["HTTP слушатель (Axum)"]
            H1["Публичные эндпоинты:<br/>GET /health, /healthz<br/>GET/POST/DELETE /mcp<br/>/servers/{id}/mcp"]
            H2["Внутренние эндпоинты:<br/>POST /rpc<br/>POST /_internal/event-store/*"]
        end

        subgraph Pipeline["Конвейер обработки запросов"]
            P1["1. Валидация версии протокола"]
            P2["2. Валидация владения сессией"]
            P3["3. Проверка переиспользования аутентификации"]
            P4["4. Маршрутизация методов"]
        end

        subgraph Cores["Основные компоненты (режим full)"]
            direction TB
            C2["Session Core<br/>Мета сессии<br/>Привязка аутентификации<br/>Управление TTL"]
            C3["Event Store<br/>Redis hash<br/>Воспроизведение событий<br/>Номера последовательностей"]
            C4["Live Stream<br/>SSE edge<br/>Keep-alive<br/>Буфер фреймов"]
            C5["Resume Core<br/>last-event-id<br/>Опрос Redis<br/>Потоковая передача"]
            C6["Affinity Core<br/>Владелец воркер<br/>Redis pub/sub<br/>Переадресация запросов"]
        end

        subgraph Cache["Локальные кэши"]
            L1["Сессии runtime<br/>LRU, на основе TTL"]
            L2["Сессии upstream инструментов<br/>Пулинг соединений"]
            L3["Планы вызовов инструментов<br/>TTL: 30с"]
            L4["RMCP клиенты<br/>Опционально"]
        end
    end

    subgraph Python["🐍 Python Gateway Backend"]
        direction TB
        PY1["Доверенная внутренняя аутентификация<br/>POST /_internal/mcp/authenticate"]
        PY2["Внутренние MCP RPC эндпоинты<br/>/initialize, /transport<br/>/tools/*, /resources/*, /prompts/*"]
        PY3["Аутентификация, токенизация, RBAC<br/>Источник истины"]
    end

    subgraph Storage["💾 Слой хранения данных"]
        direction TB
        PG[("(PostgreSQL<br/>инструменты, ресурсы<br/>промпты, серверы)")]
        RD[("(Redis<br/>хранилище событий<br/>кэш сессий<br/>ключи affinity<br/>pub/sub)")]
        UP["Upstream MCP серверы<br/>HTTP/SSE"]
    end

    Client -->|"HTTP/1.1 или HTTP/2<br/>GET/POST/DELETE /mcp"| Nginx
    Nginx --> Rust
    Rust -->|"Доверенные внутренние вызовы"| Python
    Python --> Storage
    Rust --> Storage
```

### 2.2 Последовательность взаимодействия компонентов

#### Стандартный поток запроса POST /mcp

```mermaid
sequenceDiagram
    participant Client
    participant Nginx
    participant Rust as Rust Runtime
    participant Python as Python Backend
    participant DB as PostgreSQL
    participant Upstream as Upstream MCP

    Client->>Nginx: 1. POST /mcp<br/>JSON-RPC тело
    Nginx->>Rust: 2. Передача

    Note over Rust: 3. Валидация версии протокола
    Note over Rust: 4. Проверка сессии (если существует)
    Note over Rust: 5. Переиспользование аутентификации? (проверка TTL)

    Rust->>Python: 6. Внутренний вызов аутентификации
    Note over Python: 7. JWT верификация, область токена
    Python-->>Rust: 8. AuthContext

    Note over Rust: 9. Маршрутизация метода

    alt 9a. DB запрос
        Rust->>DB: 9a. Запрос
        DB-->>Rust: 9a. Результат
    else 9b. Backend
        Rust->>Python: 9b. Запрос
        Python-->>Rust: 9b. Ответ
    else 9c. Upstream
        Rust->>Upstream: 9c. Вызов
        Upstream-->>Rust: 9c. Результат
    end

    Nginx-->>Client: 11. Ответ
```

#### Поток инициализации сессии

```mermaid
sequenceDiagram
    participant Client
    participant Rust as Rust Runtime
    participant Python as Python Backend
    participant Upstream as Upstream MCP

    Client->>Rust: POST /mcp<br/>method: initialize<br/>session-id: S1

    Rust->>Python: 1. Аутентификация
    Python-->>Rust: 2. AuthContext

    Note over Rust: 3. Создать RuntimeSessionRecord<br/>- owner_email<br/>- server_id<br/>- auth_binding_fingerprint<br/>- encoded_auth_context<br/>- TTL expiry

    Note over Rust: 4. Сохранить в<br/>- In-memory кэш<br/>- Redis (если включен)

    Rust->>Python: 5. Передача initialize
    Python-->>Rust: 6. Ответ с<br/>mcp-session-id

    Rust-->>Client: 7. Ответ + session-id
```

#### Поток повторного использования сессии

```mermaid
sequenceDiagram
    participant Client
    participant Rust as Rust Runtime
    participant Python as Python Backend

    Client->>Rust: POST /mcp<br/>session-id: S1<br/>method: tools/list

    Note over Rust: 1. Поиск сессии S1<br/>в памяти

    Note over Rust: 2. Валидация<br/>- owner_email совпадает?<br/>- server_id совпадает?<br/>- auth_binding_fingerprint?<br/>- TTL истек?

    alt HIT - Переиспользовать auth_context
        Note over Rust: Пропустить вызов Python
    else MISS
        Rust->>Python: Вызов аутентификации Python
        Python-->>Rust: Обновление сессии
    end

    Note over Rust: 3. Маршрутизация на direct DB

    Rust-->>Client: 4. Ответ
```

---

## 3. Конечные точки и потоки запросов

### 3.1 Публичные конечные точки

| Метод | Endpoint | Описание | Тип обработчика |
|-------|----------|----------|-----------------|
| `GET` | `/health` | Проверка здоровья с полной статистикой | Local |
| `GET` | `/healthz` | Проверка здоровья (публичный, минимальный) | Local |
| `GET` | `/mcp` | SSE стриминг для live ответов | Core (Session/Live/Resume) |
| `POST` | `/mcp` | Обработка JSON-RPC запросов | Main RPC handler |
| `DELETE` | `/mcp` | Завершение сессии | Backend proxy |
| `GET` | `/servers/{server_id}/mcp` | Server-scoped SSE стриминг | Core + Server scope |
| `POST` | `/servers/{server_id}/mcp` | Server-scoped JSON-RPC | Main RPC + Server scope |
| `DELETE` | `/servers/{server_id}/mcp` | Server-scoped завершение сессии | Backend proxy |

### 3.2 Внутренние конечные точки

| Метод | Endpoint | Описание | Тип обработчика |
|-------|----------|----------|-----------------|
| `POST` | `/rpc` | Прямой JSON-RPC (без аутентификации) | Main RPC handler |
| `POST` | `/_internal/event-store/store` | Сохранение события в Redis | Event Store Core |
| `POST` | `/_internal/event-store/replay` | Воспроизведение событий из Redis | Event Store Core |

### 3.3 Таблица маршрутизации методов

| MCP метод | Тип обработчика | Описание |
|-----------|-----------------|----------|
| `ping` | **Local** | Возвращает пустой результат `{}` немедленно |
| `initialize` | **Backend** (с Session Core) | Пересылает на Python, создает запись сессии |
| `notifications/initialized` | **Backend** | Уведомление на Python бэкенд |
| `notifications/message` | **Backend** | Уведомление о сообщении на Python |
| `notifications/cancelled` | **Backend** | Уведомление об отмене на Python |
| `notifications/*` (другие) | **Local** | Возвращает пустой успех (catch-all) |
| `tools/list` | **Direct DB** (server-scoped) или **Backend** | Запрос PostgreSQL или прокси на Python |
| `tools/call` | **Upstream** (прямое выполнение) | Вызов upstream MCP сервера напрямую |
| `resources/list` | **Direct DB** (server-scoped) или **Backend** | Запрос PostgreSQL или прокси |
| `resources/read` | **Direct DB** (server-scoped, простые параметры) или **Backend** | Запрос PostgreSQL или прокси |
| `resources/subscribe` | **Backend** | Подписка на изменения ресурсов |
| `resources/unsubscribe` | **Backend** | Отписка от изменений ресурсов |
| `resources/templates/list` | **Direct DB** (server-scoped) или **Backend** | Запрос PostgreSQL или прокси |
| `prompts/list` | **Direct DB** (server-scoped) или **Backend** | Запрос PostgreSQL или прокси |
| `prompts/get` | **Direct DB** (server-scoped, простые параметры) или **Backend** | Запрос PostgreSQL или прокси |
| `roots/list` | **Backend** | Список клиентских roots |
| `completion/complete` | **Backend** | Предложения дополнения |
| `sampling/createMessage` | **Backend** | Запрос LLM sampling |
| `logging/setLevel` | **Backend** | Установка уровня логирования |
| `sampling/*`, `completion/*`, `logging/*`, `elicitation/*` | **Local** | Catch-all, возвращает пустой успех |

---

## 4. Детальный поток обработки запросов

### 4.1 Основной конвейер обработки запросов

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POST /mcp запрос поступает                           │
│                    Заголовки: authorization, mcp-session-id, и т.д.     │
│                    Тело: JSON-RPC {method, params, id, jsonrpc}         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 1: Валидация версии протокола                                      │
│  ─────────────────────────────────                                      │
│  • Извлечь заголовок mcp-protocol-version                               │
│  • Проверить список supported_protocol_versions                         │
│  • Если невалидно → Вернуть ошибку -32602 (Invalid Params)              │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 2: Валидация сессии (если session_core_enabled)                    │
│  ─────────────────────────────────────────────────                      │
│  • Извлечь mcp-session-id из заголовка или query param                  │
│  • Поиск сессии в кэше runtime_sessions                                 │
│  • Если не найдено → Вернуть 404 "Session not found"                    │
│  • Валидировать server_id совпадает (если server-scoped запрос)         │
│  • Валидировать auth_binding_fingerprint совпадает                      │
│  • Если несовпадение → Вернуть 403 "Session access denied"              │
│  • Инжектировать заголовки mcp-session-id и x-contextforge-server-id    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 3: Affinity переадресация (если affinity_core_enabled)             │
│  ─────────────────────────────────────────────────                      │
│  • Сгенерировать affinity ключ из session_id                            │
│  • Поиск owner_worker в Redis                                           │
│  • Если другой воркер → Опубликовать в Redis pub/sub                    │
│  • Ожидать ответа на уникальном канале ответа                           │
│  • Если получен переадресованный ответ → Вернуть клиенту                │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 4: Решение о маршрутизации метода                                  │
│  ─────────────────────────────────                                      │
│  Определение обработчика на основе метода:                              │
│  • ping → Local обработчик                                              │
│  • initialize → Session Core обработчик                                 │
│  • tools/list → Direct DB (если server-scoped + DB pool) или Backend    │
│  • tools/call → Upstream прямое выполнение                              │
│  • resources/*, prompts/* → Direct DB или Backend                       │
│  • notifications/* → Backend или Local (catch-all)                      │
│  • другие → Backend или Local (catch-all)                               │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 5: Выполнение обработчика                                          │
│  ─────────────────────────                                              │
│  См. разделы 5-7 для потоков конкретных обработчиков                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Шаг 6: Конструирование ответа                                          │
│  ───────────────────────────────                                        │
│  • Обернуть результат/ошибку в JSON-RPC конверт                         │
│  • Добавить заголовки runtime:                                          │
│    - x-contextforge-mcp-runtime: rust                                   │
│    - x-contextforge-mcp-session-core: rust|python                       │
│    - x-contextforge-mcp-event-store: rust|python                        │
│    - и т.д.                                                             │
│  • Переслать безопасные заголовки от бэкенда (content-type, mcp-session-id) │
│  • Вернуть клиенту                                                      │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
                              Ответ
```

### 4.2 Поток аутентификации

```mermaid
flowchart TD
    A["Старт:<br/>authenticate_public_request_if_needed"] --> B{"1. Запрос требует<br/>аутентификации?"}

    B -->|"Нет, внутренний"| C["Вернуть заголовки как есть"]
    B -->|"Да, публичный"| D["2. Удалить клиентские<br/>внутренние заголовки"]

    D --> E["3. Построить<br/>InternalAuthenticateRequest<br/>{method, path, query_string,<br/>headers, client_ip}"]

    E --> F["4. POST<br/>/_internal/mcp/authenticate<br/>→ Python Backend"]

    F --> G["5. Разобрать ответ<br/>InternalAuthenticateResponse<br/>{authContext: {email, teams,<br/>is_admin, is_authenticated}}"]

    G --> H["6. Закодировать auth_context<br/>как заголовок<br/>x-contextforge-auth-context: base64"]

    H --> I["Вернуть заголовки, path"]

    style D fill:#FFE4B5
    style F fill:#FFE4B5
    style H fill:#90EE90
```

### 4.3 Поток повторного использования аутентификации сессии

```mermaid
flowchart TD
    A["Старт:<br/>can_reuse_session_auth"] --> B{"1. session_auth_<br/>reuse включен?"}

    B -->|Нет| C["❌ MISS: отключено"]
    B -->|Да| D{"2. Сессия в<br/>кэше runtime_sessions?"}

    D -->|Нет| E["❌ MISS: no_session"]
    D -->|Да| F{"3. server_id<br/>совпадает?"}

    F -->|Нет| G["❌ MISS:<br/>server_scope_mismatch"]
    F -->|Да| H{"4. encoded_auth_<br/>context присутствует?"}

    H -->|Нет| I["❌ MISS:<br/>missing_encoded_auth_context"]
    H -->|Да| J{"5. auth_binding_<br/>fingerprint присутствует?"}

    J -->|Нет| K["❌ MISS:<br/>missing_auth_binding_fingerprint"]
    J -->|Да| L["6. Вычислить текущий<br/>fingerprint из заголовков<br/>SHA256 auth + server-id"]

    L --> M{"Fingerprint<br/>совпадает?"}
    M -->|Нет| N["❌ MISS:<br/>auth_binding_mismatch"]
    M -->|Да| O{"7. TTL не<br/>истек?"}

    O -->|Нет| P["❌ MISS: ttl_expired"]
    O -->|Да| Q["✅ HIT<br/>Вернуть encoded_auth_context"]

    style C fill:#FFB6C1
    style E fill:#FFB6C1
    style G fill:#FFB6C1
    style I fill:#FFB6C1
    style K fill:#FFB6C1
    style N fill:#FFB6C1
    style P fill:#FFB6C1
    style Q fill:#90EE90
```

### 4.4 Поток прямого DB запроса

```mermaid
flowchart TD
    A["Старт:<br/>direct_server_tools_list"] --> B["1. Получить DB соединение<br/>из пула"]

    B --> C{"2. Построить SQL запрос<br/>на основе auth контекста"}

    C -->|is_admin| D["SELECT * FROM tools<br/>WHERE server_id = $1"]
    C -->|non-admin| E["SELECT * FROM tools<br/>WHERE server_id = $1<br/>AND public = true<br/>OR team_id = ANY $2"]

    D --> F["3. Выполнить<br/>параметризированный запрос"]
    E --> F

    F --> G["4. Преобразовать строки в<br/>MCP Tool объекты"]

    G --> H["5. Построить JSON-RPC<br/>ответ"]

    H --> I["Вернуть ответ"]

    style C fill:#FFE4B5
    style F fill:#87CEEB
    style H fill:#90EE90
```

### 4.5 Поток прямого выполнения tools/call

```mermaid
flowchart TD
    A["Старт:<br/>handle_tools_call"] --> B["1. Разрешить план вызова инструмента<br/>от Python<br/>POST /tools/call/resolve"]

    B --> C["2. Проверить кэш для<br/>разрешенного плана"]

    C -->|"В кэше, не истек"| D["Использовать кэшированный план"]
    C -->|"Не в кэше"| E["3. Построить upstream<br/>ключ сессии"]

    D --> F["4. Получить или создать<br/>upstream сессию"]
    E --> F

    F -->|"Сессия в кэше"| G["Переиспользовать сессию"]
    F -->|"Нет сессии"| H["Создать новую сессию<br/>с вызовом initialize"]

    G --> I["5. Вызвать upstream сервер<br/>POST server_url<br/>tools/call"]
    H --> I

    I --> J["6. Разобрать<br/>upstream ответ"]

    J --> K["7. Отправить метрики<br/>на Python<br/>POST /tools/{id}/metric"]

    K --> L["8. Построить JSON-RPC<br/>ответ"]

    L --> M["Вернуть ответ"]

    style B fill:#FFE4B5
    style I fill:#FFB6C1
    style K fill:#90EE90
    style L fill:#90EE90
```

---

## 5. Операции с базой данных

### 5.1 Подключение к PostgreSQL

**Конфигурация:**
- Окружение: `MCP_RUST_DATABASE_URL`
- Формат: `postgresql://user:pass@host:port/dbname?sslmode=require`
- Пул соединений: `deadpool-postgres` (или эквивалент)
- Размер пула: 20 соединений (настраивается)
- TLS: Поддерживается через параметр `sslmode`

**Поддерживаемые SSL режимы:**
- `disable` - Без TLS
- `prefer` - TLS если доступен
- `require` - TLS требуется (по умолчанию для production)

**Примечание:** Аутентификация клиентскими сертификатами (`sslcert`, `sslkey`) пока НЕ поддерживается.

### 5.2 Прямые DB запросы

#### tools/list запрос

```sql
-- Admin пользователь (видит все инструменты)
SELECT
    id, name, description, input_schema, annotations,
    server_id, public, team_id, created_at, updated_at
FROM tools
WHERE server_id = $1
ORDER BY name;

-- Non-admin пользователь (видит публичные + командные инструменты)
SELECT
    id, name, description, input_schema, annotations,
    server_id, public, team_id, created_at, updated_at
FROM tools
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
-- $2 = ARRAY['team-a', 'team-b']
```

#### resources/list запрос

```sql
-- Non-admin пользователь
SELECT
    id, uri, name, description, mime_type,
    server_id, public, team_id, created_at, updated_at
FROM resources
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

#### resources/read запрос

```sql
SELECT
    id, uri, name, description, mime_type, content,
    server_id, public, team_id
FROM resources
WHERE server_id = $1
  AND uri = $2
  AND (public = true OR team_id = ANY($3))
LIMIT 1;
```

#### prompts/list запрос

```sql
-- Non-admin пользователь
SELECT
    id, name, description, arguments,
    server_id, public, team_id, created_at, updated_at
FROM prompts
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

#### prompts/get запрос

```sql
SELECT
    id, name, description, arguments,
    server_id, public, team_id
FROM prompts
WHERE server_id = $1
  AND name = $2
  AND (public = true OR team_id = ANY($3))
LIMIT 1;
```

#### resource_templates/list запрос

```sql
SELECT
    id, uri_template, name, description, mime_type,
    server_id, public, team_id
FROM resource_templates
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

### 5.3 Требования к DB операциям

| ID | Требование | Детали |
|----|------------|--------|
| DB-1 | Пулинг соединений | Использовать пул соединений, макс 20 соединений |
| DB-2 | Параметризированные запросы | ВСЕ запросы ДОЛЖНЫ использовать параметры для предотвращения SQL инъекций |
| DB-3 | Фильтрация видимости по команде | Фильтровать по `public = true OR team_id = ANY($teams)` |
| DB-4 | Обход для админов | Администраторы пропускают фильтрацию по команде |
| DB-5 | Область сервера | Все запросы фильтруются по `server_id` |
| DB-6 | Поддержка TLS | Поддержка PostgreSQL TLS через `sslmode` |
| DB-7 | Обработка ошибок | Ошибки DB → 502 Bad Gateway с отредактированным сообщением |
| DB-8 | Таймаут | Таймаут запроса: 30 секунд (настраивается) |

---

## 6. Операции кэширования

### 6.1 Уровни кэширования

```mermaid
flowchart TB
    subgraph L1["L1: In-Memory кэш<br/>HashMap + Mutex"]
        direction TB
        M1["runtime_sessions<br/>HashMap<String,<br/>RuntimeSessionRecord>"]
        M2["upstream_tool_sessions<br/>HashMap<String,<br/>UpstreamToolSession>"]
        M3["resolved_tool_call_plans<br/>HashMap<String,<br/>CachedResolvedToolCallPlan>"]
        M4["rmcp_upstream_clients<br/>HashMap<String,<br/>CachedRmcpUpstreamClient>"]
        M5["Истечение на основе TTL<br/>+ периодический очиститель"]
    end

    subgraph L2["L2: Redis кэш<br/>межворкерное разделение"]
        direction TB
        R1["Записи сессий<br/>{prefix}:rust:mcp:session:{id}"]
        R2["Хранилище событий<br/>{prefix}:eventstore:{stream_id}<br/>Redis Hash"]
        R3["Ключи Affinity<br/>{prefix}:affinity:{session_id}"]
        R4["Владелец пула<br/>{prefix}:pool_owner:{session_id}"]
        R5["Очистка на основе SCAN<br/>не KEYS"]
    end

    L1 <-->|"Синхронизация при чтении/записи"| L2

    style M1 fill:#90EE90
    style M2 fill:#90EE90
    style M3 fill:#90EE90
    style M4 fill:#90EE90
    style R1 fill:#87CEEB
    style R2 fill:#87CEEB
    style R3 fill:#87CEEB
    style R4 fill:#87CEEB
```

### 6.2 Паттерны ключей Redis

| Паттерн ключа | Пример | TTL | Описание |
|---------------|--------|-----|----------|
| `{prefix}:rust:mcp:session:{session_id}` | `mcpgw:rust:mcp:session:abc-123` | 3600с | Запись сессии runtime |
| `{prefix}:eventstore:{stream_id}` | `mcpgw:eventstore:abc-123` | 3600с | Hash хранилища событий |
| `{prefix}:affinity:{session_id}` | `mcpgw:affinity:abc-123` | 3600с | Воркер владелец affinity |
| `{prefix}:pool_owner:{session_id}` | `mcpgw:pool_owner:abc-123` | 3600с | Владелец пула сессии |
| `{prefix}:rust:tool-plan:{hash}` | `mcpgw:rust:tool-plan:sha256...` | 30с | Кэшированный план вызова инструмента |

### 6.3 Операции кэширования

#### Операции кэша сессий

```rust
// Получить сессию (проверяет TTL)
async fn get_runtime_session(state: &AppState, session_id: &str) -> Option<RuntimeSessionRecord> {
    // 1. Проверить in-memory кэш
    let sessions = state.runtime_sessions.lock().await;
    if let Some(record) = sessions.get(session_id) {
        if record.last_used.elapsed() < state.session_ttl() {
            return Some(record.clone());
        }
    }

    // 2. Проверить Redis
    let redis = state.redis().await?;
    let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
    let stored: Option<StoredRuntimeSessionRecord> = redis.get(&key).await.ok()?;

    // 3. Перестроить запись с локальным таймингом
    let record = RuntimeSessionRecord::from(stored?);

    // 4. Обновить in-memory кэш
    sessions.insert(session_id.to_string(), record.clone());

    Some(record)
}

// Upsert сессии (сохраняет в оба кэша)
async fn upsert_runtime_session(state: &AppState, session_id: String, record: RuntimeSessionRecord) {
    // 1. Сохранить в in-memory
    state.runtime_sessions.lock().await.insert(session_id.clone(), record.clone());

    // 2. Сохранить в Redis
    if let Some(redis) = state.redis().await {
        let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
        let stored = StoredRuntimeSessionRecord::from(&record);
        let _: () = redis.set(&key, stored).await.ok();
        let _: () = redis.expire(&key, state.session_ttl().as_secs() as i64).await.ok();
    }
}

// Удалить сессию
async fn remove_runtime_session(state: &AppState, session_id: &str) {
    // 1. Удалить из in-memory
    state.runtime_sessions.lock().await.remove(session_id);

    // 2. Удалить из Redis
    if let Some(redis) = state.redis().await {
        let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
        let _: () = redis.del(&key).await.ok();
    }
}
```

#### Операции хранилища событий

```rust
// Сохранить событие (атомарный Lua скрипт)
async fn store_event_in_rust_event_store(
    state: &AppState,
    request: EventStoreStoreRequest,
) -> Result<String, Response> {
    let redis = state.redis().await
        .ok_or_else(|| json_response(501, json!({"detail": "Redis недоступен"})))?;

    let prefix = event_store_key_prefix(state, request.key_prefix.as_deref());
    let stream_key = format!("{}:{}", prefix, request.stream_id);

    // Lua скрипт для атомарной операции
    let script = r#"
        local stream_key = KEYS[1]
        local event_id = ARGV[1]
        local seq_num = tonumber(ARGV[2])
        local message = ARGV[3]
        local max_events = tonumber(ARGV[4])
        local ttl = tonumber(ARGV[5])

        -- Добавить событие в hash
        redis.call('HSET', stream_key, event_id, message)

        -- Отслеживать последовательность
        redis.call('HINCRBY', stream_key .. ':index', event_id, seq_num)

        -- Обрезать если превышает макс
        local count = redis.call('HLEN', stream_key)
        if count > max_events then
            -- Удалить старейшие события (упрощенно)
        end

        -- Установить TTL
        redis.call('EXPIRE', stream_key, ttl)
        redis.call('EXPIRE', stream_key .. ':index', ttl)

        return event_id
    "#;

    let event_id = Uuid::new_v4().to_string();
    let seq_num = get_next_seq_num(&redis, &stream_key).await?;
    let message = request.message.map(|m| m.to_string()).unwrap_or("null".to_string());

    Script::new(script)
        .key(&stream_key)
        .arg(&event_id)
        .arg(seq_num)
        .arg(&message)
        .arg(request.max_events_per_stream.unwrap_or(100))
        .arg(request.ttl_seconds.unwrap_or(3600))
        .invoke_async(&mut redis)
        .await
        .map_err(|e| json_response(500, json!({"detail": "Ошибка Redis"})))
}

// Воспроизвести события
async fn replay_events_from_rust_event_store(
    state: &AppState,
    request: EventStoreReplayRequest,
) -> Result<EventStoreReplayResponse, Response> {
    let redis = state.redis().await
        .ok_or_else(|| json_response(501, json!({"detail": "Redis недоступен"})))?;

    let prefix = event_store_key_prefix(state, request.key_prefix.as_deref());

    // Получить stream_id и seq_num из last_event_id
    let (stream_id, last_seq) = get_stream_and_seq_from_event_id(&request.last_event_id)?;
    let stream_key = format!("{}:{}", prefix, stream_id);

    // Получить все события с seq > last_seq
    let entries: HashMap<String, String> = redis.hgetall(&stream_key).await?;
    let index: HashMap<String, i64> = redis.hgetall(&format!("{}:index", stream_key)).await?;

    // Отфильтровать и отсортировать события
    let mut events: Vec<EventStoreReplayEvent> = entries
        .into_iter()
        .filter_map(|(event_id, message)| {
            let seq = index.get(&event_id)?;
            if *seq > last_seq {
                Some(EventStoreReplayEvent {
                    event_id,
                    message: serde_json::from_str(&message).ok()?,
                })
            } else {
                None
            }
        })
        .collect();

    events.sort_by_key(|e| index.get(&e.event_id).copied().unwrap_or(0));

    Ok(EventStoreReplayResponse {
        stream_id: Some(stream_id),
        events,
    })
}
```

### 6.4 Требования к кэшированию

| ID | Требование | Детали |
|----|------------|--------|
| C-1 | In-memory кэш | LRU-стиль с защитой Mutex |
| C-2 | Redis кэш | Для межворкерного разделения сессий |
| C-3 | Истечение на основе TTL | Все записи имеют TTL |
| C-4 | Периодический очиститель | Фоновая задача для очистки истекших записей |
| C-5 | Хеширование ключей кэша | SHA256 для сложных ключей |
| C-6 | Атомарные операции Redis | Использовать Lua скрипты для атомарности |
| C-7 | SCAN не KEYS | Использовать SCAN для очистки (production-safe) |
| C-8 | Настраиваемый префикс | Префикс ключей Redis (по умолчанию: `mcpgw:`) |

---

## 7. Сбор метрик

### 7.1 Структура статистики выполнения

```json
{
  "session_auth_reuse": {
    "hits": 1000,
    "misses": 50,
    "backend_auth_round_trips": 100,
    "miss_disabled": 0,
    "miss_no_session": 30,
    "miss_server_scope_mismatch": 5,
    "miss_missing_encoded_auth_context": 5,
    "miss_missing_auth_binding_fingerprint": 0,
    "miss_auth_binding_mismatch": 10,
    "miss_ttl_expired": 0
  },
  "session_access_denials": {
    "server_scope_mismatches": 15,
    "missing_auth_context": 10,
    "owner_email_mismatches": 20,
    "missing_auth_binding_fingerprint": 5,
    "auth_binding_mismatches": 25
  },
  "affinity": {
    "forward_attempts": 100,
    "forwarded_requests": 95
  }
}
```

### 7.2 Точки сбора метрик

#### Метрики повторного использования аутентификации сессии

```rust
fn record_session_auth_reuse_hit(&self) {
    self.session_auth_reuse_hits.fetch_add(1, Ordering::Relaxed);
}

fn record_session_auth_reuse_miss(&self, reason: SessionAuthReuseMissReason) {
    self.session_auth_reuse_misses.fetch_add(1, Ordering::Relaxed);
    match reason {
        SessionAuthReuseMissReason::Disabled => {
            self.session_auth_reuse_miss_disabled.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::NoSession => {
            self.session_auth_reuse_miss_no_session.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::ServerScopeMismatch => {
            self.session_auth_reuse_miss_server_scope_mismatch.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::MissingEncodedAuthContext => {
            self.session_auth_reuse_miss_missing_encoded_auth_context.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::MissingAuthBindingFingerprint => {
            self.session_auth_reuse_miss_missing_auth_binding_fingerprint.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::AuthBindingMismatch => {
            self.session_auth_reuse_miss_auth_binding_mismatch.fetch_add(1, Ordering::Relaxed);
        }
        SessionAuthReuseMissReason::TtlExpired => {
            self.session_auth_reuse_miss_ttl_expired.fetch_add(1, Ordering::Relaxed);
        }
    }
}

fn record_session_auth_backend_round_trip(&self) {
    self.session_auth_backend_round_trips.fetch_add(1, Ordering::Relaxed);
}
```

#### Метрики отказа доступа к сессии

```rust
fn record_session_access_denial(&self, reason: SessionAccessDenyReason) {
    match reason {
        SessionAccessDenyReason::MissingAuthContext => {
            self.session_access_missing_auth_context.fetch_add(1, Ordering::Relaxed);
        }
        SessionAccessDenyReason::OwnerEmailMismatch => {
            self.session_access_owner_email_mismatches.fetch_add(1, Ordering::Relaxed);
        }
        SessionAccessDenyReason::MissingAuthBindingFingerprint => {
            self.session_access_missing_auth_binding_fingerprint.fetch_add(1, Ordering::Relaxed);
        }
        SessionAccessDenyReason::AuthBindingMismatch => {
            self.session_access_auth_binding_mismatches.fetch_add(1, Ordering::Relaxed);
        }
    }
}

fn record_session_server_scope_mismatch(&self) {
    self.session_access_server_scope_mismatches.fetch_add(1, Ordering::Relaxed);
}
```

#### Метрики affinity

```rust
fn record_affinity_forward_attempt(&self) {
    self.affinity_forward_attempts.fetch_add(1, Ordering::Relaxed);
}

fn record_affinity_forwarded_request(&self) {
    self.affinity_forwarded_requests.fetch_add(1, Ordering::Relaxed);
}
```

### 7.3 Предоставление метрик

Метрики предоставляются через:
1. **Эндпоинт `GET /health`** - Полная статистика в JSON ответе
2. **Заголовки ответа** - Заголовки идентификации runtime
3. **Структурированные логи** - Логирование каждого запроса с методом и режимом

Пример ответа health:
```json
{
  "status": "ok",
  "runtime": "rust",
  "active_sessions": 42,
  "runtime_stats": { ... }
}
```

### 7.4 Требования к метрикам

| ID | Требование | Детали |
|----|------------|--------|
| M-1 | Атомарные счетчики | Использовать атомарные операции для потокобезопасности |
| M-2 | Все причины miss | Отслеживать все причины miss переиспользования аутентификации сессии |
| M-3 | Все причины отказа | Отслеживать все причины отказа доступа к сессии |
| M-4 | Отслеживание affinity | Отслеживать попытки переадресации и успехи |
| M-5 | Поездки к бэкенду | Считать вызовы аутентификации Python |
| M-6 | Эндпоинт health | Предоставлять всю статистику в `/health` |
| M-7 | Заголовки runtime | Добавлять заголовки `x-contextforge-mcp-*` к ответам |
| M-8 | Структурированное логирование | Логировать метод и режим для каждого запроса |

---

## 8. Модели данных

### 8.1 RuntimeSessionRecord

```json
{
  "owner_email": "user@example.com",
  "server_id": "server-123",
  "protocol_version": "2025-11-25",
  "client_capabilities": {"roots": {"listChanged": true}},
  "encoded_auth_context": "base64_encoded_json",
  "auth_binding_fingerprint": "sha256_hash_of_auth_headers",
  "auth_context_expires_at_epoch_ms": 1234567890000,
  "created_at": "instant_timestamp",
  "last_used": "instant_timestamp"
}
```

**Сохранено в Redis** (исключает локальный тайминг):
```json
{
  "owner_email": "user@example.com",
  "server_id": "server-123",
  "protocol_version": "2025-11-25",
  "client_capabilities": {"roots": {"listChanged": true}},
  "encoded_auth_context": "base64_encoded_json",
  "auth_binding_fingerprint": "sha256_hash",
  "auth_context_expires_at_epoch_ms": 1234567890000
}
```

### 8.2 InternalAuthContext

```json
{
  "email": "user@example.com",
  "teams": ["team-a", "team-b"],
  "is_admin": false,
  "is_authenticated": true
}
```

### 8.3 ResolvedMcpToolCallPlan

```json
{
  "eligible": true,
  "fallback_reason": null,
  "tool_id": "tool-123",
  "server_id": "server-456",
  "server_url": "http://upstream-server/mcp",
  "remote_tool_name": "git_commit",
  "headers": {
    "authorization": "Bearer ..."
  },
  "timeout_ms": 30000,
  "transport": "sse",
  "parsed_headers": [(HeaderName, HeaderValue)],
  "headers_hash": 1234567890
}
```

### 8.4 UpstreamToolSession

```json
{
  "session_id": "upstream-session-uuid",
  "last_used": "instant_timestamp"
}
```

### 8.5 EventStoreReplayEvent

```json
{
  "event_id": "event-uuid",
  "message": {"jsonrpc": "2.0", "result": {...}}
}
```

### 8.6 RuntimeStatsSnapshot

```json
{
  "session_auth_reuse": {
    "hits": 1000,
    "misses": 50,
    "backend_auth_round_trips": 100,
    "miss_disabled": 0,
    "miss_no_session": 30,
    "miss_server_scope_mismatch": 5,
    "miss_missing_encoded_auth_context": 5,
    "miss_missing_auth_binding_fingerprint": 0,
    "miss_auth_binding_mismatch": 10,
    "miss_ttl_expired": 0
  },
  "session_access_denials": {
    "server_scope_mismatches": 15,
    "missing_auth_context": 10,
    "owner_email_mismatches": 20,
    "missing_auth_binding_fingerprint": 5,
    "auth_binding_mismatches": 25
  },
  "affinity": {
    "forward_attempts": 100,
    "forwarded_requests": 95
  }
}
```

---

## 9. Конфигурация

### 9.1 Аргументы CLI и переменные окружения

| Аргумент CLI | Переменная окружения | По умолчанию | Описание |
|--------------|---------------------|--------------|----------|
| `--backend-rpc-url` | `MCP_RUST_BACKEND_RPC_URL` | `http://127.0.0.1:4444/rpc` | Эндпоинт RPC бэкенда |
| `--listen-http` | `MCP_RUST_LISTEN_HTTP` | `127.0.0.1:8787` | Адрес прослушивания HTTP |
| `--listen-uds` | `MCP_RUST_LISTEN_UDS` | - | Путь Unix domain socket |
| `--public-listen-http` | `MCP_RUST_PUBLIC_LISTEN_HTTP` | - | Публичный адрес прослушивания HTTP |
| `--protocol-version` | `MCP_RUST_PROTOCOL_VERSION` | `2025-11-25` | Основная версия протокола |
| `--supported-protocol-version` | `MCP_RUST_SUPPORTED_PROTOCOL_VERSIONS` | (по умолчанию) | Поддерживаемые версии через запятую |
| `--server-name` | `MCP_RUST_SERVER_NAME` | `ContextForge` | Имя сервера |
| `--server-version` | `MCP_RUST_SERVER_VERSION` | (из package) | Версия сервера |
| `--instructions` | `MCP_RUST_INSTRUCTIONS` | (текст по умолчанию) | Инструкции сервера |
| `--request-timeout-ms` | `MCP_RUST_REQUEST_TIMEOUT_MS` | `30000` | Таймаут запроса в мс |
| `--client-connect-timeout-ms` | `MCP_RUST_CLIENT_CONNECT_TIMEOUT_MS` | `5000` | Таймаут подключения клиента |
| `--client-pool-idle-timeout-seconds` | `MCP_RUST_CLIENT_POOL_IDLE_TIMEOUT_SECONDS` | `90` | Таймаут простоя пула |
| `--client-pool-max-idle-per-host` | `MCP_RUST_CLIENT_POOL_MAX_IDLE_PER_HOST` | `1024` | Макс соединений простоя на хост |
| `--client-tcp-keepalive-seconds` | `MCP_RUST_CLIENT_TCP_KEEPALIVE_SECONDS` | `30` | Интервал TCP keepalive |
| `--tools-call-plan-ttl-seconds` | `MCP_RUST_TOOLS_CALL_PLAN_TTL_SECONDS` | `30` | TTL кэша плана вызова инструмента |
| `--upstream-session-ttl-seconds` | `MCP_RUST_UPSTREAM_SESSION_TTL_SECONDS` | `300` | TTL upstream сессии |
| `--use-rmcp-upstream-client` | `MCP_RUST_USE_RMCP_UPSTREAM_CLIENT` | `false` | Использовать клиент RMCP библиотеки |
| `--session-core-enabled` | `MCP_RUST_SESSION_CORE_ENABLED` | `false` | Включить session core |
| `--event-store-enabled` | `MCP_RUST_EVENT_STORE_ENABLED` | `false` | Включить event store |
| `--resume-core-enabled` | `MCP_RUST_RESUME_CORE_ENABLED` | `false` | Включить resume core |
| `--live-stream-core-enabled` | `MCP_RUST_LIVE_STREAM_CORE_ENABLED` | `false` | Включить live stream core |
| `--affinity-core-enabled` | `MCP_RUST_AFFINITY_CORE_ENABLED` | `false` | Включить affinity core |
| `--session-auth-reuse-enabled` | `MCP_RUST_SESSION_AUTH_REUSE_ENABLED` | `false` | Включить переиспользование аутентификации сессии |
| `--session-auth-reuse-ttl-seconds` | `MCP_RUST_SESSION_AUTH_REUSE_TTL_SECONDS` | `30` | TTL переиспользования аутентификации сессии |
| `--session-ttl-seconds` | `MCP_RUST_SESSION_TTL_SECONDS` | `3600` | TTL сессии |
| `--event-store-max-events-per-stream` | `MCP_RUST_EVENT_STORE_MAX_EVENTS_PER_STREAM` | `100` | Макс событий на поток |
| `--event-store-ttl-seconds` | `MCP_RUST_EVENT_STORE_TTL_SECONDS` | `3600` | TTL хранилища событий |
| `--event-store-poll-interval-ms` | `MCP_RUST_EVENT_STORE_POLL_INTERVAL_MS` | `250` | Интервал опроса хранилища событий |
| `--cache-prefix` | `MCP_RUST_CACHE_PREFIX` | `mcpgw:` | Префикс ключей кэша Redis |
| `--database-url` | `MCP_RUST_DATABASE_URL` | - | URL подключения PostgreSQL |
| `--redis-url` | `MCP_RUST_REDIS_URL` | - | URL подключения Redis |
| `--db-pool-max-size` | `MCP_RUST_DB_POOL_MAX_SIZE` | `20` | Макс размер пула DB |
| `--log-filter` | `MCP_RUST_LOG` | `info` | Фильтр уровня логирования |

### 9.2 Пресеты режимов

| Режим | Публичный `/mcp` | Session Core | Event Store | Resume | Live Stream | Affinity | Auth Reuse |
|-------|------------------|--------------|-------------|--------|-------------|----------|------------|
| `off` | Python | Нет | Нет | Нет | Нет | Нет | Нет |
| `shadow` | Python | Нет | Нет | Нет | Нет | Нет | Нет |
| `edge` | Rust | Нет | Нет | Нет | Нет | Нет | Да |
| `full` | Rust | Да | Да | Да | Да | Да | Да |

---

## 10. Обработка ошибок

### 10.1 Коды ошибок JSON-RPC

| Код | Сообщение | Когда |
|-----|-----------|-------|
| `-32600` | Invalid Request | Невалидный формат JSON-RPC |
| `-32601` | Method Not Found | Неизвестный MCP метод |
| `-32602` | Invalid Params | Отсутствующие/невалидные параметры |
| `-32603` | Internal Error | Внутренняя ошибка Rust |
| `-32000` | Server error | Ошибка бэкенда/Rust (общая) |
| `-32003` | Access denied | Отказ сессии/аутентификации |

### 10.2 Формат ответа об ошибке

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": {
    "code": -32003,
    "message": "Session access denied",
    "data": "See server logs"
  }
}
```

### 10.3 Сокращение ошибок

**Ошибки видимые клиенту:**
- Детали внутренней ошибки РЕДАКТИРУЮТСЯ
- Возвращать общее сообщение: `"See server logs"` или `"CLIENT_ERROR_DETAIL"`

**Логи на стороне сервера:**
- Полные детали ошибки логируются
- Включать трассировки стека для отладки

### 10.4 Сопоставление статусов HTTP

| Ошибка JSON-RPC | HTTP статус |
|-----------------|-------------|
| `-32600` | 400 Bad Request |
| `-32601` | 404 Not Found |
| `-32602` | 400 Bad Request |
| `-32603` | 500 Internal Server Error |
| `-32000` | 502 Bad Gateway |
| `-32003` | 403 Forbidden |
| Session not found | 404 Not Found |
| Auth failure | 401 Unauthorized |

### 10.5 Требования к обработке ошибок

| ID | Требование | Детали |
|----|------------|--------|
| E-1 | Формат JSON-RPC | Все ошибки в формате JSON-RPC 2.0 |
| E-2 | Коды ошибок | Использовать стандартные коды ошибок JSON-RPC |
| E-3 | Сокращение для клиента | Никогда не раскрывать внутренние ошибки клиентам |
| E-4 | Логирование на сервере | Логировать полные детали ошибок на сервере |
| E-5 | Сопоставление HTTP | Сопоставлять ошибки JSON-RPC со статусами HTTP |
| E-6 | Обработка таймаутов | Обрабатывать таймауты с соответствующей ошибкой |
| E-7 | Ошибки соединения | Обрабатывать сбои соединения корректно |
| E-8 | Поведение fallback | Переключаться на Python при ошибках Rust когда безопасно |

---

## 11. История версий

| Версия | Дата | Автор | Изменения |
|--------|------|---------|-----------|
| 1.0 | 2026-03-17 | ContextForge Team | Начальные требования |

---

## 12. Ссылки

- [README Rust MCP Runtime](../../tools_rust/mcp_runtime/README.md)
- [РАЗРАБОТКА Rust MCP Runtime](../../tools_rust/mcp_runtime/DEVELOPING.md)
- [ДИЗАЙН ТЕСТИРОВАНИЯ Rust MCP Runtime](../../tools_rust/mcp_runtime/TESTING-DESIGN.md)
- [СТАТУС Rust MCP Runtime](../../tools_rust/mcp_runtime/STATUS.md)
- [Архитектура: Rust MCP Runtime](../../docs/docs/architecture/rust-mcp-runtime.md)
- [ADR-043: Сайдкар Rust MCP Runtime](../../docs/docs/architecture/adr/043-rust-mcp-runtime-sidecar-mode-model.md)
