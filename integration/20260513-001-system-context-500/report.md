# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260513-001 |
| **Title** | GET /api/admin/system-context-fields returns 500 Internal Server Error |
| **Type** | integration — Will be selected in Step 4 |
| **Severity** | high |
| **Status** | open |
| **Analyzed At** | 2026-05-13T00:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

Calling `GET /api/admin/system-context-fields?page=1&per_page=20` returns HTTP 500 with no body.

### 1.2 Reproduction Steps

1. Start the admin-backend server
2. Send GET request: `GET http://localhost:5174/api/admin/system-context-fields?page=1&per_page=20`
3. Observe: `HTTP/1.1 500 Internal Server Error 80ms`

### 1.3 Expected Behavior

Returns paginated list of system context fields with JSON body:
```json
{
  "fields": [...],
  "total": 0,
  "page": 1,
  "per_page": 20
}
```

### 1.4 Actual Behavior

HTTP 500 Internal Server Error with no response body.

---

## 2. Root Cause Analysis

### 2.1 Implementation-Level Cause

**Root cause: Database table `system_context_fields` does not exist at runtime.**

Code path:
1. `routes.rs:36` `list_fields_handler` → `service::list_fields()`
2. `service.rs:85` `list_fields()` → `repo::find_all()`
3. `repo.rs:7` `find_all()` → `sqlx::query_as()` with SQL `SELECT COUNT(*) FROM system_context_fields ...`

The query uses parameterized SQL with `sqlx::query_as::<_, SystemContextField>()` which requires the table to exist and have the correct schema. If the table is missing, `sqlx` will return a `sqlx::Error::Database` that gets converted to `AppError::Database("A database error occurred")` (generic message, no detail leaked) and the endpoint returns HTTP 500.

**Why no migration ran:** The `0014_create_system_context_fields.sql` migration exists but was never applied to the target database. This could be because:
- The database migrations were not run after the feature was added
- The migration was added but `sqlx migrate run` was not executed

### 2.2 Workflow-Level Cause

**Missing verification step in the deployment pipeline.**

The workflow for adding a new feature module (system_context) did not include:
1. A verification step to ensure migrations have been applied before the feature is considered "done"
2. An automated check that the feature's database table exists

The migration file `0014_create_system_context_fields.sql` was created and committed, but no mechanism ensured it was applied to the target database.

### 2.3 Code Graph

```
client
  └─> GET /api/admin/system-context-fields?page=1&per_page=20
        └─> Router (system_context_router)
              └─> list_fields_handler (routes.rs:36)
                    └─> service::list_fields(pool, query) (service.rs:85)
                          └─> repo::find_all(pool, page, per_page, ...) (repo.rs:7)
                                └─> sqlx::query_as + COUNT + SELECT (sqlx)
                                      └─ [500 if table missing]
```

---

## 3. Fix and Validation

### 3.1 Immediate Fix

Run the pending migration against the target database:

```bash
cd admin-backend
sqlx migrate run
```

Or manually apply the migration:

```sql
CREATE TABLE IF NOT EXISTS system_context_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_name VARCHAR(100) NOT NULL UNIQUE,
    field_type VARCHAR(20) NOT NULL DEFAULT 'string',
    description TEXT,
    options JSONB,
    reference_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_system_context_fields_field_name
    ON system_context_fields (field_name);
```

### 3.2 Automated Validation Mechanism

To prevent recurrence, add a startup check in `admin-backend/src/main.rs` that verifies all required tables exist before accepting traffic:

```rust
async fn verify_database_schema(pool: &PgPool) -> Result<(), Box<dyn std::error::Error>> {
    let required_tables = ["system_context_fields", "users", "api_definitions"];
    for table in required_tables {
        let exists = sqlx::query_scalar::<_, bool>(
            "SELECT EXISTS(SELECT FROM information_schema.tables WHERE table_name = $1)"
        )
        .bind(table)
        .fetch_one(pool)
        .await?;
        if !exists {
            return Err(format!("Required table '{}' is missing. Run migrations first.", table).into());
        }
    }
    Ok(())
}
```

Call `verify_database_schema(&pool).await?` during startup before the server begins accepting requests.

### 3.3 Workflow Change Proposal

1. **Add migration verification to CI:** In the admin-backend CI pipeline, add a step that runs `sqlx migrate run` and verifies the migration succeeds.
2. **Add health check for schema:** Add a `/health/schema` endpoint that verifies all expected tables exist, so deployment automation can confirm the database is properly set up.
3. **Document migration requirement:** In the feature development guide, add a note that new feature modules with database tables must include migration verification in the PR checklist.

---

*Initial document created in Step 1. Root cause analysis appended in Step 3. Fix and validation appended in Step 4.*