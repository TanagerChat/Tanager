# Tanager Chat — Architecture

> Rust-native, open-source team chat. Self-hostable on a Raspberry Pi, scalable to a multi-tenant SaaS.

## 1. Guiding Principles

- **Single-binary simplicity for self-hosters.** A homelab user runs one binary + PostgreSQL and gets the full experience. No Redis, no message queue, no external dependencies required at small scale.
- **Scale-out, not scale-up.** Every architectural decision must allow horizontal scaling by adding nodes, not by buying a bigger server.
- **Workspace as the isolation boundary.** All data is scoped to a workspace. This is the natural sharding key, the tenancy boundary, and the security perimeter.
- **Interface-driven storage.** Business logic talks to traits, not to PostgreSQL directly. Backing stores can be swapped without rewriting application code.
- **API-first.** The server is a real-time API that happens to serve a web client, not a web app that happens to have an API.

---

## 2. High-Level Overview

```
┌─────────────────────────────────────────────────────────┐
│                      Clients                            │
│         (Web · Mobile · CLI · Bot SDK)                  │
└──────────────┬──────────────────────┬───────────────────┘
               │ HTTPS/REST           │ WebSocket
               ▼                      ▼
┌─────────────────────────────────────────────────────────┐
│                  API Gateway (Axum)                     │
│                                                         │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │   Auth &    │ │   REST API   │ │  WebSocket Hub   │  │
│  │   Session   │ │   Handlers   │ │  (Real-time)     │  │
│  └──────┬──────┘ └──────┬───────┘ └────────┬─────────┘  │
│         │               │                  │            │
│         ▼               ▼                  ▼            │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Service Layer (Domain Logic)        │   │
│  │                                                  │   │
│  │  Workspace · Channel · Message · User · File     │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │                               │
│  ┌──────────────────────▼───────────────────────────┐   │
│  │            Storage Traits (Interfaces)           │   │
│  │                                                  │   │
│  │  MessageStore · ChannelStore · UserStore · ...   │   │
│  └──────┬──────────────┬──────────────┬─────────────┘   │
│         │              │              │                 │
└─────────┼──────────────┼──────────────┼─────────────────┘
          ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ PostgreSQL │ │ S3/MinIO   │ │ NATS       │
   │ (+ Citus)  │ │ (Files)    │ │ (Pub/Sub)  │
   └────────────┘ └────────────┘ └────────────┘
```

---

## 3. Database Architecture

### 3.1 PostgreSQL as the Foundation

PostgreSQL is the sole database for all structured data: users, workspaces, channels, messages, reactions, permissions, and audit logs. It provides ACID guarantees, full-text search via `tsvector`, and JSONB for flexible metadata — all without introducing additional infrastructure.

### 3.2 Sharding Strategy: Workspace as the Distribution Key

Every table includes a `workspace_id` column. Every query is scoped by `workspace_id`. This is non-negotiable — it is the foundation of both multi-tenancy and horizontal scaling.

**Why `workspace_id` and not `channel_id` or `user_id`?**

- A workspace's data is self-contained. Cross-workspace queries are rare (admin analytics only) and can tolerate slower execution.
- Users belong to workspaces. Channels belong to workspaces. Messages belong to channels within workspaces. The hierarchy is clean.
- Workspace-level sharding means a single shard holds all the data a user needs during a session, minimizing cross-shard joins.

### 3.3 Scale-Out Path with Citus

