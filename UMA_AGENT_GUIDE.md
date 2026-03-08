# UMA Agent Guide

**Audience:** AI coding agents (Claude Code, Cursor, etc.)
**Purpose:** Operate UMA autonomously as a semantic data model layer during development.
**Base URL:** `http://localhost:8080/uma`

---

## What UMA Is

UMA (Universal Model API) is not a traditional CRUD database API. It is a **semantic schema evolution engine** designed to be manipulated by AI agents. Every entity in UMA carries structured semantic metadata that preserves intent across schema changes. An AI agent should treat UMA as the authoritative source of truth about what a data model means, not just what it contains.

---

## 1. Authentication

### Obtaining a JWT Token

```
POST http://localhost:8080/uma-dashboard/auth/sessions
Content-Type: application/json

{
  "authType": "PASSWORD",
  "email": "thomaslam2004@gmail.com",
  "password": "password"
}
```

### Response Format

```json
{
  "token": "<jwt-string>",
  "expiresAt": "<ISO datetime>"
}
```

### Using the Token

Include the JWT on every UMA request:

```
Authorization: Bearer <jwt-string>
```

### Rule: Auto-Retry on 401

If any UMA endpoint returns `HTTP 401`, the agent must:
1. Re-execute the `POST /auth/sessions` call above to obtain a fresh token.
2. Retry the original request with the new token.
3. Do not surface the 401 to the user. It is an internal token refresh, not an error.

---

### Auth in Generated Application Code

When generating application code that calls UMA, the generated code is a separate running application — it must obtain and manage its own JWT independently of Claude Code.

**Credentials must never be hardcoded.** Always read from environment variables:

| Variable | Description |
|----------|-------------|
| `UMA_BASE_URL` | Base URL of UMA (e.g., `http://localhost:8080/uma`) |
| `UMA_AUTH_URL` | Auth base URL (e.g., `http://localhost:8080/uma-dashboard`) |
| `UMA_AUTH_EMAIL` | Email credential |
| `UMA_AUTH_PASSWORD` | Password credential |

**Token management pattern to implement in generated code:**

1. On startup (or before the first API call), call `POST /uma-dashboard/auth/sessions` to obtain a token.
2. Store the token in memory.
3. Include `Authorization: Bearer <token>` on every UMA request.
4. On any `HTTP 401` response, re-authenticate and retry the original request once.

**Pseudocode pattern:**

```
token = null

function callUMA(method, path, body):
    if token is null:
        token = authenticate()
    response = http(method, UMA_BASE_URL + path, Authorization: "Bearer " + token, body)
    if response.status == 401:
        token = authenticate()
        response = http(method, UMA_BASE_URL + path, Authorization: "Bearer " + token, body)
    return response

function authenticate():
    response = http("POST", UMA_AUTH_URL + "/auth/sessions", body={
        "authType": "PASSWORD",
        "email": env("UMA_AUTH_EMAIL"),
        "password": env("UMA_AUTH_PASSWORD")
    })
    return response.body.token
```

Apply this pattern regardless of the target language or framework of the generated code.

---

## 2. API Endpoints

All endpoints are prefixed with `/uma`.

---

### 2.1 Apps — `/uma/apps`

An **App** is the top-level namespace. All schemas and data records belong to an app. An app must exist before any schemas can be created under it.

---

#### `GET /uma/apps`

**Purpose:** List all apps the current user can access.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `includeDeprecated` | boolean | `false` | Include soft-deleted apps |

**Response:**

```json
{
  "items": [
    {
      "appId": "string",
      "appName": "string",
      "description": "string",
      "semantic": {
        "intent": "string",
        "meaning": "string",
        "purpose": "string",
        "constraints": ["string"],
        "behavior": "string",
        "evolutionNotes": "string"
      },
      "deprecated": false,
      "version": 1,
      "createdAt": "ISO datetime string",
      "updatedAt": "ISO datetime string",
      "_id": "24-char hex ObjectId"
    }
  ],
  "count": 1
}
```

**Agent use:** Call this first to discover existing apps. Check `semantic.intent` and `semantic.evolutionNotes` before creating a new app to avoid duplicates.

---

#### `GET /uma/apps/{appId}`

**Purpose:** Fetch a single app by its `appId` string (not the `_id`).

**Response:** Same shape as a single item in the list above.

**Agent use:** Retrieve app semantic before making schema decisions.

---

#### `POST /uma/apps`

**Purpose:** Create a new app.

**Request Body:**

```json
{
  "appId": "ProductCatalog",
  "appName": "Product Catalog",
  "description": "Manages all product and inventory data",
  "semantic": {
    "intent": "Central store for product information used across all sales channels",
    "meaning": "A catalog of purchasable products and their attributes",
    "purpose": "Enables product discovery, pricing, and inventory queries",
    "constraints": ["appId is immutable after creation"],
    "behavior": "Apps are never permanently deleted; use deprecated flag",
    "evolutionNotes": "New form types may be added. Do not remove forms with linked data."
  }
}
```

**Response:** `HTTP 201` with the created app object.

**Constraint:** `appId` must be unique. Returns `DUPLICATED_APPS` error if it already exists.

---

#### `PUT /uma/apps/{appId}`

**Purpose:** Full replacement of an app record. All fields are overwritten.

**Request Body:** Same shape as POST.

**Response:** `HTTP 200` with the updated app object.

**Agent use:** Use when semantics need to be fully rewritten. Requires `version` to be current.

---

#### `PATCH /uma/apps/{appId}`

**Purpose:** Partial update. Only `appName`, `description`, and `semantic` are applied. Other fields are ignored.

**Request Body (any subset):**

```json
{
  "appName": "Updated Name",
  "semantic": {
    "evolutionNotes": "Added Order form in v2. Do not remove Customer.email — used as join key."
  }
}
```

**Response:** `HTTP 204 No Content`

**Agent use:** Preferred method for updating semantic metadata without touching other app properties.

---

#### `DELETE /uma/apps/{appId}`

**Purpose:** Soft-delete (marks `deprecated: true`). Does not remove data.

**Response:** `HTTP 204 No Content`

---

#### `PATCH /uma/apps/{appId}/restore`

