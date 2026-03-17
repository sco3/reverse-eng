# Rust MCP Runtime Re-implementation Requirements

**Version:** 1.0
**Date:** 2026-03-17
**Language:** English

---

## What Is This Module?

### Plain Language Description

**In one sentence:** This Rust module is a **high-performance HTTP gateway** that sits between MCP clients (like AI assistants, IDEs, or LLM applications) and a Python backend, handling the MCP protocol traffic faster while letting Python handle security.

**What it actually does, step by step:**

1. **Listens for HTTP requests** on port 8787 (by default) from MCP clients
2. **Receives MCP protocol messages** - these are JSON-RPC requests like:
   - "Give me the list of available tools" (`tools/list`)
   - "Call this tool with these arguments" (`tools/call`)
   - "Give me this resource" (`resources/read`)
   - "Give me this prompt" (`prompts/get`)
   - "Initialize a new session" (`initialize`)
3. **Checks if the request is allowed** by calling Python's authentication endpoint
4. **Decides where to send the request:**
   - **Handle locally** (no network call): `ping` → returns `{}` immediately
   - **Query database directly** (fast path): `tools/list`, `resources/list`, `prompts/list` → queries PostgreSQL, returns results
   - **Call upstream server** (direct execution): `tools/call` → calls the actual MCP server (like a Git server, file server, etc.)
   - **Proxy to Python** (fallback): Everything else → forwards to Python backend
5. **Manages sessions** - remembers which user owns which session, reuses authentication for speed
6. **Streams responses back** to clients using SSE (Server-Sent Events) for long-running requests
7. **Collects metrics** - counts cache hits, auth failures, session denials, etc.