[Citus](https://github.com/citusdata/citus) is an open-source PostgreSQL extension (AGPL) that enables transparent horizontal sharding. It is free to self-host.

**How it works:**

1. You designate `workspace_id` as the distribution column.
2. Citus shards tables across multiple PostgreSQL worker nodes.
3. SQL queries that include `workspace_id` in the WHERE clause are routed to the correct shard automatically.
4. The coordinator node handles query planning and distribution.

**Deployment progression:**

| Stage | Setup | When |
|-------|-------|------|
| Homelab | Single PostgreSQL node (Citus extension installed but idle) | Day one |
| Small SaaS | Single PostgreSQL node with Citus managing local shards | < 100 workspaces |
| Growing SaaS | Citus coordinator + 2-4 worker nodes | 100-10,000 workspaces |
| Large SaaS | Citus coordinator + N worker nodes, read replicas per shard | 10,000+ workspaces |

**Key benefit:** the application code does not change between stages. The same SQL, the same connection string, the same ORM queries. Citus handles the routing.

### 3.4 Schema Design Principles

- **Composite primary keys.** All tables use `(workspace_id, id)` as the primary key. This ensures co-located data on the same shard and prevents ID collisions across workspaces.
- **ULIDs for identifiers.** Lexicographically sortable, timestamp-embedded, and globally unique without coordination. Preferred over UUIDs v4 for index-friendly inserts.
- **Timestamps everywhere.** `created_at` and `updated_at` on every table, stored as `TIMESTAMPTZ`. Message ordering uses `(workspace_id, channel_id, created_at, id)` to guarantee deterministic sort order even with identical timestamps.
- **Soft deletes for messages.** Messages are marked `deleted_at` rather than removed. This preserves threading integrity and allows compliance exports.
- **JSONB for extensibility.** A `metadata JSONB` column on messages, channels, and workspaces for future feature flags, custom fields, and integrations without schema migrations.

### 3.5 Core Tables

```sql
-- Workspaces
CREATE TABLE workspaces (
    id          ULID PRIMARY KEY,
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL UNIQUE,
    owner_id    ULID NOT NULL,
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Users (global, not sharded)
CREATE TABLE users (
    id              ULID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,               -- NULL if OAuth-only
    avatar_url      TEXT,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Workspace memberships (distributed by workspace_id)
CREATE TABLE workspace_members (
    workspace_id    ULID NOT NULL REFERENCES workspaces(id),
    user_id         ULID NOT NULL REFERENCES users(id),
    role            TEXT NOT NULL DEFAULT 'member',  -- owner | admin | member | guest
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, user_id)
);

-- Channels (distributed by workspace_id)
CREATE TABLE channels (
    workspace_id    ULID NOT NULL REFERENCES workspaces(id),
    id              ULID NOT NULL,
    name            TEXT NOT NULL,
    kind            TEXT NOT NULL DEFAULT 'public',  -- public | private | dm
    topic           TEXT,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, id),
    UNIQUE (workspace_id, name)
);

-- Channel memberships (distributed by workspace_id)
CREATE TABLE channel_members (
    workspace_id    ULID NOT NULL,
    channel_id      ULID NOT NULL,
    user_id         ULID NOT NULL,
    last_read_at    TIMESTAMPTZ,
    muted           BOOLEAN NOT NULL DEFAULT FALSE,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, channel_id, user_id),
    FOREIGN KEY (workspace_id, channel_id) REFERENCES channels(workspace_id, id)
);

-- Messages (distributed by workspace_id)
CREATE TABLE messages (
    workspace_id    ULID NOT NULL,
    channel_id      ULID NOT NULL,
    id              ULID NOT NULL,
    author_id       ULID NOT NULL,
    content         TEXT NOT NULL,
    thread_id       ULID,               -- NULL if top-level, points to parent message id
    edited_at       TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ,
    metadata        JSONB DEFAULT '{}',  -- attachments, embeds, reactions summary
    search_vector   TSVECTOR,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, id),
    FOREIGN KEY (workspace_id, channel_id) REFERENCES channels(workspace_id, id)
);

-- Index for fetching messages in channel order (the most common query)
CREATE INDEX idx_messages_channel_timeline
    ON messages (workspace_id, channel_id, created_at DESC, id DESC)
    WHERE deleted_at IS NULL;

-- Full-text search index
CREATE INDEX idx_messages_search
    ON messages USING GIN (search_vector)
    WHERE deleted_at IS NULL;

-- Reactions (distributed by workspace_id)
CREATE TABLE reactions (
    workspace_id    ULID NOT NULL,
    message_id      ULID NOT NULL,
    user_id         ULID NOT NULL,
    emoji           TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, message_id, user_id, emoji),
    FOREIGN KEY (workspace_id, message_id) REFERENCES messages(workspace_id, id)
);

-- File attachments (distributed by workspace_id)
CREATE TABLE attachments (
    workspace_id    ULID NOT NULL,
    id              ULID NOT NULL,
    message_id      ULID,
    uploader_id     ULID NOT NULL,
    filename        TEXT NOT NULL,
    content_type    TEXT NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,       -- S3/MinIO object key
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (workspace_id, id),
    FOREIGN KEY (workspace_id, message_id) REFERENCES messages(workspace_id, id)
);
```

### 3.6 Full-Text Search

PostgreSQL's built-in full-text search via `tsvector` and `GIN` indexes handles message search without Elasticsearch for the majority of deployments. The `search_vector` column is populated via a trigger on insert/update:

```sql
CREATE FUNCTION update_message_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', COALESCE(NEW.content, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_message_search_vector
    BEFORE INSERT OR UPDATE OF content ON messages
    FOR EACH ROW EXECUTE FUNCTION update_message_search_vector();
```

**Scale-out search path:** When PostgreSQL FTS becomes a bottleneck (typically > 10M messages per workspace with heavy search traffic), introduce Meilisearch or Elasticsearch as a read replica fed by CDC (Change Data Capture) from PostgreSQL. The application code switches implementations behind the `SearchStore` trait.

### 3.7 Connection Pooling

PgBouncer sits between the application and PostgreSQL in all deployment modes:

- **Homelab:** Single PgBouncer instance, transaction-mode pooling, 20 connections.
- **SaaS:** PgBouncer per Citus node, with the application's workspace router directing connections to the appropriate pool.

---

## 4. Real-Time Messaging

### 4.1 Single-Node Mode (Homelab)

On a single node, real-time messaging uses in-process channels (Tokio broadcast channels). When a message is persisted to PostgreSQL, the service layer publishes it to an in-memory channel keyed by `(workspace_id, channel_id)`. All connected WebSocket tasks subscribed to that channel receive the event immediately.

No external message broker is needed. This keeps the homelab deployment to: one binary + PostgreSQL.

### 4.2 Multi-Node Mode (SaaS)

When running multiple API server instances behind a load balancer, in-process channels can't reach WebSocket connections on other nodes. NATS provides the fan-out layer:

1. Message is persisted to PostgreSQL.
2. Service layer publishes the event to NATS subject `workspace.{workspace_id}.channel.{channel_id}`.
3. Every API server instance subscribes to the subjects matching its connected clients.
4. Each instance forwards the event to the relevant local WebSocket connections.

NATS was chosen because it's a single binary with no dependencies (like Tanager itself), supports millions of subjects, and has first-class Rust client support (`async-nats`). It runs comfortably on 64 MB of RAM.

### 4.3 Feature Detection

The application detects which mode to use at startup:

```
if nats_url is configured → use NATS pub/sub
else → use in-process Tokio broadcast channels
```

Both implementations satisfy the same `PubSub` trait. The service layer doesn't know or care which is active.

---

## 5. File Storage

All file uploads go through an S3-compatible API:

| Deployment | Backend |
|-----------|---------|
| Homelab | MinIO (single-node) or local filesystem via S3 gateway |
| SaaS | AWS S3, Cloudflare R2, or any S3-compatible store |

Files are stored with the key pattern: `{workspace_id}/{channel_id}/{message_id}/{filename}`.

Presigned URLs are used for both uploads and downloads, keeping file bytes off the API server entirely.

---

## 6. Authentication & Session Management

### 6.1 Built-in Auth

- Email/password with Argon2id hashing.
- Session tokens stored in PostgreSQL with configurable TTL.
- TOTP-based 2FA.

### 6.2 External Identity Providers

- OAuth 2.0 / OpenID Connect support for Google, GitHub, GitLab, and generic OIDC providers.
- SAML 2.0 for enterprise SSO (SaaS paid tier).
- Self-hosters can point at their own Keycloak, Authentik, or any OIDC-compliant IdP.

### 6.3 API Authentication

- Bearer tokens for REST API calls.
- Token-based auth for WebSocket upgrade requests.
- Bot tokens with scoped permissions for integrations.

---

## 7. Deployment Modes

### 7.1 Homelab (Single Binary)

```bash
# Minimum viable deployment
tanager serve --database-url postgres://localhost/tanager
```

Everything runs in one process: HTTP server, WebSocket hub, background workers (search indexing, file cleanup). The only external dependency is PostgreSQL.

Optional: pass `--nats-url`, `--s3-endpoint`, `--redis-url` to enable distributed features as the user adds infrastructure.

### 7.2 SaaS (Multi-Tenant)

```
Load Balancer (Cloudflare / nginx)
    ├── API Server × N (stateless Axum instances)
    ├── NATS cluster (3 nodes)
    ├── PostgreSQL + Citus (coordinator + workers)
    ├── PgBouncer (per database node)
    ├── S3 (Cloudflare R2)
    └── Background Workers × N (search indexing, notifications, cleanup)
```

All API servers are stateless. Horizontal scaling is achieved by adding more API server instances and more Citus worker nodes.

---

## 8. Technology Choices

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Rust | Memory safety, single-binary deploys, low resource usage |
| HTTP Framework | Axum | Tokio-native, tower middleware ecosystem, strong typing |
| Database | PostgreSQL + Citus | Proven reliability, horizontal scaling without app changes |
| Connection Pool | PgBouncer | Reduces connection overhead, essential for multi-tenant |
| Message Broker | NATS (optional) | Single binary, lightweight, Rust-native client |
| File Storage | S3-compatible | Universal interface, works with MinIO (homelab) or cloud |
| Search | PostgreSQL FTS → Meilisearch | Built-in first, dedicated engine when needed |
| Cache | In-process → Valkey | Start with no external cache, add when needed |
| ID Generation | ULID | Sortable, timestamp-embedded, no coordination needed |
| Password Hashing | Argon2id | Current best practice, memory-hard |

---

## 9. Future Considerations (Not in MVP)

These are not implemented in Phase 1 but the architecture must not prevent them:

- **End-to-end encryption** for DMs (vodozemac library, same as Matrix).
- **Federation** between self-hosted instances via ActivityPub or a custom protocol.
- **Bridges** to Telegram, WhatsApp, Discord, IRC.
- **Voice/video** via LiveKit or Jitsi integration.
- **Read replicas** for heavy-read workspaces.
- **CDC pipeline** (PostgreSQL logical replication) feeding search indexes and analytics.

---

## Appendix A: Why Not These Alternatives?

| Alternative | Why not (for now) |
|------------|-------------------|
| MySQL/MariaDB | Weaker JSONB support, no native FTS comparable to tsvector, Citus has no MySQL equivalent |
| MongoDB | Document model doesn't fit relational workspace/channel/message hierarchy cleanly |
| ScyllaDB/Cassandra | Excellent for messages at Discord-scale, but adds operational complexity too early. Can be introduced behind the `MessageStore` trait later |
| CockroachDB | Distributed SQL but higher latency for single-node, heavier resource footprint than PostgreSQL + Citus |
| SQLite | No concurrent write support for multi-user, no horizontal scaling path |
| Kafka | Overkill for pub/sub at this stage, NATS is simpler and lighter |