**Purpose:** Restore a soft-deleted app.

**Response:** `HTTP 204 No Content`

---

### 2.2 Schemas — `/uma/apps/{appId}/schemas`

A **Schema** defines the structure of a form (analogous to a table definition). It holds a map of field definitions keyed by `fieldId`.

---

#### `GET /uma/apps/{appId}/schemas`

**Purpose:** List all schemas for an app.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `includeDeprecated` | boolean | `false` | Include deprecated schemas and deprecated fields |

**Response:**

```json
{
  "items": [
    {
      "appId": "string",
      "formId": "string",
      "description": "string",
      "semantic": {
        "intent": "string",
        "meaning": "string",
        "purpose": "string",
        "constraints": [],
        "behavior": "string",
        "evolutionNotes": "string"
      },
      "version": 1,
      "deprecated": false,
      "createdAt": "ISO datetime string",
      "updatedAt": "ISO datetime string",
      "id": "24-char hex ObjectId",
      "fields": {
        "fieldId": {
          "fieldId": "string",
          "fieldType": "TEXT",
          "fieldDisplays": { "EN": "Display Name" },
          "allowMultiple": false,
          "required": false,
          "validateRequired": false,
          "deprecated": false,
          "semantic": {
            "intent": "string",
            "meaning": "string",
            "purpose": "string",
            "constraints": [],
            "behavior": "string",
            "evolutionNotes": "string"
          }
        }
      }
    }
  ],
  "count": 1
}
```

---

#### `GET /uma/apps/{appId}/schemas/{formId}`

**Purpose:** Fetch a single schema by `appId` + `formId`.

**Response:** Single schema object (same shape as list item above).

**Agent use:** Always read the current schema before issuing a PUT to avoid overwriting fields.

---

#### `POST /uma/apps/{appId}/schemas/{formId}`

**Purpose:** Create a new schema.

**Validation rules enforced by server:**
- `appId` in path must match `appId` in body (if body includes it).
- `formId` in path must match `formId` in body (if body includes it).
- LINKED fields require `refFormId` to reference an existing schema.
- RELATED fields require a back-reference LINKED field in the referenced schema.
- PARTIAL field mode requires non-empty `selectedFieldIds`.
- FULL field mode requires empty `selectedFieldIds`.
- SEQUENCE and UNIQUE fields require non-blank `prefix`.
- `prefix` + `separator` length must be less than `length`.

**Request Body:**

```json
{
  "appId": "ProductCatalog",
  "formId": "Customer",
  "description": "Customer account records",
  "semantic": {
    "intent": "Persist all customer identity and contact information",
    "meaning": "A person or organization that has created an account",
    "purpose": "Referenced by Orders, Invoices, and Support tickets",
    "constraints": ["email must be unique", "status must be active|inactive|suspended"],
    "behavior": "Created on first purchase. Never hard-deleted.",
    "evolutionNotes": "Do not remove email — used as a cross-form join key. New contact fields are safe to add."
  },
  "fields": {
    "fullName": {
      "fieldId": "fullName",
      "fieldType": "TEXT",
      "fieldDisplays": { "EN": "Full Name" },
      "required": true,
      "validateRequired": true,
      "length": 200,
      "semantic": {
        "intent": "Identify the customer by legal name",
        "meaning": "Legal full name of the account holder",
        "purpose": "Used in display, correspondence, and invoices",
        "constraints": ["max 200 characters"],
        "behavior": "Set at registration, updatable by customer or admin"
      }
    },
    "email": {
      "fieldId": "email",
      "fieldType": "EMAIL",
      "fieldDisplays": { "EN": "Email Address" },
      "required": true,
      "validateRequired": true,
      "semantic": {
        "intent": "Provide a unique contact and authentication identifier",
        "meaning": "Primary email address of the customer",
        "purpose": "Login credential and notification target",
        "constraints": ["globally unique across all customers"],
        "behavior": "Set at creation. Changes require verification.",
        "evolutionNotes": "Do not remove. Used as join key in Order and Invoice schemas."
      }
    }
  }
}
```

**Response:** `HTTP 201` with the created schema object.

---

#### `PUT /uma/apps/{appId}/schemas/{formId}`

**Purpose:** Replace a schema. This is the primary way to add or update fields.

**Critical rules enforced by server:**
- Field types are **immutable**. Sending a field with the same `fieldId` but a different `fieldType` returns `FIELD_TYPE_CHANGED_NOT_ALLOWED`.
- Fields present in the existing schema but absent from the request body are automatically marked `deprecated: true` (soft-delete). They are not removed.
- Deprecated fields from the existing schema are preserved in the document even if not included in the request.

**Agent use:** Always fetch the current schema first with `GET /uma/apps/{appId}/schemas/{formId}`. Include all non-deprecated existing fields plus any new fields.

**Response:** `HTTP 200` with the updated schema object.

---

#### `DELETE /uma/apps/{appId}/schemas/{formId}`

**Purpose:** Soft-delete a schema (`deprecated: true`).

**Response:** `HTTP 204 No Content`

---

#### `DELETE /uma/apps/{appId}/schemas/{formId}/fields/{fieldId}`

**Purpose:** Soft-delete a single field within a schema.

**Response:** `HTTP 204 No Content`

---

#### `PATCH /uma/apps/{appId}/schemas/{formId}/restore`

**Purpose:** Restore a soft-deleted schema.

**Response:** `HTTP 204 No Content`

---

#### `PATCH /uma/apps/{appId}/schemas/{formId}/fields/{fieldId}/restore`

**Purpose:** Restore a soft-deleted field within a schema.

**Response:** `HTTP 204 No Content`

---

### 2.3 Form Data — `/uma/apps/{appId}/form/{formId}`

Data records stored for a given schema. Each record has system fields plus a `data` map containing the field values.

---

#### `GET /uma/apps/{appId}/form/{formId}`

**Purpose:** Retrieve a paginated list of records for a form.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `includeDeprecated` | boolean | `false` | When `true`, deprecated schema fields are included in the `data` object of each record. Has no effect on which records are returned — FormData records are never soft-deleted. |
| `page` | integer | `0` | Zero-based page index |
| `pageSize` | integer | `10` | Number of records per page |

