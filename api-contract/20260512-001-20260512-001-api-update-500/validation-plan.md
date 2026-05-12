# Validation Plan

## Overview

This validation plan covers the automated verification mechanisms for fixing the PUT `/api/admin/apis/{id}` returning 500 error issue.

## Unit Tests

### repo.rs - update function boundary cases

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Only environment_id provided | `environment_id=Some(uuid)`, all others `None` | Only `environment_id` column updated |
| All nullable fields set to None | All parameters `None` | No columns updated, returns existing record |
| auth_config = serde_json::Value::Null | `auth_config=Some(Value::Null)` | `auth_config` column set to SQL NULL |
| auth_config = serde_json::Value::Object({}) | `auth_config=Some(Value::Object({}))` | `auth_config` column set to `{}` |

### service.rs - update_api validation

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| environment_id changed to non-existent | Valid UUID not in environments table | Returns `AppError::BadRequest` with foreign key message |
| Locked API update attempt | API with active skill dependencies | Returns `AppError::Locked` with skill names |

## Integration Tests

### PUT /api/admin/apis/{id} scenarios

1. **Happy path - single field update**
   - Request: `PUT /api/admin/apis/{id}` with `{ "api_alias": "new_alias" }`
   - Expected: 200 OK, `needs_retest: false`

2. **Happy path - request-relevant field update**
   - Request: `PUT /api/admin/apis/{id}` with `{ "path": "/v2/new" }`
   - Expected: 200 OK, `needs_retest: true`

3. **Update non-existent API**
   - Request: `PUT /api/admin/apis/{non-existent-uuid}` with `{ "api_alias": "test" }`
   - Expected: 404 Not Found

4. **Update with locked API**
   - Setup: Create API with active skill dependency
   - Request: `PUT /api/admin/apis/{id}` with `{ "path": "/v2/test" }`
   - Expected: 423 Locked

## Static Analysis

- **clippy**: Run `cargo clippy -- -D warnings` to ensure no type safety issues
- **sqlx check**: Run `cargo sqlx --prepare` to validate all SQL queries at compile time

## Runtime Checks

- **Logging**: In development mode, log the actual SQL query and parameters on failure
- **Error response**: Ensure 500 errors include a correlation ID for log tracing
