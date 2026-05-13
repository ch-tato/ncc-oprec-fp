# Custom SIEM Application — Software Architecture & Implementation Document

**Project Type:** NCC Open Recuitment Final Project 2026 — Full Stack SIEM  
**Team Members:** Muhammad Quthbi Danish Abqori (@ch-tato), Ananda Aryasatya Zhafran Aqila ()[], Raditya Zhafran Pranuja
**Timeline:** 11-23 May 2026
**Document Version:** 1.0
---

## Table of Contents

1. [Full System Architecture](#1-full-system-architecture)
2. [Complete Folder Structure](#2-complete-folder-structure)
3. [Database Design](#3-database-design)
4. [Backend Architecture](#4-backend-architecture)
5. [Log Collection Design](#5-log-collection-design)
6. [YAML Rule Engine Design](#6-yaml-rule-engine-design)
7. [Alerting System](#7-alerting-system)
8. [Frontend Architecture](#8-frontend-architecture)
9. [UI/UX Design Plan](#9-uiux-design-plan)
10. [REST API Design](#10-rest-api-design)
11. [WebSocket Design](#11-websocket-design)
12. [DevOps Architecture](#12-devops-architecture)
13. [Jenkins CI/CD Pipeline](#13-jenkins-cicd-pipeline)
14. [SonarQube Integration](#14-sonarqube-integration)
15. [Security Best Practices](#15-security-best-practices)
16. [Implementation Roadmap](#16-implementation-roadmap)
17. [Risk Management](#17-risk-management)
18. [Testing Strategy](#18-testing-strategy)
19. [Deployment Strategy](#19-deployment-strategy)
20. [Performance Considerations](#20-performance-considerations)
21. [Final Recommendations](#21-final-recommendations)

---

## 1. Full System Architecture

### 1.1 Architecture Overview

The system follows a **monolithic-with-service-decomposition** pattern. Rather than full microservices (which would be overengineering for 11 days), the backend is a single deployable binary internally decomposed into discrete service packages. This provides clean separation of concerns without operational overhead.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT BROWSER                                │
│                    React SPA (Vite + Tailwind + Recharts)                  │
└───────────────────────────────┬────────────────────────────────────────────┘
                                │ HTTPS / WSS
                                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          NGINX REVERSE PROXY                               │
│              /api/* → backend:8080   /ws → backend:8080/ws                 │
│              /* → React SPA (static files)                                 │
└───────────────────────────────┬────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      GO BACKEND (Gin + Gorilla WS)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐    │
│  │   HTTP API   │  │  WebSocket   │  │  Log Watcher │  │  Rule Engine  │    │
│  │  (Gin Router)│  │   Hub        │  │  (fsnotify)  │  │  (YAML-based) │    │
│  └──────┬───────┘  └──────┬───────┘  └───────┬──────┘  └────────┬──────┘    │
│         │                 │                  │                  │           │
│  ┌──────▼─────────────────▼──────────────────▼──────────────────▼────────┐  │
│  │                        SERVICE LAYER                                  │  │
│  │  LogSourceService │ EventService │ AlertService │ RuleService         │  │
│  └──────────────────────────────────┬────────────────────────────────────┘  │
│                                     │                                       │
│  ┌──────────────────────────────────▼─────────────────────────────────────┐ │
│  │                      REPOSITORY LAYER                                  │ │
│  │         PostgreSQL via pgx driver (connection pool)                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
          ┌─────────────────────┼────────────────────┐
          ▼                     ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  PostgreSQL      │  │  Host Filesystem │  │  Discord Webhook │
│  (events,alerts, │  │  (log files via  │  │  (outbound HTTP  │
│   rules, sources)│  │   bind mount)    │  │   notifications) │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 1.2 Data Flow

**Ingest Flow (Log → Alert):**

```
Log File on Disk
      │
      ▼ (fsnotify detects write event)
File Reader (reads new bytes via offset tracking)
      │
      ▼
Log Parser (regex-based, extracts: timestamp, level, message, host, PID)
      │
      ▼
Event struct
      │
      ├──► PostgreSQL (persist parsed event)
      │
      ▼
Rule Engine (evaluate event against loaded YAML rules)
      │
      ▼ (if rule matches)
Alert Generator
      │
      ├──► PostgreSQL (persist alert)
      ├──► WebSocket Hub (broadcast to all connected clients)
      └──► Discord Webhook (HTTP POST notification)
```

**Query Flow (Frontend → Data):**

```
React SPA
    │
    ├── REST GET /api/v1/events → Gin Handler → EventService → PostgreSQL
    ├── REST GET /api/v1/alerts → Gin Handler → AlertService → PostgreSQL
    └── WS  /ws → WebSocket Hub → real-time alert pushes
```

### 1.3 Service Responsibilities

| Component | Responsibility |
|---|---|
| **Nginx** | TLS termination, reverse proxy, static file serving, path routing |
| **Gin HTTP Server** | REST API routing, middleware chain, request validation |
| **WebSocket Hub** | Client registry, alert broadcasting, connection lifecycle |
| **Log Watcher** | File watching, offset management, new-path registration |
| **Log Parser** | Regex parsing, struct extraction, error handling |
| **Rule Engine** | YAML rule loading, event evaluation, alert generation |
| **PostgreSQL** | All persistent state: events, alerts, log sources, rules |
| **Discord Webhook** | External alert delivery |

### 1.4 Why This Architecture Is Optimal

For a 3-person team with 11 days, this architecture strikes the correct balance because:

- **Single deployable binary** — No inter-service network calls, no service mesh, no distributed tracing needed. Everything is in-process, making debugging straightforward.
- **Internal decomposition** — Clean package boundaries enforce separation of concerns without the operational burden of microservices.
- **PostgreSQL as single source of truth** — Eliminates the need for Redis, Kafka, or any additional message broker. The database handles persistence and acts as the durable event store.
- **fsnotify + goroutines** — Native Go concurrency handles concurrent file watching without external dependencies.
- **WebSocket for real-time** — One persistent connection per client replaces polling, giving genuine real-time feel with minimal complexity.

### 1.5 Tradeoff Analysis

| Decision | Advantage | Tradeoff |
|---|---|---|
| Single binary backend | Simple deployment, easy debugging | Can't scale individual components independently |
| PostgreSQL only (no Redis) | Fewer moving parts, less ops overhead | Higher DB load if events are very high frequency |
| In-process rule engine | No IPC latency, simple state | Rules reload requires app restart (acceptable for demo) |
| No auth/JWT | Saves ~1-2 days of dev time | Not production-ready; acceptable for internal demo |
| Docker Compose (not K8s) | Works on a single VPS, easy to operate | Not horizontally scalable |

### 1.6 Scalability Discussion

This architecture is intentionally scoped for a single-node deployment. If it needed to scale:

- The Log Watcher goroutines would be extracted into a separate collector service.
- PostgreSQL would gain read replicas.
- The WebSocket Hub would be backed by Redis Pub/Sub to support multi-instance deployments.
- The Rule Engine would be externalized as a sidecar or Lambda function.

None of these are needed for the university demo. The current design handles hundreds of log lines per second and dozens of concurrent WebSocket clients comfortably on a modest VPS.

---

## 2. Complete Folder Structure

### 2.1 Backend Structure

```
backend/
├── cmd/
│   └── server/
│       └── main.go                # Entrypoint: wires all dependencies, starts server
├── internal/
│   ├── api/
│   │   ├── handler/
│   │   │   ├── alert_handler.go   # HTTP handlers for /alerts endpoints
│   │   │   ├── event_handler.go   # HTTP handlers for /events endpoints
│   │   │   ├── logsource_handler.go
│   │   │   ├── rule_handler.go
│   │   │   └── stats_handler.go   # Dashboard statistics
│   │   ├── middleware/
│   │   │   ├── cors.go            # CORS headers
│   │   │   ├── logger.go          # Request logging middleware
│   │   │   └── recovery.go        # Panic recovery
│   │   └── router.go              # Gin router setup, all routes defined here
│   ├── config/
│   │   └── config.go              # Reads env vars, provides Config struct
│   ├── domain/
│   │   ├── alert.go               # Alert struct, severity constants
│   │   ├── event.go               # ParsedEvent struct
│   │   ├── logsource.go           # LogSource struct
│   │   └── rule.go                # Rule struct (mirrors YAML schema)
│   ├── parser/
│   │   ├── parser.go              # Parser interface
│   │   ├── syslog_parser.go       # Syslog format parser
│   │   ├── nginx_parser.go        # Nginx access/error log parser
│   │   ├── generic_parser.go      # Fallback regex parser
│   │   └── factory.go             # Returns correct parser by log type
│   ├── repository/
│   │   ├── alert_repo.go          # Alert DB queries
│   │   ├── event_repo.go          # Event DB queries
│   │   ├── logsource_repo.go      # LogSource DB queries
│   │   └── rule_repo.go           # Rule DB queries
│   ├── ruleengine/
│   │   ├── engine.go              # Core rule evaluator
│   │   ├── loader.go              # Loads/validates YAML rules
│   │   └── matcher.go             # Condition matching logic
│   ├── service/
│   │   ├── alert_service.go       # Alert business logic
│   │   ├── event_service.go       # Event business logic
│   │   ├── logsource_service.go   # LogSource management
│   │   └── rule_service.go        # Rule CRUD + reload
│   ├── watcher/
│   │   ├── watcher.go             # fsnotify wrapper, goroutine per file
│   │   ├── reader.go              # Offset-aware file reader
│   │   └── registry.go            # Tracks active watchers
│   └── websocket/
│       ├── hub.go                 # Client registry + broadcast
│       └── client.go              # Individual WS connection handler
├── migrations/
│   ├── 001_create_tables.sql
│   └── 002_add_indexes.sql
├── rules/
│   ├── ssh_brute_force.yaml
│   ├── failed_login.yaml
│   └── high_error_rate.yaml
├── Dockerfile
├── .env.example
└── go.mod
```

**Folder Rationale:**

- `cmd/` — Standard Go project layout. Entrypoint only; no business logic here.
- `internal/` — The `internal` package prevents external packages from importing these. Enforces clean boundaries.
- `domain/` — Pure data structures. No methods, no imports of other internal packages. Dependencies point inward to domain, never outward.
- `repository/` — All SQL is here. Nothing else touches the database. Swap PostgreSQL for SQLite in tests trivially.
- `service/` — Business logic. Calls repositories. No HTTP concerns. Independently testable.
- `api/handler/` — HTTP only. Calls services. Handles request parsing, validation, response serialization.
- `migrations/` — Versioned SQL migrations run at startup or by a migration tool.

### 2.2 Frontend Structure

```
frontend/
├── public/
│   └── favicon.ico
├── src/
│   ├── api/
│   │   ├── client.ts              # Axios instance with base URL + interceptors
│   │   ├── alerts.ts              # Alert API calls
│   │   ├── events.ts              # Event API calls
│   │   ├── logSources.ts          # LogSource API calls
│   │   ├── rules.ts               # Rule API calls
│   │   └── stats.ts               # Dashboard stats API calls
│   ├── components/
│   │   ├── common/
│   │   │   ├── Badge.tsx          # Severity badge (color-coded)
│   │   │   ├── Card.tsx           # Dashboard card wrapper
│   │   │   ├── DataTable.tsx      # Reusable paginated table
│   │   │   ├── EmptyState.tsx     # Empty data placeholder
│   │   │   ├── LoadingSpinner.tsx
│   │   │   └── StatusDot.tsx      # Live connection indicator
│   │   ├── alerts/
│   │   │   ├── AlertTable.tsx
│   │   │   ├── AlertRow.tsx
│   │   │   └── AlertFilters.tsx
│   │   ├── charts/
│   │   │   ├── EventsTimelineChart.tsx
│   │   │   ├── SeverityDonutChart.tsx
│   │   │   └── TopSourcesBarChart.tsx
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── TopBar.tsx
│   │   │   └── Layout.tsx         # Wraps Sidebar + TopBar + children
│   │   ├── logSources/
│   │   │   ├── LogSourceCard.tsx
│   │   │   ├── LogSourceForm.tsx
│   │   │   └── LogSourceList.tsx
│   │   └── rules/
│   │       ├── RuleCard.tsx
│   │       ├── RuleEditor.tsx     # YAML textarea editor
│   │       └── RuleList.tsx
│   ├── hooks/
│   │   ├── useAlerts.ts           # SWR/React Query for alerts
│   │   ├── useWebSocket.ts        # WS connection + reconnection logic
│   │   ├── useStats.ts
│   │   └── useLogSources.ts
│   ├── pages/
│   │   ├── Dashboard.tsx          # Main overview page
│   │   ├── Alerts.tsx             # Full alert table
│   │   ├── Events.tsx             # Raw event browser
│   │   ├── LogSources.tsx         # Log source management
│   │   └── Rules.tsx              # Rule management
│   ├── store/
│   │   └── alertStore.ts          # Zustand store for live alerts
│   ├── types/
│   │   ├── alert.ts
│   │   ├── event.ts
│   │   ├── logsource.ts
│   │   └── rule.ts
│   ├── utils/
│   │   ├── severity.ts            # Severity → color/label mapping
│   │   ├── datetime.ts            # Date formatting helpers
│   │   └── ws.ts                  # WebSocket URL builder
│   ├── App.tsx
│   ├── main.tsx
│   └── router.tsx                 # React Router v6 routes
├── index.html
├── tailwind.config.js
├── vite.config.ts
└── package.json
```

### 2.3 Infrastructure Structure

```
infra/
├── docker-compose.yml             # Main compose: backend, frontend, postgres, nginx
├── docker-compose.dev.yml         # Dev overrides: hot reload, exposed ports
├── nginx/
│   ├── nginx.conf                 # Main nginx config
│   └── conf.d/
│       └── siem.conf              # Server block: proxy + static
├── jenkins/
│   └── Jenkinsfile                # Pipeline definition
├── sonarqube/
│   ├── sonar-project.properties   # Backend SonarQube config
│   └── sonar-project-frontend.properties
├── postgres/
│   └── init.sql                   # Initial schema (used by Docker entrypoint)
└── scripts/
    ├── deploy.sh                  # Production deploy script
    ├── backup-db.sh               # PostgreSQL backup
    └── health-check.sh            # Container health check
```

---

## 3. Database Design

### 3.1 Full PostgreSQL Schema

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "pgcrypto";  -- for gen_random_uuid()

-- ============================================================
-- ENUM TYPES
-- ============================================================
CREATE TYPE severity_level AS ENUM ('info', 'low', 'medium', 'high', 'critical');
CREATE TYPE log_source_status AS ENUM ('active', 'inactive', 'error');

-- ============================================================
-- TABLE: log_sources
-- Tracks registered log file paths
-- ============================================================
CREATE TABLE log_sources (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    file_path   TEXT NOT NULL UNIQUE,
    log_type    VARCHAR(64) NOT NULL DEFAULT 'generic',  -- 'syslog', 'nginx', 'generic'
    status      log_source_status NOT NULL DEFAULT 'active',
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE: events
-- All parsed log entries. High volume table.
-- ============================================================
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    log_source_id   UUID NOT NULL REFERENCES log_sources(id) ON DELETE CASCADE,
    raw_line        TEXT NOT NULL,
    message         TEXT,
    hostname        VARCHAR(255),
    process         VARCHAR(128),
    pid             INTEGER,
    log_level       VARCHAR(32),
    event_time      TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    extra           JSONB                       -- arbitrary parsed key/value pairs
);

-- ============================================================
-- TABLE: rules
-- YAML-defined detection rules (stored as YAML text + metadata)
-- ============================================================
CREATE TABLE rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    yaml_content    TEXT NOT NULL,              -- raw YAML for editing in UI
    severity        severity_level NOT NULL DEFAULT 'medium',
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE: alerts
-- Alerts generated by rule engine matches
-- ============================================================
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID REFERENCES rules(id) ON DELETE SET NULL,
    rule_name       VARCHAR(255) NOT NULL,      -- denormalized: survives rule deletion
    event_id        BIGINT REFERENCES events(id) ON DELETE SET NULL,
    log_source_id   UUID REFERENCES log_sources(id) ON DELETE SET NULL,
    severity        severity_level NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    raw_line        TEXT,
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_at TIMESTAMPTZ,
    discord_sent    BOOLEAN NOT NULL DEFAULT false
);

-- ============================================================
-- INDEXES
-- ============================================================

-- Events: most queries filter by time range and source
CREATE INDEX idx_events_event_time       ON events (event_time DESC);
CREATE INDEX idx_events_log_source_id    ON events (log_source_id);
CREATE INDEX idx_events_received_at      ON events (received_at DESC);
CREATE INDEX idx_events_log_level        ON events (log_level);

-- Partial index for fast alert dashboard queries (unacknowledged)
CREATE INDEX idx_alerts_triggered_at     ON alerts (triggered_at DESC);
CREATE INDEX idx_alerts_severity         ON alerts (severity);
CREATE INDEX idx_alerts_unack            ON alerts (acknowledged) WHERE acknowledged = false;
CREATE INDEX idx_alerts_log_source_id    ON alerts (log_source_id);

-- JSONB index for extra field queries (optional, add if needed)
CREATE INDEX idx_events_extra_gin        ON events USING GIN (extra);

-- ============================================================
-- UPDATED_AT TRIGGER
-- ============================================================
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_sources_updated_at
    BEFORE UPDATE ON log_sources
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_rules_updated_at
    BEFORE UPDATE ON rules
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### 3.2 Table Explanations

**`log_sources`** — Stores paths and metadata for each registered log file. The `log_type` field drives parser selection. `status` lets the UI show which watchers are healthy.

**`events`** — The high-volume core table. Every parsed log line becomes a row. The `extra JSONB` column captures parser-specific fields without requiring schema migrations. The `event_time` comes from the log itself (parsed timestamp), while `received_at` is the ingestion timestamp — both are preserved for latency analysis.

**`rules`** — Stores the YAML definition alongside extracted metadata. Storing `yaml_content` as text means rules are editable from the UI and re-parseable at runtime. The `enabled` flag allows toggling without deletion.

**`alerts`** — Denormalized `rule_name` ensures historical alert records survive rule deletion. The `discord_sent` flag enables retry logic for failed webhook deliveries.

### 3.3 Query Optimization Strategy

- All time-range queries use `event_time DESC` index — this is the primary access pattern for the event browser.
- Dashboard stats use `COUNT` with `WHERE` clauses covered by the severity and acknowledgment indexes.
- The partial index on `acknowledged = false` makes the "active alerts" widget extremely fast regardless of total alert volume.
- Pagination uses `LIMIT` + `OFFSET` for simplicity; cursor-based pagination is not needed for demo scale.

### 3.4 Data Retention Strategy

For the demo project, no automatic retention policy is needed. If the system runs continuously during testing, add a manual cleanup query:

```sql
-- Delete events older than 7 days (run manually or as a scheduled job)
DELETE FROM events WHERE received_at < NOW() - INTERVAL '7 days';
```

In production, this would be a PostgreSQL cron job (`pg_cron`) or a background goroutine with a ticker.

---

## 4. Backend Architecture

### 4.1 Layered Architecture

```
┌─────────────────────────────────┐
│       HTTP / WebSocket Layer    │  ← Gin handlers, WS hub (no business logic)
├─────────────────────────────────┤
│         Service Layer           │  ← Business logic, orchestration
├─────────────────────────────────┤
│        Repository Layer         │  ← All SQL queries, zero business logic
├─────────────────────────────────┤
│          Domain Layer           │  ← Pure structs, constants, interfaces
└─────────────────────────────────┘
```

Dependency direction: **Handler → Service → Repository → Domain**. No layer imports from a layer above it.

### 4.2 Interface Usage

Interfaces are defined at the point of consumption (services consume repository interfaces, handlers consume service interfaces):

```go
// internal/service/alert_service.go
type AlertRepository interface {
    Create(ctx context.Context, alert *domain.Alert) error
    List(ctx context.Context, filter AlertFilter) ([]domain.Alert, int64, error)
    Acknowledge(ctx context.Context, id uuid.UUID) error
}

type AlertService struct {
    repo    AlertRepository
    wsHub   WebSocketHub
    discord DiscordNotifier
}
```

This enables unit testing with mock implementations without requiring a live database.

### 4.3 Configuration Management

All configuration is loaded from environment variables at startup. A `Config` struct is populated once and passed via dependency injection:

```go
// internal/config/config.go
type Config struct {
    DBHost         string
    DBPort         int
    DBName         string
    DBUser         string
    DBPassword     string
    ServerPort     int
    DiscordWebhook string
    RulesDir       string
    LogLevel       string
}

func Load() (*Config, error) {
    // Uses os.Getenv with defaults
    // Returns error if required fields are missing
}
```

Never use global variables for config. Pass the `Config` struct down through constructors.

### 4.4 Middleware Design

```go
// Applied globally
router.Use(middleware.Recovery())   // Prevents panics from crashing the server
router.Use(middleware.Logger())     // Structured request logging
router.Use(middleware.CORS())       // Allows frontend origin

// Applied per-group
api := router.Group("/api/v1")
api.Use(middleware.RequestID())     // Adds X-Request-ID header
```

### 4.5 Authentication Strategy

**Decision: No authentication for the demo.**

Reasoning: Building JWT auth, token refresh, and secure storage would consume 1.5–2 days of development time. Since this is a university demo on an internal network, the risk is acceptable. The architecture is designed so auth middleware can be inserted into the Gin group without changing any handler code.

If needed in the last days, a simple API key header check (single hardcoded key from env var) can be added as middleware in under 30 minutes.

### 4.6 Concurrency Model

```
main goroutine
├── Gin HTTP server (goroutine per request, managed by Gin)
├── WebSocket Hub (single goroutine, manages all clients via select loop)
├── Log Watcher Registry (goroutine per watched file)
│   ├── Watcher goroutine: file A
│   ├── Watcher goroutine: file B
│   └── Watcher goroutine: file C
└── Alert Dispatcher (goroutine, reads from alertChan, fans out to WS + Discord)
```

**Channel design:**

```go
// Channels connecting components
parsedEventChan chan *domain.ParsedEvent  // Watcher → Rule Engine (buffered: 1000)
alertChan       chan *domain.Alert        // Rule Engine → Dispatcher (buffered: 100)
wsBroadcastChan chan []byte               // Dispatcher → WS Hub (buffered: 100)
```

Buffered channels prevent goroutine blocking under burst load. Buffer sizes are tuned for demo workloads.

### 4.7 Error Handling Strategy

- **Handler level:** Always return structured JSON error responses. Never let errors surface as HTML.
- **Service level:** Wrap errors with context using `fmt.Errorf("alertService.Create: %w", err)`.
- **Repository level:** Map `pgx` not-found errors to sentinel errors (`ErrNotFound`) that services can check.
- **Watcher level:** Log errors and continue. A parse error on one line should never stop the watcher.
- **Panic recovery:** The recovery middleware logs the panic and returns a 500. The server keeps running.

```go
// Standard error response
type ErrorResponse struct {
    Error   string `json:"error"`
    Code    string `json:"code"`
    Details string `json:"details,omitempty"`
}
```

### 4.8 Logging Strategy

Use Go's `slog` package (standard library, Go 1.21+) with JSON output in production:

```go
slog.Info("alert triggered",
    "rule_name", rule.Name,
    "severity", alert.Severity,
    "source", event.LogSourceID,
)
```

Log levels: `DEBUG` (dev), `INFO` (prod). All logs go to stdout and are captured by Docker.

---

## 5. Log Collection Design

### 5.1 How File Watching Works

`fsnotify` is a cross-platform file system notification library. For each registered log source, a goroutine is started that watches the file for `WRITE` events:

```go
func (w *FileWatcher) Start(ctx context.Context) error {
    watcher, _ := fsnotify.NewWatcher()
    watcher.Add(w.FilePath)

    for {
        select {
        case event := <-watcher.Events:
            if event.Op&fsnotify.Write == fsnotify.Write {
                w.readNewLines()
            }
        case err := <-watcher.Errors:
            slog.Error("watcher error", "path", w.FilePath, "err", err)
        case <-ctx.Done():
            return nil
        }
    }
}
```

### 5.2 Offset-Aware Reading

Each watcher maintains a file offset to avoid re-reading previously processed lines:

```go
type FileWatcher struct {
    FilePath  string
    offset    int64
    parser    parser.Parser
    eventChan chan<- *domain.ParsedEvent
}

func (w *FileWatcher) readNewLines() {
    f, _ := os.Open(w.FilePath)
    defer f.Close()
    f.Seek(w.offset, io.SeekStart)

    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
        line := scanner.Text()
        event, err := w.parser.Parse(line)
        if err != nil {
            slog.Debug("parse error", "line", line, "err", err)
            continue
        }
        w.eventChan <- event
    }
    w.offset, _ = f.Seek(0, io.SeekCurrent)
}
```

### 5.3 Dynamic Path Registration

When a user adds a new log source via the API, the `LogSourceService` saves it to the database and notifies the `WatcherRegistry` to start a new goroutine:

```go
// service/logsource_service.go
func (s *LogSourceService) Create(ctx context.Context, ls *domain.LogSource) error {
    if err := s.repo.Create(ctx, ls); err != nil {
        return err
    }
    // Signal the registry to watch this new path
    s.watcherRegistry.AddWatcher(ls)
    return nil
}
```

No restart required. The registry maintains a `sync.Map` of active watchers, preventing duplicate registration.

### 5.4 Log Rotation Handling

When a log file is rotated (renamed and recreated), `fsnotify` may stop receiving events for the original inode. The solution:

- Detect a `RENAME` or `REMOVE` event on the watched path.
- Wait briefly (100ms), then reopen the file at offset 0 (the new file starts from the beginning).
- Re-add the path to the watcher.

```go
case event.Op&fsnotify.Rename == fsnotify.Rename:
    time.Sleep(100 * time.Millisecond)
    w.offset = 0
    watcher.Add(w.FilePath)  // Re-watch the new file at the same path
```

### 5.5 Handling Malformed Lines

Parsing errors are logged at `DEBUG` level and skipped. The watcher continues processing subsequent lines. Malformed lines never crash the pipeline. The raw line is preserved in `events.raw_line` even if parsing is partial.

### 5.6 Parser Architecture

```go
// internal/parser/parser.go
type Parser interface {
    Parse(line string) (*domain.ParsedEvent, error)
    Name() string
}
```

Parsers implement this interface. The factory selects based on `log_type`:

```go
// internal/parser/factory.go
func New(logType string) Parser {
    switch logType {
    case "syslog":
        return &SyslogParser{}
    case "nginx":
        return &NginxParser{}
    default:
        return &GenericParser{}
    }
}
```

**Regex Examples:**

```go
// Syslog: "May  5 12:34:56 hostname sshd[1234]: Failed password for root"
var syslogRe = regexp.MustCompile(
    `^(?P<month>\w+)\s+(?P<day>\d+)\s+(?P<time>[\d:]+)\s+(?P<host>\S+)\s+(?P<proc>\S+?)(?:\[(?P<pid>\d+)\])?: (?P<msg>.+)$`,
)

// Nginx access: '127.0.0.1 - - [05/May/2025:12:34:56 +0000] "GET / HTTP/1.1" 200 612'
var nginxAccessRe = regexp.MustCompile(
    `^(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<bytes>\d+)`,
)
```

Regexes are compiled once at package init (`var re = regexp.MustCompile(...)`) — never inside the parse loop.

---

## 6. YAML Rule Engine Design

### 6.1 Rule YAML Structure

```yaml
# rules/ssh_brute_force.yaml
id: ssh-brute-force-001
name: "SSH Brute Force Detected"
description: "Detects repeated SSH authentication failures from the same source"
enabled: true
severity: high

# Which log sources this rule applies to (by log_type)
log_types:
  - syslog

# ALL conditions must match (AND logic)
conditions:
  - field: message
    operator: contains
    value: "Failed password"
  - field: process
    operator: equals
    value: "sshd"

# Optional: threshold within a time window
threshold:
  count: 5          # 5 or more matches
  window_seconds: 60  # within 60 seconds

alert:
  title: "SSH Brute Force: {{hostname}}"
  description: "{{count}} failed SSH logins in {{window_seconds}}s from host {{hostname}}"
```

```yaml
# rules/high_error_rate.yaml
id: nginx-high-error-001
name: "High HTTP Error Rate"
severity: medium
enabled: true
log_types:
  - nginx

conditions:
  - field: status
    operator: starts_with
    value: "5"   # 5xx errors

threshold:
  count: 10
  window_seconds: 30

alert:
  title: "High 5xx Error Rate on Nginx"
  description: "{{count}} HTTP 5xx errors in {{window_seconds}}s"
```

```yaml
# rules/failed_sudo.yaml
id: sudo-failure-001
name: "Sudo Authentication Failure"
severity: medium
enabled: true
log_types:
  - syslog

conditions:
  - field: message
    operator: contains
    value: "authentication failure"
  - field: process
    operator: contains
    value: "sudo"

# No threshold: match fires immediately on every occurrence
alert:
  title: "Sudo Failure on {{hostname}}"
  description: "User attempted sudo and failed: {{message}}"
```

### 6.2 Supported Operators

| Operator | Description |
|---|---|
| `equals` | Exact string match |
| `contains` | Substring match |
| `starts_with` | Prefix match |
| `ends_with` | Suffix match |
| `regex` | Full regex match |
| `greater_than` | Numeric comparison |
| `less_than` | Numeric comparison |

### 6.3 Rule Loading Mechanism

Rules are loaded from two sources:

1. **YAML files in `rules/` directory** — Loaded at startup via `filepath.Walk`.
2. **Database `rules` table** — Rules created/edited via UI are persisted here.

At startup, file-based rules are seeded into the database if they don't exist. The engine works exclusively from the in-memory loaded ruleset populated from the database.

```go
// internal/ruleengine/loader.go
type RuleLoader struct {
    rules     []*domain.Rule
    mu        sync.RWMutex  // Protects concurrent access during hot reload
}

func (l *RuleLoader) Load(rules []*domain.Rule) {
    l.mu.Lock()
    defer l.mu.Unlock()
    l.rules = rules
}

func (l *RuleLoader) GetRules() []*domain.Rule {
    l.mu.RLock()
    defer l.mu.RUnlock()
    return l.rules
}
```

When a rule is updated via the API, the service calls `loader.Load(allRules)` — an atomic in-memory swap. No restart needed.

### 6.4 Threshold/Window Logic

The threshold engine uses an in-memory sliding window per (rule_id, group_key):

```go
// internal/ruleengine/engine.go
type windowKey struct {
    RuleID   string
    GroupKey string  // e.g., hostname value for per-host counting
}

type windowCounter struct {
    timestamps []time.Time
    mu         sync.Mutex
}

func (e *Engine) checkThreshold(rule *domain.Rule, event *domain.ParsedEvent) bool {
    key := windowKey{RuleID: rule.ID, GroupKey: event.Hostname}
    counter := e.windows[key]  // sync.Map

    counter.mu.Lock()
    defer counter.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-time.Duration(rule.Threshold.WindowSeconds) * time.Second)

    // Evict old timestamps
    fresh := counter.timestamps[:0]
    for _, t := range counter.timestamps {
        if t.After(cutoff) {
            fresh = append(fresh, t)
        }
    }
    fresh = append(fresh, now)
    counter.timestamps = fresh

    return len(fresh) >= rule.Threshold.Count
}
```

This approach is simple, memory-efficient for demo scale, and correct. It does not persist across restarts (acceptable for the demo).

### 6.5 Rule Evaluation Pipeline

```
ParsedEvent enters engine
        │
        ▼
For each enabled rule:
    1. Check log_type match (fast exit if wrong log type)
    2. Evaluate all conditions (short-circuit AND)
    3. If threshold configured: check sliding window
    4. If threshold met (or no threshold): generate Alert
        │
        ▼
Alert written to alertChan
```

The evaluation loop runs synchronously per event. For the expected log volume (hundreds/sec), this is fast enough. Each rule evaluation is O(conditions) — typically 1–3 comparisons.

---

## 7. Alerting System

### 7.1 Real-Time Alert Flow

```
Rule Engine generates Alert struct
        │
        ▼ (sends to buffered alertChan)
Alert Dispatcher goroutine receives alert
        │
        ├──► 1. Persist to PostgreSQL (alerts table)
        ├──► 2. WebSocket Hub.Broadcast(alertJSON)
        └──► 3. Discord Webhook POST
```

The dispatcher is a single goroutine reading from `alertChan`. This serializes DB writes while keeping the rule engine non-blocking.

### 7.2 WebSocket Broadcasting

```go
// internal/websocket/hub.go
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
        case client := <-h.unregister:
            h.mu.Lock()
            delete(h.clients, client)
            close(client.send)
            h.mu.Unlock()
        case message := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // Client buffer full: drop message, don't block hub
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}
```

### 7.3 Discord Webhook Integration

```go
// internal/service/alert_service.go
type DiscordPayload struct {
    Embeds []DiscordEmbed `json:"embeds"`
}

type DiscordEmbed struct {
    Title       string `json:"title"`
    Description string `json:"description"`
    Color       int    `json:"color"`   // Decimal color based on severity
    Timestamp   string `json:"timestamp"`
}

func severityToColor(s domain.Severity) int {
    switch s {
    case domain.SeverityCritical: return 0xFF0000  // Red
    case domain.SeverityHigh:     return 0xFF6600  // Orange
    case domain.SeverityMedium:   return 0xFFCC00  // Yellow
    case domain.SeverityLow:      return 0x0099FF  // Blue
    default:                      return 0x999999  // Grey
    }
}
```

### 7.4 Retry Strategy

Discord webhooks occasionally fail. Implement a simple retry with exponential backoff:

```go
func (d *DiscordNotifier) Send(alert *domain.Alert) error {
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        if attempt > 0 {
            time.Sleep(time.Duration(attempt*attempt) * time.Second)
        }
        if err := d.post(alert); err != nil {
            lastErr = err
            continue
        }
        return nil
    }
    // Mark discord_sent = false in DB so it can be retried later
    return lastErr
}
```

---

## 8. Frontend Architecture

### 8.1 State Management Strategy

The frontend uses a **two-layer state approach:**

- **Server state:** React Query (TanStack Query) for all API data — handles caching, refetching, loading states, and error states automatically.
- **Client state:** Zustand for UI-only state (e.g., sidebar open/closed, live alert feed that grows via WebSocket).

This is the minimal viable state management. No Redux, no Context overengineering.

```typescript
// store/alertStore.ts
import { create } from 'zustand'
import type { Alert } from '../types/alert'

interface AlertStore {
  liveAlerts: Alert[]
  addAlert: (alert: Alert) => void
  clearAlerts: () => void
}

export const useAlertStore = create<AlertStore>((set) => ({
  liveAlerts: [],
  addAlert: (alert) => set((state) => ({
    liveAlerts: [alert, ...state.liveAlerts].slice(0, 100) // Keep last 100
  })),
  clearAlerts: () => set({ liveAlerts: [] }),
}))
```

### 8.2 WebSocket Integration

```typescript
// hooks/useWebSocket.ts
export function useWebSocket() {
  const addAlert = useAlertStore((s) => s.addAlert)
  const [connected, setConnected] = useState(false)

  useEffect(() => {
    let ws: WebSocket
    let reconnectTimeout: ReturnType<typeof setTimeout>

    const connect = () => {
      ws = new WebSocket(buildWsUrl('/ws'))

      ws.onopen = () => setConnected(true)
      ws.onclose = () => {
        setConnected(false)
        reconnectTimeout = setTimeout(connect, 3000)  // Reconnect after 3s
      }
      ws.onmessage = (event) => {
        const msg = JSON.parse(event.data)
        if (msg.type === 'alert') {
          addAlert(msg.payload)
        }
      }
    }

    connect()
    return () => {
      clearTimeout(reconnectTimeout)
      ws?.close()
    }
  }, [])

  return { connected }
}
```

### 8.3 API Communication Layer

```typescript
// api/client.ts
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api/v1',
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
})

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    console.error('API Error:', error.response?.data)
    return Promise.reject(error)
  }
)
```

### 8.4 Routing Strategy

React Router v6 with nested routes:

```typescript
// router.tsx
export const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      { index: true, element: <Dashboard /> },
      { path: 'alerts', element: <Alerts /> },
      { path: 'events', element: <Events /> },
      { path: 'sources', element: <LogSources /> },
      { path: 'rules', element: <Rules /> },
    ],
  },
])
```

---

## 9. UI/UX Design Plan

### 9.1 Color System

```
Background:     #0F172A  (dark navy)
Surface:        #1E293B  (card background)
Border:         #334155  (subtle borders)
Text Primary:   #F1F5F9
Text Secondary: #94A3B8

Severity Colors:
  CRITICAL:  #EF4444  (red-500)
  HIGH:      #F97316  (orange-500)
  MEDIUM:    #EAB308  (yellow-500)
  LOW:       #3B82F6  (blue-500)
  INFO:      #6B7280  (gray-500)
```

### 9.2 Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  SIEM Dashboard              [● Live] [5 alerts today]          │
├──────────┬──────────────────────────────────────────────────────┤
│          │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │
│ Sidebar  │  │  Total   │ │ Critical │ │  Active  │ │ Log    │  │
│          │  │  Events  │ │  Alerts  │ │  Sources │ │ Sources│  │
│ Dashboard│  │  12,453  │ │    3     │ │    5     │ │  OK    │  │
│ Alerts   │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │
│ Events   │                                                       │
│ Sources  │  ┌───────────────────────────┐ ┌──────────────────┐  │
│ Rules    │  │  Events Timeline (24h)    │ │ Severity Split   │  │
│          │  │  [Recharts LineChart]     │ │ [Donut Chart]    │  │
│          │  └───────────────────────────┘ └──────────────────┘  │
│          │                                                       │
│          │  ┌───────────────────────────────────────────────┐   │
│          │  │  Recent Alerts (last 10)                      │   │
│          │  │  [CRITICAL] SSH Brute Force  2m ago  [ACK]    │   │
│          │  │  [HIGH]     Sudo Failure     5m ago  [ACK]    │   │
│          │  └───────────────────────────────────────────────┘   │
└──────────┴──────────────────────────────────────────────────────┘
```

### 9.3 Alert Table Page

Columns: Severity | Rule Name | Source | Triggered At | Status | Actions

Filters: Severity (multi-select chips), Date range, Acknowledged status, Search (rule name)

The table auto-refreshes when new WebSocket alerts arrive — a toast notification appears and the table count increments in real time.

### 9.4 Real-Time Indicators

- **Connection dot** (top right): green pulse animation when WS connected, grey when disconnected.
- **Alert toast**: slides in from top-right when a new alert arrives via WebSocket.
- **Stat cards**: update in real time as alerts arrive (Zustand store).

### 9.5 Log Source Management Page

Card-based layout. Each card shows: path, log type, status badge (active/error/inactive), event count (last 24h), "Remove" button. An "Add Source" form opens as a slide-over panel with fields: name, file path, log type dropdown, description.

### 9.6 Rule Management Page

List of rule cards. Each shows: name, severity badge, enabled toggle, "Edit" / "Delete" buttons. "Add Rule" opens a YAML text editor (CodeMirror or simple `<textarea>` with monospace font). Rules are validated on submit.

---

## 10. REST API Design

### 10.1 API Versioning

All endpoints are prefixed with `/api/v1/`. If the API needs breaking changes, `/api/v2/` is introduced. Both versions can coexist in the same binary.

### 10.2 Standard Response Envelopes

**Success:**
```json
{
  "data": { ... },
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 20
  }
}
```

**Error:**
```json
{
  "error": "validation_failed",
  "message": "file_path is required",
  "details": { "field": "file_path" }
}
```

### 10.3 Full Endpoint Specification

#### Log Sources

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sources` | List all log sources |
| `POST` | `/api/v1/sources` | Register a new log source |
| `GET` | `/api/v1/sources/:id` | Get a single log source |
| `PATCH` | `/api/v1/sources/:id` | Update log source metadata |
| `DELETE` | `/api/v1/sources/:id` | Remove log source + stop watcher |

**POST /api/v1/sources — Request:**
```json
{
  "name": "System Syslog",
  "file_path": "/var/log/syslog",
  "log_type": "syslog",
  "description": "Main system log"
}
```

**GET /api/v1/sources — Response:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "System Syslog",
      "file_path": "/var/log/syslog",
      "log_type": "syslog",
      "status": "active",
      "created_at": "2025-05-05T10:00:00Z"
    }
  ],
  "meta": { "total": 1 }
}
```

#### Events

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/events` | List events with filters |
| `GET` | `/api/v1/events/:id` | Get single event |

**GET /api/v1/events — Query Params:**
- `source_id` (UUID)
- `level` (info/warn/error)
- `from` (ISO 8601)
- `to` (ISO 8601)
- `search` (string, searches message)
- `page` (int, default 1)
- `per_page` (int, default 50, max 200)

#### Alerts

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/alerts` | List alerts with filters |
| `GET` | `/api/v1/alerts/:id` | Get single alert |
| `PATCH` | `/api/v1/alerts/:id/acknowledge` | Acknowledge an alert |
| `DELETE` | `/api/v1/alerts/:id` | Delete alert |

**GET /api/v1/alerts — Query Params:**
- `severity` (comma-separated: `high,critical`)
- `acknowledged` (bool)
- `from`, `to` (ISO 8601)
- `page`, `per_page`

**PATCH /api/v1/alerts/:id/acknowledge — Response:**
```json
{
  "data": {
    "id": "...",
    "acknowledged": true,
    "acknowledged_at": "2025-05-05T10:30:00Z"
  }
}
```

#### Rules

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/rules` | List all rules |
| `POST` | `/api/v1/rules` | Create rule (YAML body) |
| `GET` | `/api/v1/rules/:id` | Get single rule |
| `PUT` | `/api/v1/rules/:id` | Replace rule YAML |
| `PATCH` | `/api/v1/rules/:id/toggle` | Enable/disable rule |
| `DELETE` | `/api/v1/rules/:id` | Delete rule |

**POST /api/v1/rules — Request:**
```json
{
  "yaml_content": "id: ssh-brute...\nname: SSH Brute Force\n..."
}
```

#### Dashboard Statistics

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/stats/overview` | Counts for stat cards |
| `GET` | `/api/v1/stats/events/timeline` | Events per hour (last 24h) |
| `GET` | `/api/v1/stats/alerts/by-severity` | Alert count grouped by severity |
| `GET` | `/api/v1/stats/top-sources` | Top 5 sources by event count |

**GET /api/v1/stats/overview — Response:**
```json
{
  "data": {
    "total_events_24h": 12453,
    "total_alerts_24h": 47,
    "critical_alerts": 3,
    "active_sources": 5
  }
}
```

**GET /api/v1/stats/events/timeline — Response:**
```json
{
  "data": [
    { "hour": "2025-05-05T10:00:00Z", "count": 234 },
    { "hour": "2025-05-05T11:00:00Z", "count": 189 }
  ]
}
```

---

## 11. WebSocket Design

### 11.1 Connection Lifecycle

```
Client → WS Upgrade Request (/ws)
       ← 101 Switching Protocols
       
Client registered in Hub.clients

Client ← {"type":"connected","payload":{"message":"SIEM connected"}}

[Loop]
Rule Engine fires Alert
    → Hub.broadcast channel
    → Hub goroutine sends to all clients
    → Client ← {"type":"alert","payload":{...alert...}}

Client disconnects (browser tab closes / network drop)
    → Hub.unregister channel
    → Hub removes client
```

### 11.2 Message Envelope

All WebSocket messages use a consistent envelope:

```typescript
interface WSMessage {
  type: 'alert' | 'event' | 'source_status' | 'connected' | 'ping'
  payload: Alert | ParsedEvent | SourceStatusUpdate | { message: string }
  timestamp: string  // ISO 8601
}
```

### 11.3 Example Payloads

**Alert message:**
```json
{
  "type": "alert",
  "timestamp": "2025-05-05T12:34:56Z",
  "payload": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "rule_name": "SSH Brute Force Detected",
    "severity": "high",
    "title": "SSH Brute Force: web-server-01",
    "description": "5 failed SSH logins in 60s from web-server-01",
    "raw_line": "May  5 12:34:51 web-server-01 sshd[2345]: Failed password for root",
    "triggered_at": "2025-05-05T12:34:56Z"
  }
}
```

**Source status update:**
```json
{
  "type": "source_status",
  "timestamp": "2025-05-05T12:34:56Z",
  "payload": {
    "source_id": "...",
    "status": "error",
    "message": "File not found: /var/log/app.log"
  }
}
```

**Ping (keepalive):**
```json
{ "type": "ping", "timestamp": "2025-05-05T12:34:56Z", "payload": {} }
```

### 11.4 Reconnection Strategy

The frontend uses exponential backoff with jitter:

```typescript
const delay = Math.min(1000 * 2 ** attempt + Math.random() * 1000, 30000)
```

Attempts: 1s → 2s → 4s → 8s → 16s → 30s (capped). The status dot turns grey with a "Reconnecting..." label.

---

## 12. DevOps Architecture

### 12.1 Docker Compose Structure

```yaml
# infra/docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: siem
      POSTGRES_USER: siem
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U siem"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ../backend
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: siem
      DB_USER: siem
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DISCORD_WEBHOOK_URL: ${DISCORD_WEBHOOK_URL}
      SERVER_PORT: 8080
      RULES_DIR: /app/rules
    volumes:
      - /var/log:/host/logs:ro   # Mount host log directory read-only
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  frontend:
    build:
      context: ../frontend
      dockerfile: Dockerfile
    # Static files served by nginx, not a runtime service
    # This service only exists to build the static files
    profiles: ["build"]  # Run manually during CI

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - frontend_dist:/usr/share/nginx/html:ro  # Built React app
      - ./certs:/etc/nginx/certs:ro  # TLS certs
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  frontend_dist:

networks:
  default:
    name: siem_network
```

### 12.2 Backend Dockerfile

```dockerfile
# Multi-stage build — keeps final image small
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o siem-server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/siem-server .
COPY --from=builder /app/rules ./rules
USER nobody:nobody
EXPOSE 8080
ENTRYPOINT ["./siem-server"]
```

### 12.3 Frontend Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### 12.4 Nginx Configuration

```nginx
# infra/nginx/conf.d/siem.conf
server {
    listen 80;
    server_name _;

    # Redirect to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your.domain.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # React SPA — serve index.html for all non-API paths
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
        expires 1h;
    }

    # API proxy
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # WebSocket proxy (requires special headers)
    location /ws {
        proxy_pass http://backend:8080/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;  # 24h — keep WS alive
    }
}
```

### 12.5 Environment Variable Management

All secrets live in a `.env` file on the VPS, never committed to git:

```bash
# .env (on VPS only, in .gitignore)
POSTGRES_PASSWORD=s3cur3_p4ssw0rd_here
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...
```

`.env.example` is committed to the repository with placeholder values as documentation.

---

## 13. Jenkins CI/CD Pipeline

### 13.1 Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE_BACKEND  = "siem-backend"
        DOCKER_IMAGE_FRONTEND = "siem-frontend"
        SONAR_PROJECT_KEY     = "siem-backend"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend: Lint & Test') {
            steps {
                dir('backend') {
                    sh 'go vet ./...'
                    sh 'go test ./... -coverprofile=coverage.out -covermode=atomic'
                }
            }
            post {
                always {
                    junit 'backend/test-results/*.xml'
                }
            }
        }

        stage('Frontend: Install & Build') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    dir('backend') {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.sources=. \
                              -Dsonar.go.coverage.reportPaths=coverage.out
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} ./backend"
                sh "docker build -t ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} ./frontend"
                sh "docker tag ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_BACKEND}:latest"
                sh "docker tag ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_FRONTEND}:latest"
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sshagent(['vps-deploy-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no deploy@${VPS_HOST} \
                          "cd /opt/siem && \
                           docker compose pull && \
                           docker compose up -d --remove-orphans"
                    '''
                }
            }
        }
    }

    post {
        failure {
            discordSend \
                webhookURL: env.DISCORD_WEBHOOK_URL,
                title: "Build FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                link: env.BUILD_URL,
                result: currentBuild.currentResult
        }
        success {
            echo "Pipeline succeeded."
        }
    }
}
```

### 13.2 Branch Strategy

| Branch | Trigger | Action |
|---|---|---|
| `feature/*` | Push | Lint + Test only |
| `develop` | Push / PR merge | Full pipeline minus Deploy |
| `main` | PR merge | Full pipeline + Deploy |

Keep it simple: `feature/` → `develop` → `main`. No release branches needed for 11 days.

### 13.3 Rollback Consideration

Every Docker image is tagged with `BUILD_NUMBER`. To rollback:

```bash
docker compose stop backend
docker run -d --name siem-backend siem-backend:42  # Previous build
```

Because images are stored locally on the VPS, rollback is instant.

---

## 14. SonarQube Integration

### 14.1 sonar-project.properties (Backend)

```properties
sonar.projectKey=siem-backend
sonar.projectName=SIEM Backend
sonar.sources=.
sonar.exclusions=**/*_test.go,**/vendor/**,**/migrations/**
sonar.tests=.
sonar.test.inclusions=**/*_test.go
sonar.go.coverage.reportPaths=coverage.out
sonar.host.url=http://sonarqube:9000
```

### 14.2 Recommended Quality Gates

| Metric | Threshold |
|---|---|
| Coverage (new code) | ≥ 50% |
| Duplicated Lines | ≤ 3% |
| Maintainability Rating | A |
| Reliability Rating | A (0 bugs) |
| Security Rating | A (0 vulnerabilities) |
| Security Hotspots Reviewed | 100% |

For a university project, 50% coverage on new code is realistic and meaningful. Targeting 80%+ would consume disproportionate development time.

### 14.3 What SonarQube Will Catch

- SQL injection risks (parameterized query validation)
- Hardcoded credentials
- Unhandled errors
- Dead code / unreachable code
- Overly complex functions (cyclomatic complexity)
- Missing error checks on `Close()` calls

Address `BLOCKER` and `CRITICAL` issues immediately. `MAJOR` issues can be reviewed; `MINOR` and `INFO` can be noted but not blocked.

---

## 15. Security Best Practices

### 15.1 SQL Injection Prevention

**Never use string concatenation in queries.** Always use parameterized queries:

```go
// ✅ CORRECT
row := db.QueryRow(ctx, "SELECT * FROM alerts WHERE id = $1", id)

// ❌ WRONG — never do this
query := "SELECT * FROM alerts WHERE id = '" + id + "'"
```

`pgx` (the recommended PostgreSQL driver) always uses parameterized queries when you use `$1` syntax.

### 15.2 Input Validation

Validate all user inputs at the handler layer before they reach services:

```go
type CreateLogSourceRequest struct {
    Name     string `json:"name" binding:"required,min=1,max=255"`
    FilePath string `json:"file_path" binding:"required"`
    LogType  string `json:"log_type" binding:"required,oneof=syslog nginx generic"`
}
```

Use Gin's built-in `binding` tags (backed by the `validator` library). Additional validation for file paths:

```go
// Prevent path traversal — only allow absolute paths to known safe prefixes
func validateFilePath(path string) error {
    if !filepath.IsAbs(path) {
        return errors.New("file_path must be an absolute path")
    }
    // Optional: restrict to specific directories
    allowed := []string{"/var/log", "/host/logs"}
    for _, prefix := range allowed {
        if strings.HasPrefix(path, prefix) {
            return nil
        }
    }
    return errors.New("file_path must be under /var/log or /host/logs")
}
```

### 15.3 XSS Prevention

The frontend uses React, which escapes all rendered values by default. Never use `dangerouslySetInnerHTML`. For the YAML editor, use a `<textarea>` — not an HTML renderer. All alert descriptions are rendered as text, not HTML.

### 15.4 WebSocket Security

- Validate the `Origin` header on WebSocket upgrade requests in production.
- Use `wss://` (WebSocket over TLS) in production, handled by Nginx.
- Implement connection rate limiting at the Nginx level.

### 15.5 Docker Security

- Run the backend container as `nobody:nobody` (non-root user, already in the Dockerfile).
- Mount host log directories as `:ro` (read-only).
- Never expose PostgreSQL port externally — only accessible within the Docker network.
- Use `docker compose` network isolation — backend can reach postgres, Nginx can reach backend, nothing else.

### 15.6 Secrets Management

- All secrets in environment variables.
- `.env` file on VPS only, never in git.
- Jenkins secrets stored in Jenkins Credentials store, injected via `withCredentials`.
- No hardcoded credentials anywhere in source code (SonarQube will flag these).

### 15.7 Rate Limiting

Add basic rate limiting at the Nginx level:

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;

location /api/ {
    limit_req zone=api burst=10 nodelay;
    proxy_pass http://backend:8080;
}
```

This prevents the API from being hammered during the demo. Simple and effective without application-level complexity.

---

## 16. Implementation Roadmap

### 16.1 11-Day Plan

#### Day 1 — Foundation & Scaffolding
**Goal:** Every developer has a working local environment with running services.

- [ ] Set up Git repository with `main`/`develop`/`feature/*` branch structure
- [ ] Initialize Go module, install dependencies (`gin`, `pgx`, `gorilla/websocket`, `fsnotify`, `go-yaml`)
- [ ] Initialize React project with Vite + Tailwind + React Router + React Query + Zustand
- [ ] Write PostgreSQL schema (`migrations/001_create_tables.sql`)
- [ ] Create `docker-compose.dev.yml` with postgres + backend + frontend (hot-reload)
- [ ] Verify all three services start and connect

**Deliverable:** `docker compose up` starts all services. Frontend shows a blank page. Backend responds to `GET /api/v1/health`.

---

#### Day 2 — Backend Core: API Skeleton + DB Layer
**Goal:** Full API skeleton with real database reads/writes.

- [ ] Implement `Config` loader
- [ ] Implement all domain structs
- [ ] Implement repository layer for all 4 entities (log_sources, events, rules, alerts)
- [ ] Implement service layer skeletons
- [ ] Implement all REST handlers (CRUD, returning dummy or real data)
- [ ] Test all endpoints with curl/Postman

**Deliverable:** Can POST a log source and GET it back from PostgreSQL.

---

#### Day 3 — Log Collection: Watcher + Parser Pipeline
**Goal:** Real log lines flow from files into the database.

- [ ] Implement `FileWatcher` with fsnotify + offset tracking
- [ ] Implement `WatcherRegistry`
- [ ] Implement `SyslogParser` with regex
- [ ] Implement `GenericParser` as fallback
- [ ] Wire: `POST /sources` → WatcherRegistry → FileWatcher → parsedEventChan → EventService → DB
- [ ] Test: Add `/var/log/syslog` (or a test file), verify events appear in DB

**Deliverable:** Log events from a real file are stored in PostgreSQL within seconds of being written.

---

#### Day 4 — Rule Engine
**Goal:** Rules load from YAML and fire alerts when conditions match.

- [ ] Implement YAML rule schema and loader
- [ ] Implement `RuleMatcher` with all operators
- [ ] Implement threshold/window logic
- [ ] Implement `AlertGenerator`
- [ ] Wire: `parsedEventChan` → `RuleEngine.Evaluate()` → `alertChan`
- [ ] Implement `AlertDispatcher`: reads alertChan → writes alert to DB
- [ ] Load sample rules (`ssh_brute_force.yaml`, `failed_login.yaml`)
- [ ] Test: Trigger a rule by writing matching log lines to a watched file

**Deliverable:** Writing "Failed password" to a watched syslog file generates an alert in the `alerts` table.

---

#### Day 5 — WebSocket + Discord
**Goal:** Alerts appear in real time. Discord notifications fire.

- [ ] Implement WebSocket Hub
- [ ] Implement WebSocket Client handler
- [ ] Add `/ws` route to Gin
- [ ] Wire: `AlertDispatcher` → `Hub.Broadcast(alertJSON)`
- [ ] Implement Discord webhook notifier with retry
- [ ] Wire: `AlertDispatcher` → `DiscordNotifier.Send(alert)`
- [ ] Test WebSocket with browser dev tools or `wscat`

**Deliverable:** Trigger a rule → alert appears instantly in browser WS console + Discord message received.

---

#### Day 6 — Frontend: Layout + Dashboard
**Goal:** Working dashboard with real data from the API.

- [ ] Implement `Layout` (Sidebar + TopBar)
- [ ] Implement `Dashboard` page with stat cards (real API calls)
- [ ] Implement `EventsTimelineChart` (Recharts LineChart with `/stats/events/timeline`)
- [ ] Implement `SeverityDonutChart` (Recharts PieChart with `/stats/alerts/by-severity`)
- [ ] Implement `useWebSocket` hook — connect to `/ws`
- [ ] Show live connection status dot

**Deliverable:** Dashboard loads with real charts. Connection dot shows green.

---

#### Day 7 — Frontend: Alerts + Events Pages
**Goal:** Alert table with real-time updates and event browser.

- [ ] Implement `Alerts` page with paginated table
- [ ] Implement severity badge component
- [ ] Implement alert acknowledge action
- [ ] Implement alert filters (severity, date range)
- [ ] Implement real-time alert feed: new WS alert → toast notification + table refresh
- [ ] Implement `Events` page with paginated table and filters
- [ ] Implement `useAlertStore` (Zustand)

**Deliverable:** Trigger a rule → alert toast appears on screen immediately. Alert table updates.

---

#### Day 8 — Frontend: Log Sources + Rules Pages
**Goal:** Full management UI for sources and rules.

- [ ] Implement `LogSources` page with source cards
- [ ] Implement "Add Source" slide-over form
- [ ] Implement source delete with confirmation
- [ ] Implement `Rules` page with rule list
- [ ] Implement YAML rule editor (textarea with monospace font)
- [ ] Implement rule enable/disable toggle
- [ ] Implement rule create/edit/delete

**Deliverable:** Can add a new log source from the UI and see it begin collecting events.

---

#### Day 9 — DevOps: Docker + Nginx + Jenkins
**Goal:** Full CI/CD pipeline operational.

- [ ] Write production `docker-compose.yml`
- [ ] Write Nginx config (reverse proxy + WS + static files)
- [ ] Write backend Dockerfile (multi-stage)
- [ ] Write frontend Dockerfile
- [ ] Configure Jenkins pipeline on CI server
- [ ] Configure SonarQube instance
- [ ] Set up VPS: install Docker, Docker Compose, Nginx, obtain TLS cert (Let's Encrypt or self-signed)
- [ ] Run first successful deployment to VPS

**Deliverable:** Push to `main` → Jenkins pipeline runs → deploys to VPS → accessible at public URL.

---

#### Day 10 — Integration Testing + Bug Fixes
**Goal:** End-to-end demo flow works flawlessly.

- [ ] Run full end-to-end test: add source → watch events → trigger rule → receive alert → Discord notification
- [ ] Fix all critical bugs found
- [ ] Add `NginxParser` for nginx access logs
- [ ] Write backend unit tests (parsers, rule matchers)
- [ ] Verify SonarQube quality gate passes
- [ ] Performance spot check: no memory leaks in watcher goroutines
- [ ] Ensure all API endpoints return correct error responses

**Deliverable:** Clean demo flow with no crashes. SonarQube green.

---

#### Day 11 — Polish + Demo Prep
**Goal:** Demo-ready. Impressive, stable, rehearsed.

- [ ] UI polish: loading states, empty states, error states
- [ ] Seed demo data: pre-populate events, alerts, rules in DB for demo
- [ ] Write a demo script (what to click, what commands to run)
- [ ] Create presentation slides with architecture diagram
- [ ] Rehearse demo with the full team
- [ ] Final deployment to VPS
- [ ] Document the README (setup instructions, architecture summary)

**Deliverable:** A polished, working SIEM demo ready for presentation.

---

### 16.2 Team Task Breakdown

**Developer A — Backend Lead**
- Days 1-5: Core backend (watcher, parser, rule engine, websocket)
- Days 6-8: API debugging, backend fixes while frontend develops
- Days 9-11: DevOps, Jenkins, deployment

**Developer B — Frontend Lead**
- Days 1-2: Frontend scaffolding, API client, types
- Days 6-8: All React pages and components
- Days 9-11: UI polish, demo prep

**Developer C — Full Stack / DevOps**
- Days 1-2: Database schema, docker-compose dev setup
- Days 3-5: Assists backend (repository layer, service layer)
- Days 6-8: Assists frontend (charts, hooks)
- Days 9-11: Jenkins, Nginx, SonarQube, VPS setup

### 16.3 Parallel Work Strategy

```
Day 1:  A: Go setup         B: React setup      C: DB schema + Docker
Day 2:  A: API handlers     B: API client       C: Repository layer
Day 3:  A: Watcher/Parser   B: Layout/Nav       C: Service layer
Day 4:  A: Rule engine      B: Dashboard UI     C: Rule YAML loader
Day 5:  A: WS + Discord     B: Charts           C: Testing + Bugfix
Day 6:  A: Bugfix/Docs      B: Alerts page      C: Events page
Day 7:  A: Nginx parser     B: Sources page     C: Rules page
Day 8:  A: Tests            B: UI polish        C: DevOps setup
Day 9:  A: Jenkins          B: Demo data seed   C: VPS deploy
Day 10: ALL: Integration testing + bug fixing
Day 11: ALL: Demo prep + rehearsal
```

---

## 17. Risk Management

### 17.1 Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| VPS setup takes too long | Medium | High | Start VPS setup on Day 8. Use DigitalOcean/Hetzner droplet for fast provisioning. |
| Jenkins/SonarQube configuration problems | High | Medium | Use Docker images for Jenkins and SonarQube. Avoids local install issues. |
| fsnotify doesn't work on mounted volumes | Medium | High | Test Day 3 immediately. Fallback: polling with a ticker every 500ms. |
| PostgreSQL migration issue at startup | Low | Medium | Use simple init.sql in Docker entrypoint. Test locally first. |
| Rule engine has wrong threshold logic | Medium | High | Write unit tests for threshold logic on Day 4 before integration. |
| WebSocket doesn't work through Nginx | Medium | Medium | Nginx config for WS is provided. Test on Day 9 first thing. |
| Team member gets sick | Low | Very High | Code review protocol: every PR reviewed before merge so others understand all code. |
| Time overrun on nice-to-have features | Very High | Medium | Strict scope control: ONLY implement the must-have list until Day 10. |

### 17.2 Fallback Strategies

**If fsnotify fails on Docker volumes:**
Replace the fsnotify watcher with a polling loop:
```go
ticker := time.NewTicker(500 * time.Millisecond)
for range ticker.C {
    w.readNewLines()  // Reads new lines using offset
}
```
This is less efficient but 100% reliable. Implementation time: 30 minutes.

**If Jenkins CI is too complex to configure:**
Replace with a simple `deploy.sh` script that SSHs into the VPS and runs `docker compose pull && docker compose up -d`. Still demonstrates deployment automation. Implementation time: 1 hour.

**If SonarQube quality gate blocks the pipeline:**
Lower the thresholds. Coverage gate can drop to 30%. The goal is demonstrating SonarQube integration, not perfect metrics.

**If Discord webhook is unreliable:**
Remove retry logic, log failure, continue. Alerts still work; just the notification fails gracefully.

### 17.3 Scope Control Strategy

**Must-Have (do not cut):**
- Log file watching + parsing
- Rule engine with at least 2 working rules
- Alert generation and persistence
- Real-time WebSocket alerts on dashboard
- Basic REST API for all entities
- Docker Compose deployment
- Jenkins pipeline (even if minimal)

**Nice-to-Have (cut if behind):**
- Nginx parser (syslog + generic is sufficient for demo)
- Alert acknowledgment UI
- Rules management UI (show rules as read-only list)
- SonarQube quality gate (show integration even if gate doesn't enforce)
- Dashboard charts (stat cards are sufficient for minimal demo)

**Cut Immediately If Day 9 Behind:**
- TLS/HTTPS (demo over HTTP is fine for a university project)
- Discord retry logic (fire and forget)
- Event browser page (redirect to a simple table or remove from nav)

---

## 18. Testing Strategy

### 18.1 Backend Testing (Go)

**Unit tests — highest priority:**

```go
// internal/parser/syslog_parser_test.go
func TestSyslogParser_Parse(t *testing.T) {
    p := &SyslogParser{}
    tests := []struct {
        name     string
        input    string
        wantErr  bool
        wantMsg  string
    }{
        {
            name:    "valid syslog line",
            input:   "May  5 12:34:56 web-01 sshd[1234]: Failed password for root",
            wantMsg: "Failed password for root",
        },
        {
            name:    "malformed line",
            input:   "not a syslog line at all",
            wantErr: true,
        },
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            event, err := p.Parse(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
                return
            }
            assert.NoError(t, err)
            assert.Equal(t, tt.wantMsg, event.Message)
        })
    }
}
```

**Test what absolutely must work:**
- All parsers (syslog, nginx, generic)
- All rule condition operators
- Threshold window logic (edge cases: exactly at threshold, below threshold)
- Alert dispatcher routing

**Deprioritize:**
- Repository layer tests (require real DB or testcontainers — too heavy for 11 days)
- End-to-end HTTP handler tests

**Run tests:**
```bash
go test ./internal/parser/... ./internal/ruleengine/... -v -race
```

The `-race` flag detects goroutine race conditions — critical for concurrent code.

### 18.2 Frontend Testing

**Deprioritize frontend testing for 11 days.** The time investment is too high relative to the demo value. Instead, verify manually. If time allows on Day 10, add one integration test for the dashboard data fetch using Vitest + React Testing Library.

### 18.3 Manual Testing Checklist

```
CORE FLOW:
[ ] Start all services with docker compose up
[ ] Add a log source via the UI (e.g., /var/log/syslog)
[ ] Verify events appear in the Events page within 5 seconds
[ ] Write a line matching the SSH brute force rule to the watched file
[ ] Verify alert appears on Dashboard within 2 seconds
[ ] Verify alert appears in Alerts page
[ ] Verify Discord webhook fires
[ ] Acknowledge the alert in the UI
[ ] Verify acknowledged status updates

API TESTS (curl):
[ ] POST /api/v1/sources — returns 201 with created source
[ ] GET /api/v1/sources — returns array
[ ] POST /api/v1/rules — returns 201 with created rule
[ ] GET /api/v1/stats/overview — returns counts
[ ] GET /api/v1/alerts?severity=high — filters correctly

RESILIENCE:
[ ] Stop the backend, restart it — watchers resume, WebSocket reconnects
[ ] Delete a watched log file — watcher logs error, doesn't crash
[ ] Send malformed JSON to API — returns 400, not 500
[ ] Open 3 browser tabs — all receive the same WebSocket alert simultaneously
```

---

## 19. Deployment Strategy

### 19.1 VPS Setup Checklist

```bash
# On a fresh Ubuntu 22.04 VPS

# 1. System update
sudo apt update && sudo apt upgrade -y

# 2. Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 3. Install Docker Compose v2
sudo apt install docker-compose-plugin

# 4. Create deployment user
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy

# 5. Create app directory
sudo mkdir -p /opt/siem
sudo chown deploy:deploy /opt/siem

# 6. Copy files
scp -r ./infra deploy@VPS_IP:/opt/siem/

# 7. Create .env file on VPS
cat > /opt/siem/.env << EOF
POSTGRES_PASSWORD=your_secure_password
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...
EOF

# 8. Get TLS certificate (Let's Encrypt)
sudo apt install certbot
sudo certbot certonly --standalone -d your.domain.com
# Certs at: /etc/letsencrypt/live/your.domain.com/

# 9. Start services
cd /opt/siem && docker compose up -d
```

### 19.2 Production Deployment Checklist

```
PRE-DEPLOY:
[ ] All tests pass locally
[ ] Jenkins pipeline green
[ ] SonarQube quality gate passed
[ ] .env file on VPS is up to date
[ ] TLS certificate valid (check expiry)

DEPLOY:
[ ] Pull latest images: docker compose pull
[ ] Start services: docker compose up -d --remove-orphans
[ ] Check logs: docker compose logs -f backend (watch for startup errors)
[ ] Verify health: curl https://your.domain.com/api/v1/health
[ ] Verify WebSocket: browser console shows "SIEM connected"
[ ] Verify nginx serves frontend: https://your.domain.com loads dashboard

POST-DEPLOY:
[ ] Trigger a test alert and verify end-to-end flow
[ ] Check Discord webhook fires
[ ] Verify PostgreSQL has data: docker compose exec postgres psql -U siem -c "SELECT COUNT(*) FROM events"
```

### 19.3 Keeping Services Alive

Use Docker's `restart: unless-stopped` policy (already in the compose file). If the VPS reboots, all services automatically restart. This is sufficient for a demo project — no need for systemd service units.

---

## 20. Performance Considerations

### 20.1 PostgreSQL Query Optimization

The most important query is the events timeline for the dashboard:

```sql
-- This query runs every 30 seconds for the chart
SELECT 
    date_trunc('hour', event_time) AS hour,
    COUNT(*) AS count
FROM events
WHERE event_time > NOW() - INTERVAL '24 hours'
GROUP BY 1
ORDER BY 1;
```

This query benefits from the `idx_events_event_time` index. At 100,000 events/24h, this query completes in under 50ms on a modest VPS.

**Pagination pattern — always use LIMIT + OFFSET with ORDER BY indexed column:**
```sql
SELECT * FROM events
ORDER BY event_time DESC
LIMIT 50 OFFSET 0;
```

### 20.2 WebSocket Scaling

The Hub's broadcast loop is a single goroutine — serialized fan-out. For dozens of concurrent clients (demo scenario), this is perfectly adequate. The per-client send channel has a 256-byte buffer. Slow clients are evicted (their channel blocks), protecting the hub from backpressure.

### 20.3 Parser Efficiency

Regexes are compiled once at package init with `regexp.MustCompile`. In Go, compiled regexes are safe for concurrent use — multiple goroutines can call `regex.FindStringSubmatch()` simultaneously without locking. For typical log volumes (thousands of lines/second), regex parsing is not a bottleneck.

### 20.4 Goroutine Efficiency

Each watched file uses one goroutine. For 10 watched files, this is 10 goroutines — negligible overhead in Go. Goroutines are cheap (2KB stack initial), and Go's runtime efficiently multiplexes them.

### 20.5 Memory Considerations

- The threshold window engine keeps timestamps in memory per (rule, group_key). For demo scale (5 rules, 10 hosts), this is under 10KB of data.
- The parsedEventChan buffer (1000 events) uses at most ~200KB of memory at peak.
- PostgreSQL connections use a pool of 10 connections — each idle connection uses ~5MB on the server side.

### 20.6 Realistic Scaling Expectations

On a $5/month VPS (1 CPU, 1GB RAM):

| Metric | Realistic Capacity |
|---|---|
| Log lines/second sustained | 500–1,000 |
| Concurrent WebSocket clients | 50–100 |
| Events stored per day | 5–10 million (with disk space) |
| API requests/second | 100–200 |
| PostgreSQL connections | 10 (pooled) |

This is far beyond any university demo workload. The system will appear fast and responsive throughout the presentation.

---

## 21. Final Recommendations

### 21.1 What to Prioritize

1. **The ingest pipeline first.** Log file → parsed event → PostgreSQL. This is the backbone of the whole system. Nothing else works without it. Allocate Days 2–3 entirely to getting this correct.

2. **The rule engine second.** A SIEM without alerting is just a log viewer. Rules and alerts are the system's primary value proposition. Get at least two working rules before touching the frontend.

3. **Real-time WebSocket third.** This is the most visually impressive feature for the demo. A live alert appearing on screen when a log rule fires will impress evaluators. Prioritize it over completeness of the management UI.

4. **A clean, dark dashboard UI.** First impressions matter for demos. A polished dark-themed dashboard communicates professionalism even if some features are incomplete.

### 21.2 What to Simplify

- **No authentication.** Save 2 days. An API key can be added as last-resort hardening.
- **No complex rule DSL.** Simple YAML with AND conditions is sufficient. Skip OR logic, nested conditions, and correlation across log sources for now.
- **No automatic retention.** Manual database cleanup if needed during testing.
- **No horizontal scaling.** Single VPS, single Docker Compose stack. It's a demo, not production.
- **Frontend state management.** React Query + Zustand covers everything. Don't add Redux.

### 21.3 What to Avoid

- **Avoid Kafka/Redis.** The PostgreSQL + in-memory channel approach is fast enough and far simpler.
- **Avoid Kubernetes.** Docker Compose on a single VPS is the right tool for this scope.
- **Avoid WebAssembly or edge computing features.** Out of scope.
- **Avoid building an authentication system from scratch.** It's a time sink with low demo value.
- **Avoid over-testing.** 50% unit test coverage on critical paths is the target. Don't spend Day 10 writing tests for features that already work.
- **Avoid premature optimization.** Profile before optimizing. At demo scale, nothing will be slow.

### 21.4 What Makes This Project Impressive

The following features, when they work together seamlessly, will create a genuinely impressive demo:

1. **Live demonstration of the ingest pipeline** — Write a test log line, show it appear in the Events page.
2. **Rule firing in real time** — Simulate an SSH brute force with a script, show the alert appear on the dashboard within 2 seconds.
3. **Discord notification** — Show the Discord channel receiving the alert simultaneously.
4. **Architecture diagram** — A well-drawn architecture diagram demonstrates engineering maturity beyond the code itself.
5. **CI/CD pipeline running live** — Push a commit during the demo, show Jenkins running, SonarQube reporting, VPS deploying.

### 21.5 Feature Priority Matrix

| Feature | Category | Effort | Impact |
|---|---|---|---|
| Log file watching + ingestion | Must-Have | High | Critical |
| Syslog parser | Must-Have | Medium | Critical |
| Rule engine (2+ rules) | Must-Have | High | Critical |
| Alert generation + persistence | Must-Have | Medium | Critical |
| WebSocket real-time alerts | Must-Have | Medium | Very High |
| Dashboard UI with stats | Must-Have | Medium | Very High |
| Discord webhook | Must-Have | Low | High |
| Docker Compose deployment | Must-Have | Medium | High |
| Jenkins CI pipeline | Must-Have | Medium | High |
| Alert table with filters | Must-Have | Medium | High |
| SonarQube integration | Must-Have | Medium | Medium |
| Log source management UI | Must-Have | Medium | Medium |
| Rules management UI | Nice-to-Have | Medium | Medium |
| Nginx parser | Nice-to-Have | Low | Medium |
| Event browser with filters | Nice-to-Have | Medium | Low |
| Alert acknowledgment | Nice-to-Have | Low | Low |
| HTTPS/TLS | Nice-to-Have | Low | Low |
| Threshold window logic | Must-Have | Medium | High |

### 21.6 The One-Sentence Architecture Principle

> **Keep it simple enough to work, sophisticated enough to impress.**

A stable SIEM that processes real logs, fires real alerts, shows them in a beautiful real-time dashboard, and deploys via an automated CI/CD pipeline is a genuinely impressive piece of engineering for a university capstone — with or without every feature on the list.

---

*Document prepared for: Custom SIEM University Capstone Project*  
*Architecture by: Principal Software Architect / Senior DevOps Engineer / Senior Security Engineer role simulation*  
*Version: 1.0 | Date: May 2025*