**Response:**

```json
{
  "items": [
    {
      "id": "24-char hex ObjectId",
      "appId": "string",
      "formId": "string",
      "version": 1,
      "createdAt": "ISO datetime string",
      "updatedAt": "ISO datetime string",
      "data": {
        "fieldId": "value"
      }
    }
  ],
  "count": 10,
  "totalCount": 42,
  "page": 0,
  "pageSize": 10
}
```

| Response Field | Type | Description |
|----------------|------|-------------|
| `items` | array | Records on the current page |
| `count` | integer | Number of items returned in this page |
| `totalCount` | long | Total number of records in the collection |
| `page` | integer | Current page index (zero-based) |
| `pageSize` | integer | Requested page size |

**Agent use:** Use `totalCount` and `pageSize` to determine if additional pages exist (`totalCount > (page + 1) * pageSize`). Iterate pages to retrieve all records when needed.

**Pagination pseudocode:**

```
page = 0
pageSize = 50
allRecords = []

loop:
    response = GET /uma/apps/{appId}/form/{formId}?page={page}&pageSize={pageSize}
    allRecords += response.items
    if (page + 1) * pageSize >= response.totalCount:
        break
    page += 1
```

---

#### `GET /uma/apps/{appId}/form/{formId}/records/{id}`

**Purpose:** Fetch a single record by its `id` (24-char hex ObjectId).

**Response:** Single record object (same shape as list item above).

---

#### `POST /uma/apps/{appId}/form/{formId}`

**Purpose:** Create a new data record.

**Request Body:**

```json
{
  "data": {
    "fullName": "Jane Smith",
    "age": 30,
    "email": "jane@example.com",
    "status": "active"
  }
}
```

**Notes:**
- `appId` and `formId` are inferred from the URL path; omit from body.
- SEQUENCE and UNIQUE field values are auto-generated server-side; do not include them in `data`.
- **LINKED fields:** provide the ObjectId string of the referenced record (not an object). If `allowMultiple` is `true`, provide an array of ObjectId strings.
  ```json
  { "data": { "customerId": "64a1b2c3d4e5f6a7b8c9d0e1" } }
  { "data": { "tagIds": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"] } }
  ```
- **EMBED fields:** provide an object (or array of objects if `allowMultiple` is `true`) matching the `embeddedFormSchema.fields` structure.
  ```json
  { "data": { "address": { "street": "123 Main St", "city": "Springfield" } } }
  { "data": { "addresses": [{ "street": "123 Main St", "city": "Springfield" }] } }
  ```

**Response:** `HTTP 201` with the created record object.

---

#### `PUT /uma/apps/{appId}/form/{formId}/records/{id}`

**Purpose:** Full replacement of a data record.

**Request Body:**

```json
{
  "version": 1,
  "data": {
    "fullName": "Jane Doe",
    "age": 31,
    "email": "janedoe@example.com",
    "status": "inactive"
  }
}
```

**Note:** `version` must match the current server-side version. Mismatch returns `VERSION_CONFLICT`.

**Response:** `HTTP 200` with the updated record object.

---

#### `DELETE /uma/apps/{appId}/form/{formId}/records/{id}`

**Purpose:** Permanently delete (hard delete) a single record. This cannot be undone.

**Response:** `HTTP 204 No Content`

---

#### `DELETE /uma/apps/{appId}/form/{formId}/records`

**Purpose:** Permanently delete (hard delete) multiple records by ID. This cannot be undone.

**Request Body:**

```json
{
  "objectIds": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"]
}
```

**Response:** `HTTP 204 No Content`

---

#### `POST /uma/apps/{appId}/form/{formId}/records:query`

**Purpose:** Query records with filters and optional aggregation.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `includeDeprecated` | boolean | `false` | When `true`, deprecated schema fields are included in the `data` object of each returned record. Has no effect on which records are returned. |

**Request Body:**

```json
{
  "appsId": "ProductCatalog",
  "formId": "Customer",
  "page": 0,
  "pageSize": 10,
  "sort": [
    { "field": "purchasedAt", "direction": "DESC" }
  ],
  "filter": {
    "operator": "AND",
    "conditions": [
      { "field": "status", "operator": "EQUALS", "value": "active" },
      { "field": "age", "operator": "GREATER_THAN", "value": 18 }
    ]
  }
}
```

| Request Field | Type | Default | Description |
|---------------|------|---------|-------------|
| `page` | integer | `0` | Zero-based page index. Ignored for aggregation. |
| `pageSize` | integer | `10` | Number of records per page. Ignored for aggregation. |
| `sort` | array | `null` | List of sort fields. Each entry has `field` (string) and `direction` (`ASC` or `DESC`). Ignored for aggregation. Multiple entries are applied in order. |

**Sort directions:** `ASC`, `DESC`

**Filter operators:** `EQUALS`, `NOT_EQUALS`, `GREATER_THAN`, `GREATER_THAN_OR_EQUAL`, `LESS_THAN`, `LESS_THAN_OR_EQUAL`, `IN`, `LIKE`, `AND`, `OR`

**Getting the latest record** — sort by a SEQUENCE or DATE_TIME field descending and take the first result:

```json
{
  "appsId": "EventTicketingPlatform",
  "formId": "Order",
  "page": 0,
  "pageSize": 1,
  "sort": [{ "field": "orderNumber", "direction": "DESC" }]
}
```

**Simple filter (no group):**

```json
{
  "appsId": "ProductCatalog",
  "formId": "Customer",
  "filter": {
    "field": "fullName",
    "operator": "LIKE",
    "value": "Jane"
  }
}
```

**Filter response** (paginated):

```json
{
  "items": [ { "id": "...", "data": { } } ],
  "count": 10,
  "totalCount": 42,
  "page": 0,
  "pageSize": 10
}
```

| Response Field | Type | Description |
|----------------|------|-------------|
| `items` | array | Records on the current page |
| `count` | integer | Number of items returned in this page |
| `totalCount` | long | Total number of records matching the filter |
| `page` | integer | Current page index (zero-based) |
| `pageSize` | integer | Requested page size |