**What it does NOT do:**
- Does NOT authenticate users itself (Python does that)
- Does NOT decide what users are allowed to do (Python does that via RBAC)
- Does NOT store tools/resources/prompts (they're in PostgreSQL)
- Does NOT replace the Python backend (it's an accelerator, not a replacement)

**Analogy:** Think of it like a **CDN for MCP traffic**. Just as a CDN caches static content to avoid hitting your web server, this Rust runtime handles common MCP requests directly to avoid hitting the Python backend. But for anything security-sensitive, it still calls back to Python.

---

## How Does It Decide Where to Send a Request?

### Request Routing Decision Logic

When a request arrives, the Rust runtime goes through a **decision tree** to determine the handler. Here's the exact logic:

```mermaid
flowchart TD
    A["📥 Request arrives<br/>POST /mcp<br/>JSON-RPC body"] --> B{"STEP 1:<br/>method == ping?"}

    B -->|YES| C["🏠 HANDLE LOCALLY<br/>Return {}<br/>Status: 200"]
    B -->|NO| D{"STEP 2:<br/>notifications/*?"}

    D -->|"YES - initialized/message/cancelled"| E["🔙 PROXY TO PYTHON<br/>Backend call"]
    D -->|"YES - other"| F["🏠 HANDLE LOCALLY<br/>Catch-all<br/>Return {}"]
    D -->|NO| G{"STEP 3:<br/>Catch-all list?<br/>sampling/*, completion/*<br/>logging/*, elicitation/*"}

    G -->|YES| H["🏠 HANDLE LOCALLY<br/>Return {}"]
    G -->|NO| I{"STEP 4:<br/>Direct DB eligible?<br/>1. server-scoped?<br/>2. DB configured?<br/>3. Method in list?"}

    I -->|"YES - tools/list"| J["💾 DIRECT DB QUERY<br/>Query PostgreSQL<br/>Return results"]
    I -->|"YES - resources/list"| J
    I -->|"YES - resources/read"| J
    I -->|"YES - resource_templates/list"| J
    I -->|"YES - prompts/list"| J
    I -->|"YES - prompts/get"| J
    I -->|NO| K{"STEP 5:<br/>method == tools/call?"}

    K -->|YES| L["🔧 RESOLVE PLAN<br/>Call Python for exec plan<br/>Call upstream server"]
    K -->|NO| M{"STEP 6:<br/>method == initialize?"}

    M -->|YES| N["📋 SESSION CORE<br/>Create session record<br/>Proxy to Python"]
    M -->|NO| O["🔙 DEFAULT: PROXY TO PYTHON<br/>POST /_internal/mcp/{method}"]

    style C fill:#90EE90
    style F fill:#90EE90
    style H fill:#90EE90
    style J fill:#87CEEB
    style L fill:#FFB6C1
    style N fill:#DDA0DD
    style E fill:#FFE4B5
    style O fill:#FFE4B5
```

### Decision Table

| Method | Condition | Handler | Why |
|--------|-----------|---------|-----|
| `ping` | Always | **Local** | No state, no auth needed, returns `{}` |
| `initialize` | Always | **Session Core + Backend** | Creates session record, then proxies to Python |
| `tools/list` | server-scoped + DB configured | **Direct DB** | Can query PostgreSQL directly with team filtering |
| `tools/list` | Not server-scoped | **Backend** | Needs Python's aggregation logic |
| `tools/call` | Always | **Upstream (via Python plan)** | Python resolves auth, Rust calls upstream server |
| `resources/list` | server-scoped + DB configured | **Direct DB** | Can query PostgreSQL directly |
| `resources/read` | server-scoped + simple params + DB | **Direct DB** | Simple lookup by URI |
| `resources/read` | Complex params or no DB | **Backend** | Needs Python's plugin hooks |
| `prompts/list` | server-scoped + DB configured | **Direct DB** | Can query PostgreSQL directly |
| `prompts/get` | server-scoped + simple params + DB | **Direct DB** | Simple lookup by name |
| `notifications/initialized` | Always | **Backend** | Needs Python's notification handlers |
| `notifications/message` | Always | **Backend** | Needs Python's notification handlers |
| `notifications/cancelled` | Always | **Backend** | Needs Python's notification handlers |
| `notifications/*` | Other | **Local (catch-all)** | Returns empty success |
| `sampling/*`, `completion/*`, `logging/*`, `elicitation/*` | Most | **Local (catch-all)** | Returns empty success |
| Everything else | Default | **Backend** | Proxy to Python for safety |

### Handler Types

```mermaid
flowchart LR
    subgraph Handlers["Handler Types"]
        direction TB
        H1["🏠 Local Handler<br/>No external calls<br/>ping, catch-alls"]
        H2["💾 Direct DB<br/>PostgreSQL query<br/>tools/list, resources/*"]
        H3["🔧 Upstream Call<br/>Direct to MCP server<br/>tools/call"]
        H4["🔙 Backend Proxy<br/>To Python internal<br/>initialize, notifications"]
        H5["📋 Session Core<br/>Create session + Backend<br/>initialize"]
    end

    H1 --> R1["✅ Fastest<br/>~1ms"]
    H2 --> R2["⚡ Fast<br/>~10-50ms"]
    H3 --> R3["🚀 Medium<br/>~50-200ms"]
    H4 --> R4["🐌 Slower<br/>~100-500ms"]
    H5 --> R5["📊 Medium<br/>~50-100ms"]

    style H1 fill:#90EE90
    style H2 fill:#87CEEB
    style H3 fill:#FFB6C1
    style H4 fill:#FFE4B5
    style H5 fill:#DDA0DD
```

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Endpoints and Request Flows](#3-endpoints-and-request-flows)
4. [Detailed Request Processing Flow](#4-detailed-request-processing-flow)
5. [Database Operations](#5-database-operations)
6. [Cache Operations](#6-cache-operations)
7. [Metrics Collection](#7-metrics-collection)
8. [Data Models](#8-data-models)
9. [Configuration](#9-configuration)
10. [Error Handling](#10-error-handling)

---

## 1. Overview

**Purpose:** This document specifies requirements for re-implementing the ContextForge Rust MCP Runtime in another programming language. The Rust MCP Runtime is an optional high-performance sidecar that handles public MCP (Model Context Protocol) HTTP traffic while delegating authentication, token scoping, and RBAC to the Python backend.

**What This Module Does:**

The Rust MCP Runtime serves as a **high-performance HTTP edge** for MCP protocol traffic. It is NOT a full MCP server - it's a **smart proxy and session manager** that:

1. **Accepts public MCP requests** (`GET/POST/DELETE /mcp`) from clients
2. **Authenticates through Python** backend (Python remains auth authority)
3. **Manages sessions** - tracks session ownership, auth context, server scope
4. **Routes requests** to appropriate handlers:
   - **Local handling**: `ping`, catch-all notifications
   - **Direct DB queries**: `tools/list`, `resources/list`, `prompts/list` (PostgreSQL)
   - **Backend proxy**: `initialize`, `notifications/*`, `roots/*`
   - **Upstream calls**: `tools/call` (direct to upstream MCP servers)
5. **Owns optional cores** in `full` mode:
   - Session metadata management
   - Redis-backed event store
   - Live SSE streaming
   - Resumable GET streams
   - Cross-worker affinity forwarding

**Key Design Decisions:**

- **Python is auth authority**: Rust calls Python's `POST /_internal/mcp/authenticate` for all public requests
- **Session-bound auth reuse**: After initial auth, Rust can reuse auth context for same session (TTL-bounded)
- **Mode-based deployment**: `off` → `shadow` → `edge` → `full` controls Rust ownership
- **Direct execution fast path**: `tools/call` can bypass Python for eligible calls
- **Strict session isolation**: Session hijacking attempts are denied with detailed metrics

---

## 2. Architecture

### 2.1 High-Level Architecture Diagram

```mermaid
flowchart TB
    subgraph Client["🖥️ Client Layer"]
        C1["MCP Clients<br/>LLMs, IDE plugins"]
    end

    subgraph Nginx["⚖️ Nginx Reverse Proxy"]
        N1["Routes /mcp to<br/>Rust edge|full or<br/>Python off|shadow"]
    end

    subgraph Rust["🦀 Rust MCP Runtime"]
        direction TB

        subgraph HTTP["HTTP Listener (Axum)"]
            H1["Public Endpoints:<br/>GET /health, /healthz<br/>GET/POST/DELETE /mcp<br/>/servers/{id}/mcp"]
            H2["Internal Endpoints:<br/>POST /rpc<br/>POST /_internal/event-store/*"]
        end

        subgraph Pipeline["Request Processing Pipeline"]
            P1["1. Protocol version validation"]
            P2["2. Session ownership validation"]
            P3["3. Auth context reuse check"]
            P4["4. Method routing"]
        end

        subgraph Cores["Core Components (full mode)"]
            direction TB
            C2["Session Core<br/>Session meta<br/>Auth binding<br/>TTL mgmt"]
            C3["Event Store<br/>Redis hash<br/>Event replay<br/>Seq numbers"]
            C4["Live Stream<br/>SSE edge<br/>Keep-alive<br/>Frame buffer"]
            C5["Resume Core<br/>last-event-id<br/>Poll Redis<br/>Tail stream"]
            C6["Affinity Core<br/>Owner worker<br/>Redis pub/sub<br/>Forward req"]
        end

        subgraph Cache["Local Caches"]
            L1["Runtime sessions<br/>LRU, TTL-based"]
            L2["Upstream tool sessions<br/>Connection pooling"]
            L3["Tool call plans<br/>TTL: 30s"]
            L4["RMCP clients<br/>Optional"]
        end
    end

    subgraph Python["🐍 Python Gateway Backend"]
        direction TB
        PY1["Trusted Internal Auth<br/>POST /_internal/mcp/authenticate"]
        PY2["Internal MCP RPC Endpoints<br/>/initialize, /transport<br/>/tools/*, /resources/*, /prompts/*"]
        PY3["Auth, Token Scoping, RBAC<br/>Source of Truth"]
    end

    subgraph Storage["💾 Data Storage Layer"]
        direction TB
        PG[("(PostgreSQL<br/>tools, resources<br/>prompts, servers)")]
        RD[("(Redis<br/>event store<br/>session cache<br/>affinity keys<br/>pub/sub)")]
        UP["Upstream MCP Servers<br/>HTTP/SSE"]
    end

    Client -->|"HTTP/1.1 or HTTP/2<br/>GET/POST/DELETE /mcp"| Nginx
    Nginx --> Rust
    Rust -->|"Trusted internal calls"| Python
    Python --> Storage
    Rust --> Storage
```

### 2.2 Component Interaction Sequence

#### Standard POST /mcp Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant Nginx
    participant Rust as Rust Runtime
    participant Python as Python Backend
    participant DB as PostgreSQL
    participant Upstream as Upstream MCP

    Client->>Nginx: 1. POST /mcp<br/>JSON-RPC body
    Nginx->>Rust: 2. Forward

    Note over Rust: 3. Validate protocol version
    Note over Rust: 4. Check session (if exists)
    Note over Rust: 5. Auth reuse? (TTL check)

    Rust->>Python: 6. Internal auth call
    Note over Python: 7. JWT verify, token scope
    Python-->>Rust: 8. AuthContext

    Note over Rust: 9. Route method

    alt 9a. DB Query
        Rust->>DB: 9a. Query
        DB-->>Rust: 9a. Result
    else 9b. Backend
        Rust->>Python: 9b. Request
        Python-->>Rust: 9b. Response
    else 9c. Upstream
        Rust->>Upstream: 9c. Call
        Upstream-->>Rust: 9c. Result
    end

    Nginx-->>Client: 11. Response
```

#### Session Initialization Flow

```mermaid
sequenceDiagram
    participant Client
    participant Rust as Rust Runtime
    participant Python as Python Backend
    participant Upstream as Upstream MCP

    Client->>Rust: POST /mcp<br/>method: initialize<br/>session-id: S1

    Rust->>Python: 1. Authenticate
    Python-->>Rust: 2. AuthContext

    Note over Rust: 3. Create RuntimeSessionRecord<br/>- owner_email<br/>- server_id<br/>- auth_binding_fingerprint<br/>- encoded_auth_context<br/>- TTL expiry

    Note over Rust: 4. Store in<br/>- In-memory cache<br/>- Redis (if enabled)

    Rust->>Python: 5. Forward initialize
    Python-->>Rust: 6. Response with<br/>mcp-session-id

    Rust-->>Client: 7. Response + session-id
```

#### Session Reuse Flow

```mermaid
sequenceDiagram
    participant Client
    participant Rust as Rust Runtime
    participant Python as Python Backend

    Client->>Rust: POST /mcp<br/>session-id: S1<br/>method: tools/list

    Note over Rust: 1. Lookup session S1<br/>in-memory

    Note over Rust: 2. Validate<br/>- owner_email matches?<br/>- server_id matches?<br/>- auth_binding_fingerprint?<br/>- TTL expired?

    alt HIT - Reuse auth_context
        Note over Rust: Skip Python call
    else MISS
        Rust->>Python: Call Python auth
        Python-->>Rust: Update session
    end

    Note over Rust: 3. Route to direct DB

    Rust-->>Client: 4. Response
```

---

## 3. Endpoints and Request Flows

### 3.1 Public Endpoints

| Method | Endpoint | Description | Handler Type |
|--------|----------|-------------|--------------|
| `GET` | `/health` | Health check with full stats | Local |
| `GET` | `/healthz` | Health check (public, minimal) | Local |
| `GET` | `/mcp` | SSE streaming for live responses | Core (Session/Live/Resume) |
| `POST` | `/mcp` | JSON-RPC request processing | Main RPC handler |
| `DELETE` | `/mcp` | Session termination | Backend proxy |
| `GET` | `/servers/{server_id}/mcp` | Server-scoped SSE streaming | Core + Server scope |
| `POST` | `/servers/{server_id}/mcp` | Server-scoped JSON-RPC | Main RPC + Server scope |
| `DELETE` | `/servers/{server_id}/mcp` | Server-scoped session termination | Backend proxy |

### 3.2 Internal Endpoints

| Method | Endpoint | Description | Handler Type |
|--------|----------|-------------|--------------|
| `POST` | `/rpc` | Direct JSON-RPC (no auth) | Main RPC handler |
| `POST` | `/_internal/event-store/store` | Store event in Redis | Event Store Core |
| `POST` | `/_internal/event-store/replay` | Replay events from Redis | Event Store Core |

### 3.3 Method Routing Table

| MCP Method | Handler Type | Description |
|------------|--------------|-------------|
| `ping` | **Local** | Returns empty result `{}` immediately |
| `initialize` | **Backend** (with Session Core) | Forwards to Python, creates session record |
| `notifications/initialized` | **Backend** | Notification to Python backend |
| `notifications/message` | **Backend** | Message notification to Python |
| `notifications/cancelled` | **Backend** | Cancellation notification to Python |
| `notifications/*` (other) | **Local** | Returns empty success (catch-all) |
| `tools/list` | **Direct DB** (server-scoped) or **Backend** | Query PostgreSQL or proxy to Python |
| `tools/call` | **Upstream** (direct execution) | Call upstream MCP server directly |
| `resources/list` | **Direct DB** (server-scoped) or **Backend** | Query PostgreSQL or proxy |
| `resources/read` | **Direct DB** (server-scoped, simple params) or **Backend** | Query PostgreSQL or proxy |
| `resources/subscribe` | **Backend** | Subscribe to resource changes |
| `resources/unsubscribe` | **Backend** | Unsubscribe from resource changes |
| `resources/templates/list` | **Direct DB** (server-scoped) or **Backend** | Query PostgreSQL or proxy |
| `prompts/list` | **Direct DB** (server-scoped) or **Backend** | Query PostgreSQL or proxy |
| `prompts/get` | **Direct DB** (server-scoped, simple params) or **Backend** | Query PostgreSQL or proxy |
| `roots/list` | **Backend** | List client roots |
| `completion/complete` | **Backend** | Completion suggestions |
| `sampling/createMessage` | **Backend** | LLM sampling request |
| `logging/setLevel` | **Backend** | Set logging level |
| `sampling/*`, `completion/*`, `logging/*`, `elicitation/*` | **Local** | Catch-all, returns empty success |

---

## 4. Detailed Request Processing Flow

### 4.1 Main Request Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POST /mcp Request Arrives                            │
│                    Headers: authorization, mcp-session-id, etc.         │
│                    Body: JSON-RPC {method, params, id, jsonrpc}         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 1: Protocol Version Validation                                    │
│  ─────────────────────────────────                                      │
│  • Extract mcp-protocol-version header                                  │
│  • Check against supported_protocol_versions list                       │
│  • If invalid → Return error -32602 (Invalid Params)                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 2: Session Validation (if session_core_enabled)                   │
│  ─────────────────────────────────────────────────                      │
│  • Extract mcp-session-id from header or query param                    │
│  • Lookup session in runtime_sessions cache                             │
│  • If not found → Return 404 "Session not found"                        │
│  • Validate server_id matches (if server-scoped request)                │
│  • Validate auth_binding_fingerprint matches                            │
│  • If mismatch → Return 403 "Session access denied"                     │
│  • Inject mcp-session-id and x-contextforge-server-id headers           │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 3: Affinity Forwarding (if affinity_core_enabled)                 │
│  ─────────────────────────────────────────────────                      │
│  • Generate affinity key from session_id                                │
│  • Lookup owner_worker in Redis                                         │
│  • If different worker → Publish to Redis pub/sub                       │
│  • Wait for response on unique response channel                         │
│  • If forwarded response received → Return to client                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 4: Method Routing Decision                                        │
│  ─────────────────────────────────                                      │
│  Determine handler based on method:                                     │
│  • ping → Local handler                                                 │
│  • initialize → Session Core handler                                    │
│  • tools/list → Direct DB (if server-scoped + DB pool) or Backend       │
│  • tools/call → Upstream direct execution                               │
│  • resources/*, prompts/* → Direct DB or Backend                        │
│  • notifications/* → Backend or Local (catch-all)                       │
│  • others → Backend or Local (catch-all)                                │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 5: Execute Handler                                                │
│  ─────────────────────────                                              │
│  See sections 5-7 for handler-specific flows                            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 6: Response Construction                                          │
│  ───────────────────────────────                                        │
│  • Wrap result/error in JSON-RPC envelope                               │
│  • Add runtime headers:                                                 │
│    - x-contextforge-mcp-runtime: rust                                   │
│    - x-contextforge-mcp-session-core: rust|python                       │
│    - x-contextforge-mcp-event-store: rust|python                        │
│    - etc.                                                               │
│  • Forward safe headers from backend (content-type, mcp-session-id)     │
│  • Return to client                                                     │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
                              Response
```

### 4.2 Authentication Flow

```mermaid
flowchart TD
    A["Start:<br/>authenticate_public_request_if_needed"] --> B{"1. Request needs<br/>authentication?"}

    B -->|"No, internal"| C["Return headers as-is"]
    B -->|"Yes, public"| D["2. Strip client<br/>internal headers"]

    D --> E["3. Build<br/>InternalAuthenticateRequest<br/>{method, path, query_string,<br/>headers, client_ip}"]

    E --> F["4. POST<br/>/_internal/mcp/authenticate<br/>→ Python Backend"]

    F --> G["5. Parse response<br/>InternalAuthenticateResponse<br/>{authContext: {email, teams,<br/>is_admin, is_authenticated}}"]

    G --> H["6. Encode auth_context<br/>as header<br/>x-contextforge-auth-context: base64"]

    H --> I["Return headers, path"]

    style D fill:#FFE4B5
    style F fill:#FFE4B5
    style H fill:#90EE90
```

### 4.3 Session Auth Reuse Flow

```mermaid
flowchart TD
    A["Start:<br/>can_reuse_session_auth"] --> B{"1. session_auth_<br/>reuse enabled?"}

    B -->|No| C["❌ MISS: disabled"]
    B -->|Yes| D{"2. Session in<br/>runtime_sessions cache?"}

    D -->|No| E["❌ MISS: no_session"]
    D -->|Yes| F{"3. server_id<br/>matches?"}

    F -->|No| G["❌ MISS:<br/>server_scope_mismatch"]
    F -->|Yes| H{"4. encoded_auth_<br/>context present?"}

    H -->|No| I["❌ MISS:<br/>missing_encoded_auth_context"]
    H -->|Yes| J{"5. auth_binding_<br/>fingerprint present?"}

    J -->|No| K["❌ MISS:<br/>missing_auth_binding_fingerprint"]
    J -->|Yes| L["6. Compute current<br/>fingerprint from headers<br/>SHA256 auth + server-id"]

    L --> M{"Fingerprint<br/>matches?"}
    M -->|No| N["❌ MISS:<br/>auth_binding_mismatch"]
    M -->|Yes| O{"7. TTL not<br/>expired?"}

    O -->|No| P["❌ MISS: ttl_expired"]
    O -->|Yes| Q["✅ HIT<br/>Return encoded_auth_context"]

    style C fill:#FFB6C1
    style E fill:#FFB6C1
    style G fill:#FFB6C1
    style I fill:#FFB6C1
    style K fill:#FFB6C1
    style N fill:#FFB6C1
    style P fill:#FFB6C1
    style Q fill:#90EE90
```

### 4.4 Direct DB Query Flow

```mermaid
flowchart TD
    A["Start:<br/>direct_server_tools_list"] --> B["1. Get DB connection<br/>from pool"]

    B --> C{"2. Build SQL query<br/>based on auth context"}

    C -->|is_admin| D["SELECT * FROM tools<br/>WHERE server_id = $1"]
    C -->|non-admin| E["SELECT * FROM tools<br/>WHERE server_id = $1<br/>AND public = true<br/>OR team_id = ANY $2"]

    D --> F["3. Execute<br/>parameterized query"]
    E --> F

    F --> G["4. Map rows to<br/>MCP Tool objects"]

    G --> H["5. Build JSON-RPC<br/>response"]

    H --> I["Return response"]

    style C fill:#FFE4B5
    style F fill:#87CEEB
    style H fill:#90EE90
```

### 4.5 Direct tools/call Execution Flow

```mermaid
flowchart TD
    A["Start:<br/>handle_tools_call"] --> B["1. Resolve tool call plan<br/>from Python<br/>POST /tools/call/resolve"]

    B --> C["2. Check cache for<br/>resolved plan"]

    C -->|"Cached, not expired"| D["Use cached plan"]
    C -->|"Not cached"| E["3. Build upstream<br/>session key"]

    D --> F["4. Get or create<br/>upstream session"]
    E --> F

    F -->|"Session cached"| G["Reuse session"]
    F -->|"No session"| H["Create new session<br/>with initialize call"]

    G --> I["5. Call upstream server<br/>POST server_url<br/>tools/call"]
    H --> I

    I --> J["6. Parse<br/>upstream response"]

    J --> K["7. Send metrics<br/>to Python<br/>POST /tools/{id}/metric"]

    K --> L["8. Build JSON-RPC<br/>response"]

    L --> M["Return response"]

    style B fill:#FFE4B5
    style I fill:#FFB6C1
    style K fill:#90EE90
    style L fill:#90EE90
```

---

## 5. Database Operations

### 5.1 PostgreSQL Connection

**Configuration:**
- Environment: `MCP_RUST_DATABASE_URL`
- Format: `postgresql://user:pass@host:port/dbname?sslmode=require`
- Connection pool: `deadpool-postgres` (or equivalent)
- Pool size: 20 connections (configurable)
- TLS: Supported via `sslmode` parameter

**Supported SSL modes:**
- `disable` - No TLS
- `prefer` - TLS if available
- `require` - TLS required (default for production)

**Note:** Client certificate authentication (`sslcert`, `sslkey`) is NOT supported yet.

### 5.2 Direct DB Queries

#### tools/list Query

```sql
-- Admin user (sees all tools)
SELECT
    id, name, description, input_schema, annotations,
    server_id, public, team_id, created_at, updated_at
FROM tools
WHERE server_id = $1
ORDER BY name;

-- Non-admin user (sees public + team tools)
SELECT
    id, name, description, input_schema, annotations,
    server_id, public, team_id, created_at, updated_at
FROM tools
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
-- $2 = ARRAY['team-a', 'team-b']
```

#### resources/list Query

```sql
-- Non-admin user
SELECT
    id, uri, name, description, mime_type,
    server_id, public, team_id, created_at, updated_at
FROM resources
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

#### resources/read Query

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

#### prompts/list Query

```sql
-- Non-admin user
SELECT
    id, name, description, arguments,
    server_id, public, team_id, created_at, updated_at
FROM prompts
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

#### prompts/get Query

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

#### resource_templates/list Query

```sql
SELECT
    id, uri_template, name, description, mime_type,
    server_id, public, team_id
FROM resource_templates
WHERE server_id = $1
  AND (public = true OR team_id = ANY($2))
ORDER BY name;
```

### 5.3 DB Operation Requirements

| ID | Requirement | Details |
|----|-------------|---------|
| DB-1 | Connection pooling | Use connection pool, max 20 connections |
| DB-2 | Parameterized queries | ALL queries MUST use parameters to prevent SQL injection |
| DB-3 | Team visibility filtering | Filter by `public = true OR team_id = ANY($teams)` |
| DB-4 | Admin bypass | Admin users skip team filtering |
| DB-5 | Server scope | All queries filtered by `server_id` |
| DB-6 | TLS support | Support PostgreSQL TLS via `sslmode` |
| DB-7 | Error handling | DB errors → 502 Bad Gateway with redacted message |
| DB-8 | Timeout | Query timeout: 30 seconds (configurable) |

---

## 6. Cache Operations

### 6.1 Cache Layers

```mermaid
flowchart TB
    subgraph L1["L1: In-Memory Cache<br/>HashMap + Mutex"]
        direction TB
        M1["runtime_sessions<br/>HashMap<String,<br/>RuntimeSessionRecord>"]
        M2["upstream_tool_sessions<br/>HashMap<String,<br/>UpstreamToolSession>"]
        M3["resolved_tool_call_plans<br/>HashMap<String,<br/>CachedResolvedToolCallPlan>"]
        M4["rmcp_upstream_clients<br/>HashMap<String,<br/>CachedRmcpUpstreamClient>"]
        M5["TTL-based expiry<br/>+ periodic sweeper"]
    end

    subgraph L2["L2: Redis Cache<br/>cross-worker sharing"]
        direction TB
        R1["Session records<br/>{prefix}:rust:mcp:session:{id}"]
        R2["Event store<br/>{prefix}:eventstore:{stream_id}<br/>Redis Hash"]
        R3["Affinity keys<br/>{prefix}:affinity:{session_id}"]
        R4["Pool owner<br/>{prefix}:pool_owner:{session_id}"]
        R5["SCAN-based cleanup<br/>not KEYS"]
    end

    L1 <-->|"Sync on read/write"| L2

    style M1 fill:#90EE90
    style M2 fill:#90EE90
    style M3 fill:#90EE90
    style M4 fill:#90EE90
    style R1 fill:#87CEEB
    style R2 fill:#87CEEB
    style R3 fill:#87CEEB
    style R4 fill:#87CEEB
```

### 6.2 Redis Key Patterns

| Key Pattern | Example | TTL | Description |
|-------------|---------|-----|-------------|
| `{prefix}:rust:mcp:session:{session_id}` | `mcpgw:rust:mcp:session:abc-123` | 3600s | Runtime session record |
| `{prefix}:eventstore:{stream_id}` | `mcpgw:eventstore:abc-123` | 3600s | Event store hash |
| `{prefix}:affinity:{session_id}` | `mcpgw:affinity:abc-123` | 3600s | Affinity owner worker |
| `{prefix}:pool_owner:{session_id}` | `mcpgw:pool_owner:abc-123` | 3600s | Session pool owner |
| `{prefix}:rust:tool-plan:{hash}` | `mcpgw:rust:tool-plan:sha256...` | 30s | Cached tool call plan |

### 6.3 Cache Operations

#### Session Cache Operations

```rust
// Get session (checks TTL)
async fn get_runtime_session(state: &AppState, session_id: &str) -> Option<RuntimeSessionRecord> {
    // 1. Check in-memory cache
    let sessions = state.runtime_sessions.lock().await;
    if let Some(record) = sessions.get(session_id) {
        if record.last_used.elapsed() < state.session_ttl() {
            return Some(record.clone());
        }
    }

    // 2. Check Redis
    let redis = state.redis().await?;
    let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
    let stored: Option<StoredRuntimeSessionRecord> = redis.get(&key).await.ok()?;

    // 3. Rebuild record with local timing
    let record = RuntimeSessionRecord::from(stored?);

    // 4. Update in-memory cache
    sessions.insert(session_id.to_string(), record.clone());

    Some(record)
}

// Upsert session (saves to both caches)
async fn upsert_runtime_session(state: &AppState, session_id: String, record: RuntimeSessionRecord) {
    // 1. Save to in-memory
    state.runtime_sessions.lock().await.insert(session_id.clone(), record.clone());

    // 2. Save to Redis
    if let Some(redis) = state.redis().await {
        let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
        let stored = StoredRuntimeSessionRecord::from(&record);
        let _: () = redis.set(&key, stored).await.ok();
        let _: () = redis.expire(&key, state.session_ttl().as_secs() as i64).await.ok();
    }
}

// Remove session
async fn remove_runtime_session(state: &AppState, session_id: &str) {
    // 1. Remove from in-memory
    state.runtime_sessions.lock().await.remove(session_id);

    // 2. Remove from Redis
    if let Some(redis) = state.redis().await {
        let key = format!("{}:rust:mcp:session:{}", state.cache_prefix(), session_id);
        let _: () = redis.del(&key).await.ok();
    }
}
```

#### Event Store Operations

```rust
// Store event (atomic Lua script)
async fn store_event_in_rust_event_store(
    state: &AppState,
    request: EventStoreStoreRequest,
) -> Result<String, Response> {
    let redis = state.redis().await
        .ok_or_else(|| json_response(501, json!({"detail": "Redis unavailable"})))?;

    let prefix = event_store_key_prefix(state, request.key_prefix.as_deref());
    let stream_key = format!("{}:{}", prefix, request.stream_id);

    // Lua script for atomic operation
    let script = r#"
        local stream_key = KEYS[1]
        local event_id = ARGV[1]
        local seq_num = tonumber(ARGV[2])
        local message = ARGV[3]
        local max_events = tonumber(ARGV[4])
        local ttl = tonumber(ARGV[5])

        -- Add event to hash
        redis.call('HSET', stream_key, event_id, message)

        -- Track sequence
        redis.call('HINCRBY', stream_key .. ':index', event_id, seq_num)

        -- Trim if exceeds max
        local count = redis.call('HLEN', stream_key)
        if count > max_events then
            -- Remove oldest events (simplified)
        end

        -- Set TTL
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
        .map_err(|e| json_response(500, json!({"detail": "Redis error"})))
}

// Replay events
async fn replay_events_from_rust_event_store(
    state: &AppState,
    request: EventStoreReplayRequest,
) -> Result<EventStoreReplayResponse, Response> {
    let redis = state.redis().await
        .ok_or_else(|| json_response(501, json!({"detail": "Redis unavailable"})))?;

    let prefix = event_store_key_prefix(state, request.key_prefix.as_deref());

    // Get stream_id and seq_num from last_event_id
    let (stream_id, last_seq) = get_stream_and_seq_from_event_id(&request.last_event_id)?;
    let stream_key = format!("{}:{}", prefix, stream_id);

    // Get all events with seq > last_seq
    let entries: HashMap<String, String> = redis.hgetall(&stream_key).await?;
    let index: HashMap<String, i64> = redis.hgetall(&format!("{}:index", stream_key)).await?;

    // Filter and sort events
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

### 6.4 Cache Requirements

| ID | Requirement | Details |
|----|-------------|---------|
| C-1 | In-memory cache | LRU-style with Mutex protection |
| C-2 | Redis cache | For cross-worker session sharing |
| C-3 | TTL-based expiry | All entries have TTL |
| C-4 | Periodic sweeper | Background task to clean expired entries |
| C-5 | Cache key hashing | SHA256 for complex keys |
| C-6 | Atomic Redis ops | Use Lua scripts for atomicity |
| C-7 | SCAN not KEYS | Use SCAN for cleanup (production-safe) |
| C-8 | Configurable prefix | Redis key prefix (default: `mcpgw:`) |

---

## 7. Metrics Collection

### 7.1 Runtime Stats Structure

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

### 7.2 Metrics Collection Points

#### Session Auth Reuse Metrics

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

#### Session Access Denial Metrics

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

#### Affinity Metrics

```rust
fn record_affinity_forward_attempt(&self) {
    self.affinity_forward_attempts.fetch_add(1, Ordering::Relaxed);
}

fn record_affinity_forwarded_request(&self) {
    self.affinity_forwarded_requests.fetch_add(1, Ordering::Relaxed);
}
```

### 7.3 Metrics Exposure

Metrics are exposed via:
1. **`GET /health` endpoint** - Full stats in JSON response
2. **Response headers** - Runtime identification headers
3. **Structured logs** - Per-request logging with method and mode

Example health response:
```json
{
  "status": "ok",
  "runtime": "rust",
  "active_sessions": 42,
  "runtime_stats": { ... }
}
```

### 7.4 Metrics Requirements

| ID | Requirement | Details |
|----|-------------|---------|
| M-1 | Atomic counters | Use atomic operations for thread safety |
| M-2 | All miss reasons | Track all session auth reuse miss reasons |
| M-3 | All denial reasons | Track all session access denial reasons |
| M-4 | Affinity tracking | Track forward attempts and successes |
| M-5 | Backend round trips | Count Python auth calls |
| M-6 | Health endpoint | Expose all stats in `/health` |
| M-7 | Runtime headers | Add `x-contextforge-mcp-*` headers to responses |
| M-8 | Structured logging | Log method and mode for each request |

---

## 8. Data Models

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

**Stored in Redis** (excludes local timing):
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

## 9. Configuration

### 9.1 CLI Arguments and Environment Variables

| CLI Argument | Environment Variable | Default | Description |
|--------------|---------------------|---------|-------------|
| `--backend-rpc-url` | `MCP_RUST_BACKEND_RPC_URL` | `http://127.0.0.1:4444/rpc` | Backend RPC endpoint |
| `--listen-http` | `MCP_RUST_LISTEN_HTTP` | `127.0.0.1:8787` | HTTP listen address |
| `--listen-uds` | `MCP_RUST_LISTEN_UDS` | - | Unix domain socket path |
| `--public-listen-http` | `MCP_RUST_PUBLIC_LISTEN_HTTP` | - | Public HTTP listen address |
| `--protocol-version` | `MCP_RUST_PROTOCOL_VERSION` | `2025-11-25` | Primary protocol version |
| `--supported-protocol-version` | `MCP_RUST_SUPPORTED_PROTOCOL_VERSIONS` | (defaults) | Comma-separated supported versions |
| `--server-name` | `MCP_RUST_SERVER_NAME` | `ContextForge` | Server name |
| `--server-version` | `MCP_RUST_SERVER_VERSION` | (from package) | Server version |
| `--instructions` | `MCP_RUST_INSTRUCTIONS` | (default text) | Server instructions |
| `--request-timeout-ms` | `MCP_RUST_REQUEST_TIMEOUT_MS` | `30000` | Request timeout in ms |
| `--client-connect-timeout-ms` | `MCP_RUST_CLIENT_CONNECT_TIMEOUT_MS` | `5000` | Client connect timeout |
| `--client-pool-idle-timeout-seconds` | `MCP_RUST_CLIENT_POOL_IDLE_TIMEOUT_SECONDS` | `90` | Pool idle timeout |
| `--client-pool-max-idle-per-host` | `MCP_RUST_CLIENT_POOL_MAX_IDLE_PER_HOST` | `1024` | Max idle connections per host |
| `--client-tcp-keepalive-seconds` | `MCP_RUST_CLIENT_TCP_KEEPALIVE_SECONDS` | `30` | TCP keepalive interval |
| `--tools-call-plan-ttl-seconds` | `MCP_RUST_TOOLS_CALL_PLAN_TTL_SECONDS` | `30` | Tool call plan cache TTL |
| `--upstream-session-ttl-seconds` | `MCP_RUST_UPSTREAM_SESSION_TTL_SECONDS` | `300` | Upstream session TTL |
| `--use-rmcp-upstream-client` | `MCP_RUST_USE_RMCP_UPSTREAM_CLIENT` | `false` | Use RMCP library client |
| `--session-core-enabled` | `MCP_RUST_SESSION_CORE_ENABLED` | `false` | Enable session core |
| `--event-store-enabled` | `MCP_RUST_EVENT_STORE_ENABLED` | `false` | Enable event store |
| `--resume-core-enabled` | `MCP_RUST_RESUME_CORE_ENABLED` | `false` | Enable resume core |
| `--live-stream-core-enabled` | `MCP_RUST_LIVE_STREAM_CORE_ENABLED` | `false` | Enable live stream core |
| `--affinity-core-enabled` | `MCP_RUST_AFFINITY_CORE_ENABLED` | `false` | Enable affinity core |
| `--session-auth-reuse-enabled` | `MCP_RUST_SESSION_AUTH_REUSE_ENABLED` | `false` | Enable session auth reuse |
| `--session-auth-reuse-ttl-seconds` | `MCP_RUST_SESSION_AUTH_REUSE_TTL_SECONDS` | `30` | Session auth reuse TTL |
| `--session-ttl-seconds` | `MCP_RUST_SESSION_TTL_SECONDS` | `3600` | Session TTL |
| `--event-store-max-events-per-stream` | `MCP_RUST_EVENT_STORE_MAX_EVENTS_PER_STREAM` | `100` | Max events per stream |
| `--event-store-ttl-seconds` | `MCP_RUST_EVENT_STORE_TTL_SECONDS` | `3600` | Event store TTL |
| `--event-store-poll-interval-ms` | `MCP_RUST_EVENT_STORE_POLL_INTERVAL_MS` | `250` | Event store poll interval |
| `--cache-prefix` | `MCP_RUST_CACHE_PREFIX` | `mcpgw:` | Redis cache key prefix |
| `--database-url` | `MCP_RUST_DATABASE_URL` | - | PostgreSQL connection URL |
| `--redis-url` | `MCP_RUST_REDIS_URL` | - | Redis connection URL |
| `--db-pool-max-size` | `MCP_RUST_DB_POOL_MAX_SIZE` | `20` | DB pool max size |
| `--log-filter` | `MCP_RUST_LOG` | `info` | Log level filter |

### 9.2 Mode Presets

| Mode | Public `/mcp` | Session Core | Event Store | Resume | Live Stream | Affinity | Auth Reuse |
|------|---------------|--------------|-------------|--------|-------------|----------|------------|
| `off` | Python | No | No | No | No | No | No |
| `shadow` | Python | No | No | No | No | No | No |
| `edge` | Rust | No | No | No | No | No | Yes |
| `full` | Rust | Yes | Yes | Yes | Yes | Yes | Yes |

---

## 10. Error Handling

### 10.1 JSON-RPC Error Codes

| Code | Message | When |
|------|---------|------|
| `-32600` | Invalid Request | Invalid JSON-RPC format |
| `-32601` | Method Not Found | Unknown MCP method |
| `-32602` | Invalid Params | Missing/invalid parameters |
| `-32603` | Internal Error | Rust internal error |
| `-32000` | Server error | Backend/Rust error (generic) |
| `-32003` | Access denied | Session/auth denial |

### 10.2 Error Response Format

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

### 10.3 Error Redaction

**Client-visible errors:**
- Internal error details are REDACTED
- Return generic message: `"See server logs"` or `"CLIENT_ERROR_DETAIL"`

**Server-side logs:**
- Full error details logged
- Include stack traces for debugging

### 10.4 HTTP Status Mapping

| JSON-RPC Error | HTTP Status |
|----------------|-------------|
| `-32600` | 400 Bad Request |
| `-32601` | 404 Not Found |
| `-32602` | 400 Bad Request |
| `-32603` | 500 Internal Server Error |
| `-32000` | 502 Bad Gateway |
| `-32003` | 403 Forbidden |
| Session not found | 404 Not Found |
| Auth failure | 401 Unauthorized |

### 10.5 Error Handling Requirements

| ID | Requirement | Details |
|----|-------------|---------|
| E-1 | JSON-RPC format | All errors in JSON-RPC 2.0 format |
| E-2 | Error codes | Use standard JSON-RPC error codes |
| E-3 | Client redaction | Never expose internal errors to clients |
| E-4 | Server logging | Log full error details server-side |
| E-5 | HTTP mapping | Map JSON-RPC errors to HTTP status codes |
| E-6 | Timeout handling | Handle timeouts with appropriate error |
| E-7 | Connection errors | Handle connection failures gracefully |
| E-8 | Fallback behavior | Fall back to Python on Rust errors when safe |

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-17 | ContextForge Team | Initial requirements |

---

## 12. References

- [Rust MCP Runtime README](../../tools_rust/mcp_runtime/README.md)
- [Rust MCP Runtime DEVELOPING](../../tools_rust/mcp_runtime/DEVELOPING.md)
- [Rust MCP Runtime TESTING-DESIGN](../../tools_rust/mcp_runtime/TESTING-DESIGN.md)
- [Rust MCP Runtime STATUS](../../tools_rust/mcp_runtime/STATUS.md)
- [Architecture: Rust MCP Runtime](../../docs/docs/architecture/rust-mcp-runtime.md)
- [ADR-043: Rust MCP Runtime Sidecar](../../docs/docs/architecture/adr/043-rust-mcp-runtime-sidecar-mode-model.md)
