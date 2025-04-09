# 🗃️ Keepr - Versioned Artifact Storage Service – Plan & Architecture
### Companion service: **Checkr** (external validator)


## 🎯 Purpose

Backend-only system to manage **versioned, typed, and validated artifacts** with:

- Semantic types (e.g. prompt, chain, plugin)
- External validator support (Checkr)
- Long-term version history + rollback
- Group-based access control (via JWT)
- REST API
- YAML-based artifact type configuration with secrets via Vault

---


## ✅ Core Features

### 📦 Artifact Fields

| Field               | Description                                      |
|--------------------|--------------------------------------------------|
| `id`               | ULID or UUID                                     |
| `name`             | Semantic artifact name                           |
| `type`             | Type (e.g. `prompt`, `frontend_plugin`, `chain`) |
| `version`          | Integer or SemVer string                         |
| `content`          | BYTEA or JSONB (depending on type)               |
| `metadata`         | JSONB: `tags`, `category`, `description`, etc.   |
| `status`           | One of: `draft`, `on_verification`, `published`, `archived`, `blocked` |
| `group_id`         | JWT-extracted access control scope               |
| `created_at`       | Timestamp                                         |
| `updated_at`       | Timestamp                                         |
| `last_accessed_at` | Timestamp for TTL cleanup logic                  |
| `content_hash`     | SHA256 hash for integrity/deduplication          |
| `usage_policy`     | `internal_only` / `public` / `restricted`        |

---

### 🔄 Versioning & Lifecycle

- `draft`: editable in-place
- `on_verification`: must pass external validation before publishing
- `published`: safe and ready for use
- `blocked`: failed validation → unusable
- `archived`: historical and immutable
  
Other rules:
- Unique constraint: `(name, type, version)`
- Rollback moves published → history and restores previous
- `artifact_history` table stores full copies with `original_id` backref

---

### 🔐 Access Control

- All access is scoped by `user_id`, `group_id`, injected from JWT
- No `user_id`,`group_id` accepted from client input
- All queries auto-filtered by `group_id`
- Access rules apply at every DB + API interaction level

---

### 🌐 Validation

- Defined in `artifact_type_config.yaml`
- Configurable per type
- Validations enforced **before** publishing the artifact
- Secrets (tokens, headers) injected from Vault or Kubernetes secrets

#### Example:
```yaml
types:
  prompt:
    requires_validation: false

  python_script:
    requires_validation: true
    validator:
      url: https://validator.example.com/python-lint
      method: POST
      headers:
        Authorization: "Bearer {{PYTHON_VALIDATOR_TOKEN}}"

  sast_plugin:
    requires_validation: true
    validator:
      url: https://secure-validator.internal/sast-check
      method: POST
```
---

## 📚 Schema (PostgreSQL)

```sql
-- Table: artifact_active
id UUID PRIMARY KEY,
name TEXT NOT NULL,
type TEXT NOT NULL,
version TEXT NOT NULL,
group_id TEXT NOT NULL,
status TEXT CHECK (status IN ('draft', 'on_verification', 'published', 'archived', 'blocked')),
content BYTEA,
json_content JSONB,
metadata JSONB,
tags JSONB,
category TEXT,
description TEXT,
created_at TIMESTAMPTZ,
updated_at TIMESTAMPTZ,
last_accessed_at TIMESTAMPTZ,
content_hash TEXT,
usage_policy TEXT

-- Table: artifact_history (same fields + original_id UUID)

-- Table: artifact_events
id UUID PRIMARY KEY,
artifact_id UUID,
event_type TEXT,       -- "created", "published", "blocked", etc.
event_data JSONB,
created_at TIMESTAMPTZ
```

## 🛠️ REST API

| Method | Endpoint                                               | Description                                                 |
|--------|--------------------------------------------------------|-------------------------------------------------------------|
| GET    | `/artifacts/`                                          | List artifacts for user's group (with filters + pagination) |
| GET    | `/artifacts/{name}`                                    | Get **latest published** version                            |
| GET    | `/artifacts/{name}/latest`                             | Get latest version (any status)                             |
| GET    | `/artifacts/{name}/{version}`                          | Get specific version                                        |
| GET    | `/artifacts/{name}/versions`                           | List all versions                                           |
| POST   | `/artifacts/`                                          | Submit new artifact (triggers validation)                   |
| PATCH  | `/artifacts/{name}/{version}/meta`                     | Update metadata for version                                 |
| POST   | `/artifacts/{name}/{version}/publish`                  | Publish version                                             |
| POST   | `/artifacts/{name}/{version}/unpublish`                | Unpublish version                                           |
| POST   | `/artifacts/{name}/{version}/archive`                  | Archive version                                             |
| DELETE | `/artifacts/{name}/{version}`                          | Soft-delete version                                         |
| POST   | `/artifacts/{name}/{version}/rollback`                 | Rollback to previous version                                |
| GET    | `/artifacts/{name}/{version}/diff?against=1.2.3`       | Show diff between two versions                              |
| POST   | `/artifacts/cleanup`                                   | Suggest artifacts for cleanup based on TTL                  |

### 🔍 Query Parameters for `GET /artifacts/`

| Parameter | Description                                               |
|-----------|-----------------------------------------------------------|
| `type`    | Filter by artifact type (e.g., `plugin`, `prompt`)        |
| `status`  | Filter by lifecycle status (e.g., `published`)            |
| `tags`    | Comma-separated tag filter (e.g., `core,ai`)              |
| `page`    | Page number for pagination                                |
| `limit`   | Page size (number of items to return, e.g., `20`)         |

## 🔧 Tech Stack
| Layer       | Stack                                         |
|-------------|-----------------------------------------------|
| API         | Python + FastAPI                              |
| DB          | PostgreSQL (JSONB, GIN, UUID/ULID)            |
| ORM         | SQLAlchemy or SQLModel                        |
| Migrations  | Alembic                                       |
| Auth        | JWT (group_id injected via dependency)        |
| Validation  | External API (config via YAML + Helm/Vault)   |
| Diffing     | `deepdiff`, `difflib`                         |
| Secrets     | Vault / K8s secrets                           |
| Deployment  | Docker + Helm                                 |

## 📡 Webhooks (Optional)
Trigger on: published, blocked, rollback
Send payload to pre-configured system
Use background tasks or Celery

## 🚫 Usage Constraints
Only published artifacts are executable or queryable by consumers
blocked, on_verification artifacts are not usable even if accessible
Use status + usage_policy to enforce protection layers

## 🧭 Roadmap Features (Not in First Version)
🔗 Artifact Relationships (dependencies)
🪪 Per-artifact access control (ACL/RBAC)
🧠 Graph-based dependency visualization
🪄 Template-based artifact creation
📊 Usage statistics + access logs