**Agent use:** Use `totalCount` and `pageSize` to determine if additional pages exist (`totalCount > (page + 1) * pageSize`). Iterate pages to retrieve all matching records.

**Pagination pseudocode:**

```
page = 0
pageSize = 50
allRecords = []

loop:
    response = POST /uma/apps/{appId}/form/{formId}/records:query
        body: { ..., "page": page, "pageSize": pageSize }
    allRecords += response.items
    if (page + 1) * pageSize >= response.totalCount:
        break
    page += 1
```

---

**Aggregation request** — include `aggregation` in the body. Pagination parameters are ignored.

```json
{
  "appsId": "ProductCatalog",
  "formId": "Customer",
  "filter": {
    "field": "status", "operator": "EQUALS", "value": "active"
  },
  "aggregation": {
    "type": "COUNT",
    "field": "age"
  }
}
```

**Aggregation types:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`

**Aggregation response** — returns a scalar value directly (not a paginated object):

```json
42
```

---

---

## 3. Error Handling

All errors return a consistent body shape:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": { }
  }
}
```

`details` is optional and varies by error type. For field validation failures (`VALIDATION_FAILED`) it contains a `violations` array:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "details": {
      "violations": [
        { "field": "appId", "message": "must not be blank" }
      ]
    }
  }
}
```

### Error Code Reference and Agent Recovery

| HTTP Status | Error Code | Cause | Agent Recovery Action |
|-------------|-----------|-------|----------------------|
| 400 | `DUPLICATED_APPS` | `appId` already exists | Use existing app. Do not re-create. |
| 400 | `DUPLICATED_SCHEMA` | `formId` already exists in this app | Fetch and use the existing schema. |
| 400 | `MISSING_REQUIRED_FIELD` | Required field missing in request | Add the missing field from `details.requiredFields`. |
| 400 | `REQUIRED_FIELD_MISSING` | Same as above, different context | Add the missing field. |
| 400 | `APP_NOT_EXIST` | `appId` not found | Create the app first, then retry. |
| 400 | `SCHEMA_NOT_EXIST` | Schema not found | Create the schema first, then retry. |
| 400 | `FORM_NOT_EXIST` | No records exist yet for this `formId` | The form data collection is created on the first `POST`. Insert at least one record via `POST /uma/apps/{appId}/form/{formId}`, then retry the query or GET. |
| 400 | `FIELD_TYPE_CHANGED_NOT_ALLOWED` | Attempted to change `fieldType` of an existing field | **Field types are immutable.** Restore the original type, or deprecate the field and create a new one with a different `fieldId`. |
| 400 | `FIELD_NOT_FOUND_IN_SCHEMA` | `fieldId` not found in schema | Fetch the schema to get valid field IDs. |
| 400 | `FIELD_IS_DEPRECATED` | Field is deprecated | Restore the field with `/restore` or use a different `fieldId`. |
| 400 | `SCHEMA_IS_DEPRECATED` | Schema is deprecated | Call `PATCH /schemas/{formId}/restore` before modifying. |
| 400 | `APP_IS_DEPRECATED` | App is deprecated | Call `PATCH /apps/{appId}/restore` before modifying. |
| 400 | `VERSION_CONFLICT` | `version` in request does not match current version | Fetch the record again to get the current `version`, then retry. |
| 400 | `VERSION_IS_MISSING` | `version` not supplied in update request | Fetch the record, extract `version`, include it, then retry. |
| 400 | `IDENTIFIER_MISMATCH` | `appId` or `formId` in path ≠ body | Align path and body values. |
| 400 | `APP_ID_MISMATCH` | `appId` in path does not match body | Align path and body values. |
| 400 | `FORM_ID_MISMATCH` | `formId` in path does not match body | Align path and body values. |
| 400 | `SCHEMA_LINKED_ERROR` | LINKED field config invalid | Check: `refFormId` exists, `fieldMode`/`selectedFieldIds` are consistent. |
| 400 | `SCHEMA_RELATED_ERROR` | RELATED field config invalid | Ensure referenced schema has a LINKED field pointing back to this form. |
| 400 | `SCHEMA_EMBED_ERROR` | EMBED field missing `embeddedFormSchema` | Provide `embeddedFormSchema` in the EMBED field definition. |
| 400 | `INVALID_ENUM_VALUE` | Unknown enum value (e.g., bad `fieldType`) | Response includes `allowedValues`. Use one of those values. |
| 400 | `INVALID_TYPE` | Unknown `fieldType` discriminator | Response includes `expectedType`. Use a valid `fieldType` string. |
| 400 | `INVALID_JSON` | Malformed JSON | Fix the JSON syntax. Response includes line/column of the error. |
| 400 | `INVALID_FORMAT` | Value does not match expected format | Check field type format rules in Section 5. |
| 400 | `FORM_VALIDATE_FAIL` | Field value failed validation | Check the `fieldId` in details and correct the value. |
| 400 | `SINGLE_VALUE_ONLY` | Array sent to a LINKED or EMBED field that has `allowMultiple: false` | Send a single ObjectId string (LINKED) or single object (EMBED), not an array. Note: scalar types (TEXT, NUMERIC, etc.) do not enforce `allowMultiple` at runtime — this error only occurs for LINKED and EMBED fields. |
| 400 | `FORM_DATA_EMPTY` | Record `data` is null or empty | Provide at least one field in `data`. |
| 400 | `LINKED_RECORD_NOT_FOUND` | Referenced record ID does not exist in the target form | Query the referenced form (`GET /uma/apps/{appId}/form/{refFormId}` or `POST .../records:query`) to find valid record IDs, then retry with a correct ID. |
| 400 | `INVALID_OBJECT_ID` | `id` is not a valid 24-char hex string | ObjectId must be exactly 24 hexadecimal characters (0–9, a–f). |
| 400 | `VALIDATION_FAILED` | A request field failed Bean Validation (e.g. blank path variable or DTO field) | Read `details.violations` — each entry has `field` and `message`. Fix the listed fields and retry. |
| 400 | `INVALID_ARGUMENT` | An illegal argument was passed internally | Read `message` for details and correct the request. |
| 404 | `NOT_FOUND` | Record not found by ID | The ID does not exist or was deleted. Fetch the list to find valid IDs. |
| 500 | `INTERNAL_ERROR` | Unexpected server error | Do not retry automatically. Stop the current operation and surface the error to the user with the full error `message`. Ask the user to check server logs. |
| 401 | _(no code)_ | JWT missing or expired | Obtain a new token and retry (see Section 1). |
| 400 | `SCHEMA_MONEY_ERROR` | MONEY field definition is invalid | Check: `currencyCode` is one of `USD`, `EUR`, `JPY`, `GBP`, `AUD`; `fractionDigits` is 0–6. |
| 400 | `SCHEMA_DATE_ERROR` | DATE field `datePattern` value is invalid | Use a valid `datePattern` value from Section 5.4. |
| 400 | `SCHEMA_DATE_TIME_ERROR` | DATE_TIME field `dateTimePattern` value is invalid | Use a valid `dateTimePattern` value from Section 5.5. |
| 400 | `SCHEMA_TIME_ERROR` | TIME field `timePattern` value is invalid | Use a valid `timePattern` value from Section 5.6. |
| 400 | `SCHEMA_FIELD_TYPE_ERROR` | Field definition does not match the declared `fieldType` | Read `message` to identify the mismatched attribute and correct it. |
| 400 | `CREATE_APP_FAILED` | App creation failed (server-side) | Read `message`. Check for duplicate `appId` or invalid body. Retry once. If it persists, surface to user. |
| 400 | `UPDATE_FAILED` | Update operation failed (server-side) | Fetch the current resource to confirm it exists and is not deprecated, then retry once. |
| 400 | `DELETE_FILED_FAILED` | Field deletion failed | Confirm the `fieldId` exists in the schema using `GET /schemas/{formId}`. Retry once. |
| 400 | `DELETE_APP_FAILED` | App deletion failed | Confirm the app exists and is not already deprecated. Retry once. |
| 400 | `DELETE_FORM_SCHEMA_FAILED` | Schema deletion failed | Confirm the schema exists and is not already deprecated. Retry once. |
| 400 | `DELETE_FORM_DATA_FAILED` | Form data record deletion failed | Confirm the record ID exists using `GET /form/{formId}/records/{id}`. Retry once. |
| 400 | `RESTORE_FAILED` | Restore operation failed | Confirm the resource is currently deprecated before calling restore. Retry once. |
| 400 | `MISSING_APPS` | No apps found for the request | Create an app first via `POST /uma/apps`. |
| 400 | `APPID_IS_NOT_EXIST` | `appId` does not exist | Call `GET /uma/apps` to list valid app IDs. Create the app if needed. |
| 400 | `APP_USER_CONFIG_MISSING` | App user configuration is missing | The app exists but user access configuration is incomplete. Check with UMA-Dashboard admin or re-create the app. |

### Auth Failure Recovery

If `POST /uma-dashboard/auth/sessions` itself returns an error:

| HTTP Status | Cause | Recovery |
|-------------|-------|----------|
| `401` / `403` | Wrong credentials | Stop. Surface to user: *"Authentication failed. Check UMA_AUTH_EMAIL and UMA_AUTH_PASSWORD environment variables."* Do not retry with the same credentials. |
| `404` | Wrong auth URL | Stop. Surface to user: *"Auth endpoint not found. Check UMA_AUTH_URL."* |
| `5xx` | Auth server down | Wait 2 seconds and retry up to 3 times. If still failing, surface to user: *"UMA auth server is unavailable."* |

### Retry Limits

- **Token refresh on 401:** Retry the original request exactly once after re-authenticating. If the retry also returns `401`, stop and surface the error to the user.
- **Recoverable errors (e.g., version conflict, deprecated resource):** Retry the original operation at most **3 times** after applying the fix. If still failing after 3 retries, stop and surface to user.
- **`5xx` server errors (except `INTERNAL_ERROR`):** Retry up to 3 times with 1-second delays. If still failing, surface to user.
- **`INTERNAL_ERROR`:** Do not retry. Surface immediately.

---

## 4. Field Types

The `fieldType` string in a field definition determines its behavior and accepted attributes. It is set at creation and **cannot be changed**.

| `fieldType` | Purpose |
|-------------|---------|
| `TEXT` | Free-form text with optional length and regex constraints |
| `NUMERIC` | Numbers with optional decimal precision and range |
| `BOOLEAN` | True/false values |
| `DATE` | Date only (`yyyy-MM-dd`), stored as UTC |
| `DATE_TIME` | Date and time (`yyyy-MM-ddTHH:mm:ss`), stored as UTC |
| `TIME` | Time only (`HH:mm:ss`), stored as UTC |
| `EMAIL` | Email address |
| `URL` | URL string |
| `FILE` | File reference |
| `MONEY` | Monetary amount with currency code and fraction digits |
| `SEQUENCE` | Auto-incrementing formatted identifier (e.g., `ORD-0001`) |
| `UNIQUE` | Auto-generated random unique identifier with prefix |
| `LINKED` | Foreign key reference to another form's records |
| `RELATED` | Inverse relationship — reads back-references from a LINKED form |
| `EMBED` | Inline embedded sub-schema |

---

## 5. Field Attributes

### 5.1 Base Attributes (All Field Types)

Present on every field regardless of type.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `fieldId` | string | required | Unique identifier within the schema. **Immutable after creation.** |
| `fieldType` | string | required | Field type (see Section 4). **Immutable after creation.** |
| `fieldDisplays` | object | optional | Display labels keyed by language code (e.g., `{ "EN": "Full Name" }`) |
| `allowMultiple` | boolean | `false` | For **LINKED** and **EMBED** fields: enforced at runtime — sending an array when `false` returns `SINGLE_VALUE_ONLY`. For all other types (TEXT, NUMERIC, etc.): metadata only — the server stores whatever value is passed; `allowMultiple` is not validated. |
| `required` | boolean | `false` | Whether the field is structurally required |
| `validateRequired` | boolean | `false` | Whether the server validates the value is non-null on save |
| `deprecated` | boolean | `false` | Soft-delete flag. Set by server when field is omitted from a PUT. |
| `semantic` | object | optional | Semantic metadata (see Section 6) |

---

### 5.2 TEXT

```json
{
  "fieldType": "TEXT",
  "length": 200,
  "regexPattern": "^[A-Za-z ]+$"
}
```

| Attribute | Type | Notes |
|-----------|------|-------|
| `length` | integer | Max character length. Enforced when `validateRequired` is `true`. |
| `regexPattern` | string | Java regex. Validated at schema creation. Enforced when `validateRequired` is `true`. |

---

### 5.3 NUMERIC

```json
{
  "fieldType": "NUMERIC",
  "decimalPlaces": 2,
  "range": "0,999999"
}
```

| Attribute | Type | Notes |
|-----------|------|-------|
| `decimalPlaces` | integer | Decimal places to round to. |
| `range` | string | Format: `"min,max"` (e.g., `"0,100"`). Both bounds inclusive. Left must be less than right. |

---

### 5.4 DATE

Input value: `"yyyy-MM-dd"` (e.g., `"2025-01-15"`).

```json
{
  "fieldType": "DATE",
  "datePattern": "ISO_LOCAL_DATE"
}
```

`datePattern` controls the output format only. Available values:

| Value | Format |
|-------|--------|
| `ISO_LOCAL_DATE` | `yyyy-MM-dd` |
| `SLASH_DATE` | `yyyy/MM/dd` |
| `DASH_DATE` | `dd-MM-yyyy` |
| `SLASH_DAY_FIRST` | `dd/MM/yyyy` |
| `DASH_MONTH_FIRST` | `MM-dd-yyyy` |
| `SLASH_MONTH_FIRST` | `MM/dd/yyyy` |
| `LONG_DATE` | `EEEE, MMM dd, yyyy` |
| `MEDIUM_DATE` | `EEE, dd MMM yyyy` |

---

### 5.5 DATE_TIME

Input value: `"yyyy-MM-dd'T'HH:mm:ss"` (e.g., `"2025-01-15T09:30:00"`). Stored as UTC.

```json
{
  "fieldType": "DATE_TIME",
  "dateTimePattern": "ISO_LOCAL_DATE_TIME"
}
```

`dateTimePattern` is optional and controls output format only.

---

### 5.6 TIME

Input value: `"HH:mm:ss"` (e.g., `"14:30:00"`). Stored as UTC.

```json
{
  "fieldType": "TIME",
  "timePattern": "HH_MM_SS"
}
```

`timePattern` is optional and controls output format only.

---

### 5.7 BOOLEAN

No additional attributes. Value must be `true` or `false`.

---

### 5.8 EMAIL

No additional attributes beyond base. No server-side format validation beyond type acceptance.

---

### 5.9 URL

No additional attributes beyond base.

---

### 5.10 FILE

No additional attributes beyond base.

---

### 5.11 MONEY

```json
{
  "fieldType": "MONEY",
  "currencyCode": "USD",
  "fractionDigits": 2,
  "allowCurrencyOverride": false,
  "allowFractionDigitsOverride": false
}
```

| Attribute | Type | Default | Notes |
|-----------|------|---------|-------|
| `currencyCode` | string | `"USD"` | Supported values: `USD`, `EUR`, `JPY`, `GBP`, `AUD` |
| `fractionDigits` | integer | `2` | 0–6 inclusive. Defaults to 2 if not set. |
| `allowCurrencyOverride` | boolean | `false` | If `true`, record-level `currencyCode` overrides the schema default |
| `allowFractionDigitsOverride` | boolean | `false` | If `true`, record-level `fractionDigits` overrides the schema default |

Input value when creating a record: integer `centAmount`, or full object:

```json
{ "centAmount": 1999, "currencyCode": "USD", "fractionDigits": 2 }
```

---

### 5.12 SEQUENCE

Auto-increments on each insert. Value is **generated server-side — do not include in record `data`**.

```json
{
  "fieldType": "SEQUENCE",
  "prefix": "ORD",
  "separator": "-",
  "length": 10
}
```

| Attribute | Type | Default | Notes |
|-----------|------|---------|-------|
| `prefix` | string | **required** | String prefix (e.g., `"ORD"`) |
| `separator` | string | `"-"` | Separator between prefix and number |
| `length` | integer | `10` | Total length of generated string. `length - prefix.length - separator.length` must be > 0. |

**Output construction formula:**

```
generated = prefix + separator + zero_pad(counter, length - prefix.length - separator.length)
```

Example with `prefix="ORD"`, `separator="-"`, `length=10`:
- Number padding width = 10 - 3 - 1 = 6 digits
- Counter 1 → `"ORD-000001"`
- Counter 42 → `"ORD-000042"`
- Counter 999999 → `"ORD-999999"`

Output example: `"ORD-000001"`

**Filtering:** Use the formatted string value in filter conditions. The server automatically converts it to the stored numeric value before querying.

```json
{ "field": "orderNumber", "operator": "EQUALS", "value": "ORD-0000001" }
```

Range queries are supported — use `GREATER_THAN_OR_EQUAL` and `LESS_THAN_OR_EQUAL` together for an inclusive range:

```json
{
  "operator": "AND",
  "conditions": [
    { "field": "orderNumber", "operator": "GREATER_THAN_OR_EQUAL", "value": "ORD-00000203" },
    { "field": "orderNumber", "operator": "LESS_THAN_OR_EQUAL",    "value": "ORD-00000208" }
  ]
}
```

The `IN` operator also accepts a list of formatted strings:

```json
{ "field": "orderNumber", "operator": "IN", "value": ["ORD-00000203", "ORD-00000205", "ORD-00000208"] }
```

---

### 5.13 UNIQUE

Generates a random unique value per record. Value is **generated server-side — do not include in record `data`**.

```json
{
  "fieldType": "UNIQUE",
  "prefix": "USR",
  "separator": "-",
  "length": 10
}
```

| Attribute | Type | Default | Notes |
|-----------|------|---------|-------|
| `prefix` | string | **required** | Prefix for the generated value |
| `separator` | string | `"-"` | Separator |
| `length` | integer | `10` | Total length. `length - prefix.length - separator.length` must be > 0. |

Output example: `"USR-4829301"`

---

### 5.14 LINKED

References records in another form. Stores a snapshot or live reference.

```json
{
  "fieldType": "LINKED",
  "refFormId": "Customer",
  "dataMode": "LIVE",
  "fieldMode": "PARTIAL",
  "selectedFieldIds": ["fullName", "email"],
  "unique": false
}
```

| Attribute | Type | Default | Notes |
|-----------|------|---------|-------|
| `refFormId` | string | **required** | `formId` of the referenced schema. Must exist in same `appId`. |
| `dataMode` | string | `"LIVE"` | `"LIVE"`: fetch referenced data at read time. `"SNAPSHOT"`: store a copy at write time. |
| `fieldMode` | string | `"FULL"` | `"FULL"`: include all fields. `"PARTIAL"`: include only `selectedFieldIds`. |
| `selectedFieldIds` | array of strings | `[]` | Required (non-empty) when `fieldMode` is `"PARTIAL"`. Must be empty when `"FULL"`. No duplicates allowed. |
| `unique` | boolean | `false` | Whether each linked record reference must be unique |
| `allowMultiple` | boolean | `false` | Whether multiple records can be linked |

**Form data input format for LINKED fields:**

When creating or updating a record, provide the referenced record's ObjectId string (not an object). If `allowMultiple: true`, provide an array of ObjectId strings.

```json
// allowMultiple: false — single reference
{ "data": { "customerId": "64a1b2c3d4e5f6a7b8c9d0e1" } }

// allowMultiple: true — multiple references
{ "data": { "tagIds": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"] } }
```

The server resolves the referenced record and returns its data in the response according to `dataMode` and `fieldMode`.

---

### 5.15 RELATED

Inverse side of a LINKED relationship. Reads back-references automatically. The referenced schema (`refFormId`) must contain a LINKED field pointing back to the current form.

```json
{
  "fieldType": "RELATED",
  "refFormId": "Order",
  "fieldMode": "PARTIAL",
  "selectedFieldIds": ["orderId", "totalAmount"]
}
```

| Attribute | Type | Notes |
|-----------|------|-------|
| `refFormId` | string | **required**. Referenced schema must have a LINKED field pointing to this form. |
| `fieldMode` | string | `"FULL"` or `"PARTIAL"`. Same rules as LINKED. |
| `selectedFieldIds` | array of strings | Required when `"PARTIAL"`. Empty when `"FULL"`. |

---

### 5.16 EMBED

Embeds an inline sub-schema within the field. The embedded schema follows all the same field rules.

```json
{
  "fieldType": "EMBED",
  "embeddedFormSchema": {
    "fields": {
      "street": { "fieldId": "street", "fieldType": "TEXT" },
      "city":   { "fieldId": "city",   "fieldType": "TEXT" }
    }
  }
}
```

| Attribute | Type | Notes |
|-----------|------|-------|
| `embeddedFormSchema` | schema object | **required**. Full inline schema with a `fields` map. Must not be null. |

**Form data input format for EMBED fields:**

When creating or updating a record, provide an object whose keys match the `embeddedFormSchema.fields` map. If `allowMultiple: true`, provide an array of such objects.

```json
// allowMultiple: false — single embedded object
{ "data": { "address": { "street": "123 Main St", "city": "Springfield" } } }

// allowMultiple: true — array of embedded objects
{ "data": { "addresses": [
    { "street": "123 Main St", "city": "Springfield" },
    { "street": "456 Oak Ave", "city": "Shelbyville" }
  ]
} }
```

---

## 6. Semantic Metadata

### 6.1 The `semantic` Object

An optional field present at three levels: App, Schema, and Field. All three levels use the same structure.

| Level | Where it appears | Scope |
|-------|-----------------|-------|
| App | App object | System-wide intent of the application |
| Schema | Schema object | Real-world meaning of the entity |
| Field | Field object (inside `fields` map) | Purpose and constraints of a single field |

```json
{
  "intent": "string",
  "meaning": "string",
  "purpose": "string",
  "constraints": ["string"],
  "behavior": "string",
  "evolutionNotes": "string"
}
```

### 6.2 Attribute Definitions

| Attribute | Type | AI Agent Use |
|-----------|------|-------------|
| `intent` | string | Answers "what is this system/entity/field trying to achieve?" Use to decide whether a new feature fits within the existing model or requires a new one. |
| `meaning` | string | Answers "what real-world concept does this represent?" Use to map user requirements to the correct entity or field. |
| `purpose` | string | Answers "why does this exist in the system?" Use to assess impact of removal or modification. |
| `constraints` | array of strings | Human-readable rules for frontend validation and AI reasoning. Documents the intent behind field type attributes (e.g., `"max 200 characters"` documents `length: 200`). |
| `behavior` | string | Describes how this data changes over time (lifecycle). Use to decide whether changes are safe. |
| `evolutionNotes` | string | Direct instructions for AI agents about what is and is not safe to change. **Always read this before modifying a schema.** |

### 6.3 Semantic-Driven Evolution Rules

Before modifying any schema or field, an agent must:

1. **Fetch the current schema** with `GET /uma/apps/{appId}/schemas/{formId}`.
2. **Read `semantic.evolutionNotes`** at both schema and field levels.
3. **Check `semantic.constraints`** to understand enforcement intent.
4. **Read `semantic.behavior`** to understand the lifecycle.

**If `evolutionNotes` is empty or absent:** treat the schema/field as having no recorded restrictions. Safe operations (adding new fields, updating display names) may proceed without confirmation. For destructive operations (deprecating a field, changing LINKED `dataMode`), still prompt the user briefly: *"No evolution notes found for [field/schema]. Confirm it is safe to [describe change]?"*

**If a requested change would violate the recorded semantic intent:**

> The agent **must not proceed silently**. It must prompt the user:
>
> *"The requested change conflicts with the recorded semantic intent of [schema/field]. The `evolutionNotes` state: '[content]'. This change may break [reason]. Do you want to proceed and update the semantic metadata to reflect the new intent?"*
>
> Only proceed after explicit user confirmation. After confirmation, update `evolutionNotes` to reflect the new intent.

**Safe evolution operations (proceed without user confirmation):**

- Adding new fields that do not conflict with existing semantics.
- Updating `fieldDisplays` or display-only metadata.
- Adding or updating `semantic` metadata without changing field behavior.
- Changing `required`/`validateRequired` when `evolutionNotes` does not forbid it.
- Inserting test data records.

**Operations that require user confirmation:**

- Removing a field (deprecating it) when `evolutionNotes` or `purpose` indicate it is referenced elsewhere.
- Changing `dataMode` on a LINKED field from `SNAPSHOT` to `LIVE` or vice versa.
- Changing `fieldMode` or `selectedFieldIds` on LINKED/RELATED fields.
- Changing `currencyCode` or `fractionDigits` on a MONEY field with existing data.
- Any change where `evolutionNotes` explicitly warns against it.

---

## 7. AI Agent Workflow

### 7.1 Startup

```
1. POST /auth/sessions → obtain JWT
2. GET /uma/apps → inspect existing apps and their semantics
3. GET /uma/apps/{appId}/schemas → inspect existing schemas
```

### 7.2 Interpreting a User Prompt

When a user requests a feature:

1. **Map the request to entities.** Use `semantic.meaning` to identify which app and form the request relates to.
2. **Identify required fields.** Compare the request to existing fields by reading `semantic.purpose`.
3. **Check `evolutionNotes` on relevant schemas and fields.** If the request violates any notes, prompt the user before continuing (see Section 6.3).
4. **Determine the operation:** new app, new schema, new fields, new records, or query.

### 7.3 Creating a New Data Model

```
1. POST /uma/apps  (if app does not exist)
2. POST /uma/apps/{appId}/schemas/{formId}  (define schema with fields and semantics)
3. If LINKED/RELATED fields are needed:
   a. Create the referenced schema first
   b. Then create the schema with LINKED/RELATED fields
4. POST /uma/apps/{appId}/form/{formId}  (insert test data)
```

### 7.4 Evolving an Existing Schema

```
1. GET /uma/apps/{appId}/schemas/{formId}  → load current schema
2. Read semantic.evolutionNotes on schema and all fields
3. If change is blocked by evolutionNotes → prompt user for confirmation
4. Merge new fields into the existing fields map
5. PUT /uma/apps/{appId}/schemas/{formId}  → submit full updated schema
   (always include existing non-deprecated fields to prevent accidental deprecation)
6. PATCH /uma/apps/{appId} or PATCH /uma/apps/{appId}/schemas/{formId}  → update semantic if needed
```

**Never omit existing fields from a PUT body unless intentionally deprecating them.**

### 7.5 Inserting Test Data

```
1. GET /uma/apps/{appId}/schemas/{formId}  → retrieve field definitions
2. For each field:
   - Use fieldType to generate a valid test value
   - Respect validateRequired: true fields (must not be null)
   - Skip SEQUENCE and UNIQUE fields (server-generates them)
   - For LINKED fields: first insert referenced records and use their IDs
3. POST /uma/apps/{appId}/form/{formId}  → insert record
4. Repeat as needed for test coverage
```

### 7.6 Querying Data

```
POST /uma/apps/{appId}/form/{formId}/records:query
```

Use for:
- Verifying inserted test data exists.
- Searching for records matching a feature's filter criteria.
- Counting or aggregating data for validation.

### 7.7 Automatic Error Recovery

The agent must handle the following errors without surfacing them to the user:

| Error | Auto-Recovery |
|-------|--------------|
| `HTTP 401` | Re-authenticate and retry. |
| `DUPLICATED_APPS` | App already exists. Use existing. Do not re-create. |
| `DUPLICATED_SCHEMA` | Schema already exists. Fetch it and merge changes via PUT. |
| `APP_NOT_EXIST` | Create the app, then retry original operation. |
| `SCHEMA_NOT_EXIST` | Create the schema, then retry original operation. |
| `VERSION_CONFLICT` | Fetch the current record to get its `version`, then retry. |
| `VERSION_IS_MISSING` | Fetch the current record, extract `version`, add to request, retry. |
| `APP_IS_DEPRECATED` | Call `PATCH /apps/{appId}/restore`, then retry. |
| `SCHEMA_IS_DEPRECATED` | Call `PATCH /schemas/{formId}/restore`, then retry. |
| `FIELD_IS_DEPRECATED` | Call `PATCH /schemas/{formId}/fields/{fieldId}/restore`, then retry. |
| `INVALID_ENUM_VALUE` | Read `allowedValues` from error response. Use a valid value and retry. |
| `INVALID_TYPE` | Read `expectedType` from error response. Fix `fieldType` and retry. |
| `FIELD_TYPE_CHANGED_NOT_ALLOWED` | Do not change type. Deprecate the existing field and add a new field with a new `fieldId` of the desired type. Prompt user if semantic notes flag the field as critical. |
| `SCHEMA_LINKED_ERROR` | Verify `refFormId` exists. Fix `fieldMode`/`selectedFieldIds` per field rules. Retry. |
| `SCHEMA_RELATED_ERROR` | Add a LINKED field to the referenced schema pointing back to this form, then retry. |
| `VALIDATION_FAILED` | Read `details.violations`. Each entry has `field` and `message`. Fix the listed fields and retry. |

### 7.8 Semantic Integrity Rule

After completing any schema evolution, update `semantic.evolutionNotes` on the modified schema to record what changed and why. This preserves the semantic chain for future evolution passes.

Example:

```
PATCH /uma/apps/{appId}
Content-Type: application/json

{
  "semantic": {
    "evolutionNotes": "Added phoneNumber (TEXT) in v2 for SMS notifications. Do not remove email — still used as join key."
  }
}
```

---

## Appendix: Immutability Rules

| Rule | What happens if violated |
|------|--------------------------|
| `fieldId` cannot be renamed | No rename endpoint exists. Deprecate and create new. |
| `fieldType` cannot be changed | Returns `FIELD_TYPE_CHANGED_NOT_ALLOWED` |
| `appId` cannot be changed after creation | No mechanism to change it |
| Deprecated schemas cannot be modified | Returns `SCHEMA_IS_DEPRECATED` |
| Deprecated apps cannot be modified | Returns `APP_IS_DEPRECATED` |
