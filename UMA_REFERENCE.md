# UMA Reference

**Audience:** AI coding agents (Claude Code, Cursor, etc.)
**Purpose:** Detailed API, field type, and semantic reference for UMA. Read this when you need exact request/response formats, field type options, or semantic evolution rules.

---

## Table of Contents

- [2. API Endpoints](#2-api-endpoints)
  - [2.1 Apps](#21-apps--umaapps)
  - [2.2 Schemas](#22-schemas--umaappsappidschemas)
  - [2.3 Form Data](#23-form-data--umaappsappidformformid)
  - [2.4 Sign-Up Policy & Role Permissions](#24-sign-up-policy--role-permissions)
- [4. Field Types](#4-field-types)
- [5. Field Attributes](#5-field-attributes)
  - [5.1 Base Attributes](#51-base-attributes-all-field-types)
  - [5.2 TEXT](#52-text)
  - [5.3 NUMERIC](#53-numeric)
  - [5.4 DATE](#54-date)
  - [5.5 DATE_TIME](#55-date_time)
  - [5.6 TIME](#56-time)
  - [5.7 BOOLEAN](#57-boolean)
  - [5.8 EMAIL](#58-email)
  - [5.9 URL](#59-url)
  - [5.10 FILE](#510-file)
  - [5.11 MONEY](#511-money)
  - [5.12 SEQUENCE](#512-sequence)
  - [5.13 UNIQUE](#513-unique)
  - [5.14 LINKED](#514-linked)
  - [5.15 RELATED](#515-related)
  - [5.16 EMBED](#516-embed)
- [6. Semantic Metadata](#6-semantic-metadata)
  - [6.1 The semantic Object](#61-the-semantic-object)
  - [6.2 Attribute Definitions](#62-attribute-definitions)
  - [6.3 Semantic-Driven Evolution Rules](#63-semantic-driven-evolution-rules)

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

**Request Body:** Same shape as POST. Must include `version`.

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
          "fieldDisplays": { "en": "Display Name" },
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
  "ownerScoped": false,
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
      "fieldDisplays": { "en": "Full Name" },
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
      "fieldDisplays": { "en": "Email Address" },
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

**Schema-level attributes:**

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `ownerScoped` | boolean | `false` | If `true`, UMA automatically records the `userId` of the creator on every data record saved to this form. All retrieval endpoints (GET list, GET single, and query) filter results to only return records owned by the currently logged-in user, unless the user's role has `scope: ALL` in its role permissions for this form. If `false` or not set, all records are visible to all users regardless of role `scope`. |

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

> ⚠️ **No `data` wrapper on form data endpoints.** Responses from `/uma/apps/{appId}/form/...` are returned at the top level — `{ "items": [...] }` or a single record object directly. Do NOT access `response.data` — access `response` directly. This is different from auth and permissions endpoints (UMA-APP / UMA-Dashboard services) which DO wrap their responses in `{ "data": { ... } }`.

---

#### `GET /uma/apps/{appId}/form/{formId}`

**Purpose:** Retrieve a paginated list of records for a form.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `includeDeprecated` | boolean | `false` | Include soft-deleted records |
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
      "deprecated": false,
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
- LINKED fields require a valid ObjectId string referencing an existing record in the target form.

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

**Purpose:** Soft-delete a single record.

**Response:** `HTTP 204 No Content`

---

#### `DELETE /uma/apps/{appId}/form/{formId}/records`

**Purpose:** Soft-delete multiple records by ID.

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
| `includeDeprecated` | boolean | `false` | Include soft-deleted records |

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

**`LIKE` uses Java regex, not SQL wildcards.** The backend compiles the `value` as a `java.util.regex.Pattern` (case-insensitive). Do not use `%` — use `.*` instead.

| Use case | Pattern |
|----------|---------|
| Contains "foo" | `.*foo.*` |
| Starts with "foo" | `^foo.*` |
| Ends with "foo" | `.*foo$` |
| Exact match | `^foo$` |

**Searchable field types** — only these field types produce meaningful results with `LIKE`. Use this list when building a search box to determine which fields to include in the filter:

`TEXT`, `EMAIL`, `URL`, `LINKED`, `RELATED`, `SEQUENCE`, `UNIQUE`

Do not apply `LIKE` to `NUMERIC`, `BOOLEAN`, `DATE`, `DATE_TIME`, `TIME`, `MONEY`, or `EMBED` fields.

**Searching inside LINKED and RELATED fields — dotted sub-field paths:**

LINKED and RELATED fields store object references, not plain strings. Sending `{ LIKE, field: "doctor", value: ".*tho.*" }` does not work — the backend cannot substring-match an object. Instead use dotted sub-field paths: `fieldId.subFieldId`.

```json
{ "operator": "LIKE", "field": "doctor.name",  "value": ".*tho.*" }
{ "operator": "LIKE", "field": "doctor.email", "value": ".*tho.*" }
```

- `fieldMode: FULL` — expand search across all text-like fields in the referenced schema
- `fieldMode: PARTIAL` — expand search only across `selectedFieldIds`
- Never send the plain `{ LIKE, field: "doctor" }` condition — replace it entirely with the expanded dotted-path conditions
- Nested LINKED/RELATED inside the referenced form are **not** recursed further

Only these types are LIKE-searchable inside a referenced record:

| Type | Searchable |
|---|---|
| `TEXT`, `EMAIL`, `URL`, `SEQUENCE`, `UNIQUE` | ✓ |
| `NUMERIC`, `BOOLEAN`, `DATE`, `DATE_TIME`, `TIME`, `MONEY`, `FILE`, `EMBED` | ✗ |
| `LINKED`, `RELATED` | ✗ (not recursed) |

**Implementation steps for any frontend:**

1. Collect all LINKED/RELATED fields from the schema's active fields
2. Fetch each referenced form's schema (parallel requests recommended)
3. For each LINKED/RELATED field, check `fieldMode` and `selectedFieldIds`
4. Expand into dotted-path `LIKE` conditions (`fieldId.subFieldId`) for each searchable sub-field
5. Never send the plain `{ LIKE, field: "doctor" }` condition — replace it entirely with the expanded dotted-path conditions

**Example** — schema has `fullName (TEXT)`, `email (EMAIL)`, `doctor (LINKED → Doctor, fieldMode: PARTIAL, selectedFieldIds: ["name"])`. Search term `tho`:

```json
{
  "operator": "OR",
  "conditions": [
    { "operator": "LIKE", "field": "fullName",    "value": ".*tho.*" },
    { "operator": "LIKE", "field": "email",       "value": ".*tho.*" },
    { "operator": "LIKE", "field": "doctor.name", "value": ".*tho.*" }
  ]
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

---

**Aggregation request** — include `aggregation` in the body. Pagination and sort are ignored.

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

### 2.4 Sign-Up Policy & Role Permissions

#### Naming Legend

The API identifier names in this section do not obviously describe what they do. Always use this table to resolve what a name means before reading or writing any code:

| Portal Label | API Identifier | Response field in `GET /permissions` | What it controls |
|---|---|---|---|
| Form Permissions | `permissionsMap` (in role permissions) | `permissionsMap` | What each role can READ / WRITE / DELETE per form |
| Sign Up Policy Management | `sign-up-policy-management-permissions` | `allowManageSignUpPolicy` | Which roles can read or update the sign-up policy |
| Role & Permission | `form-management-permissions` | `allowManagePermissions` | **Master permission** — which roles can manage role permissions AND grant/revoke both management rights |
| User Management | *(documented separately)* | `allowManageUsers` | Which roles can manage users |

> ⚠️ **`form-management-permissions` is misleadingly named.** Despite sounding like it only relates to forms, it is the **master IAM permission** — a role with this right can manage role permissions, grant/revoke sign-up policy management access, and grant/revoke this right itself. If no role has this right enabled, all permission management at runtime is read-only and can only be recovered via the UMA portal.

---

#### What Guards What

| To access this at runtime | The role needs this right | Check this field in `GET /permissions` |
|---|---|---|
| `GET/POST /uma/apps/{appId}/iam/sign-up-policy` | `sign-up-policy-management-permissions` | `allowManageSignUpPolicy: true` |
| `GET/POST /uma/apps/{appId}/iam/roles-permissions` | `form-management-permissions` | `allowManagePermissions: true` |
| `GET/POST /uma/apps/{appId}/iam/sign-up-policy-management-permissions` | `form-management-permissions` | `allowManagePermissions: true` |
| `GET/POST /uma/apps/{appId}/iam/form-management-permissions` | `form-management-permissions` | `allowManagePermissions: true` |
| `GET /uma/apps/{appId}/permissions` | None — available to any logged-in user | — |

---

#### Access Paths

All management endpoints are available via two paths depending on the caller:

| Caller | Service | Base path | Auth |
|---|---|---|---|
| Tenant (AI agent, dev time) | UMA-APP | `http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/...` | Dev token |
| App user with management right (runtime) | UMA | `http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/...` | User Bearer token |

The tenant always has full access regardless of role settings. App users need the specific right enabled for their role.

---

#### Sign-Up Policy

Controls who is allowed to register for the app.

**Object fields:**

| Field | Type | Description |
|---|---|---|
| `type` | string | `PUBLIC` — any valid email can sign up. `DOMAIN` — only emails from `allowedDomains` can sign up. |
| `status` | string | `OPEN` — sign-up is allowed. `CLOSE` — sign-up is disabled entirely. |
| `allowedDomains` | array of strings | Required and non-empty when `type` is `DOMAIN`. Ignored when `type` is `PUBLIC`. |
| `defaultRole` | string | Role automatically assigned to every new user on sign-up. Must exist in `roles`. |
| `roles` | array of strings | All roles defined for this app. This is the role registry — a role must be in this list to be usable anywhere in the system. |

**Default policy auto-created when app is created:**

```
status: OPEN
type: PUBLIC
defaultRole: ADMIN
roles: [ADMIN]
allowedDomains: []
```

**Dev time (tenant) — via UMA-APP service:**

```
GET  http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/sign-up-policy?tenantId={tenantId}&appId={appId}
POST http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/sign-up-policy
Authorization: Bearer <devToken>
```

**Runtime (app user) — requires `allowManageSignUpPolicy: true` in `GET /permissions` (Sign Up Policy Management / `sign-up-policy-management-permissions`):**

```
GET  http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy
Authorization: Bearer <userToken>
```

GET response / POST request body:

```json
{
  "tenantId": "tnt_1vzaisck",
  "appId": "DentalBooking",
  "type": "PUBLIC",
  "status": "OPEN",
  "allowedDomains": [],
  "defaultRole": "ADMIN",
  "roles": ["ADMIN"]
}
```

**Constraint:** If `type` is `DOMAIN`, `allowedDomains` must be non-empty — returns `SIGN_UP_POLICY_DOMAIN_MISSING` otherwise.

**Common sign-up policy patterns:**

| Business need | `status` | `type` | `allowedDomains` |
|---|---|---|---|
| Open to anyone | `OPEN` | `PUBLIC` | `[]` |
| Company staff only (self-register) | `OPEN` | `DOMAIN` | `["company.com"]` |
| Invite only | `CLOSE` | any | any |

**Invite-only flow** — when `status: CLOSE`, self-registration is disabled entirely. A staff member with `allowManageUsers: true` creates accounts manually via the User Management endpoints. After account creation, the system automatically sends a verification email to the new user. The user verifies their email, resets their password, and can then log in. This covers the full invite-only onboarding pattern without any additional configuration.

---

#### Sign-Up Error Codes

Returned by `POST /uma-app/auth/users` when the sign-up policy rejects the request:

| HTTP | Error Code | Cause |
|---|---|---|
| 412 | `SIGN_UP_POLICY_MISSING` | No sign-up policy exists for this app |
| 403 | `SIGN_UP_CLOSE` | Sign-up policy `status` is `CLOSE` |
| 400 | `SIGN_UP_INVALIDATE_EMAIL` | Email format is invalid |
| 403 | `SIGN_UP_INVALIDATE_DOMAIN` | Email domain is not in `allowedDomains` |
| 412 | `SIGN_UP_POLICY_DOMAIN_MISSING` | `type` is `DOMAIN` but `allowedDomains` is empty |
| 412 | `SIGN_UP_DEFAULT_ROLE_MISSING` | `defaultRole` is not set on the policy |
| 412 | `SIGN_UP_POLICY_NO_ROLES` | `roles` is empty on the policy |
| 400 | `SIGN_UP_INVALID_ROLE` | Role in request is not in the `roles` list |
| 412 | `ROLES_NOT_FOUND` | Role referenced does not exist |

---

#### Role Permissions

Controls what each role can do per form (schema).

**Object fields:**

| Field | Type | Description |
|---|---|---|
| `role` | string | The role name. Must exist in the sign-up policy `roles` list. |
| `permissionsMap` | object | Map of `formId → Permissions`. Ignored when `fullAccess` is `true`. |
| `fullAccess` | boolean | If `true`, role has unlimited access to all forms including newly created schemas. `permissionsMap` is empty. |
| `readOnly` | boolean | If `true`, role can only READ across all forms regardless of `permissionsMap`. |

**`Permissions` object** (value in `permissionsMap`):

| Field | Type | Description |
|---|---|---|
| `actions` | array of strings | Allowed actions: `READ`, `WRITE`, `DELETE` |
| `scope` | string or null | `OWN` or `null` — user sees only records they created. `ALL` — user sees all records. Only applies when the form schema has `ownerScoped: true`. If `ownerScoped` is `false` or not set, all records are visible regardless of `scope`. |

**Default ADMIN role permissions auto-created when app is created:**

```
fullAccess: true
readOnly: false
allowManageUsers: true
allowManagePermissions: true
allowManageSignUpPolicy: true
```

Because `fullAccess: true`, the ADMIN role automatically gains access to all new schemas without any permission updates.

**Dev time (tenant) — via UMA-APP service:**

```
GET  http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/roles-permissions?tenantId={tenantId}&appId={appId}&role={role}
POST http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/roles-permissions
Authorization: Bearer <devToken>
```

**Runtime (app user) — requires `allowManagePermissions: true` in `GET /permissions` (Role & Permission / `form-management-permissions`):**

```
GET  http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/roles-permissions?role={role}
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/roles-permissions
Authorization: Bearer <userToken>
```

GET response / POST request body:

```json
{
  "tenantId": "tnt_1vzaisck",
  "appId": "DentalBooking",
  "role": "USER",
  "permissionsMap": {
    "Order": {
      "actions": ["READ", "WRITE", "DELETE"],
      "scope": "OWN"
    },
    "Product": {
      "actions": ["READ"],
      "scope": "ALL"
    }
  },
  "fullAccess": false,
  "readOnly": false
}
```

**Note:** Non-`fullAccess` roles must be explicitly updated with permissions for every new schema added to the app.

---

#### Management Access Rights

Controls which roles can manage sign-up policy and role permissions at runtime. There are two management rights — see the **Naming Legend** and **What Guards What** tables above before reading this section.

The tenant always has full access to all management endpoints regardless of role settings.

---

**Sign Up Policy Management** (`sign-up-policy-management-permissions`) — grants a role the ability to read and update the sign-up policy:

| Protected endpoint | Description |
|---|---|
| `GET/POST /uma/apps/{appId}/iam/sign-up-policy` | Read or update the sign-up policy |

Dev time (tenant):

```
GET  http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/sign-up-policy-management-permissions?role={role}
POST http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/sign-up-policy-management-permissions
Authorization: Bearer <devToken>
```

Runtime (app user — requires `allowManagePermissions: true` in `GET /permissions`):

```
GET  http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy-management-permissions?role={role}
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy-management-permissions
Authorization: Bearer <userToken>
```

---

**Role & Permission** (`form-management-permissions`) — the master permission right, grants a role the ability to manage all permissions at runtime:

| Protected endpoint | Description |
|---|---|
| `GET/POST /uma/apps/{appId}/iam/roles-permissions` | Read or update role permissions |
| `GET/POST /uma/apps/{appId}/iam/form-management-permissions` | Grant/revoke this right for other roles |
| `GET/POST /uma/apps/{appId}/iam/sign-up-policy-management-permissions` | Grant/revoke sign-up policy management right for other roles |

Dev time (tenant):

```
GET  http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/form-management-permissions?role={role}
POST http://{UMA_APP_HOST}:{UMA_APP_PORT}/uma-app/form-management-permissions
Authorization: Bearer <devToken>
```

Runtime (app user — requires `allowManagePermissions: true` in `GET /permissions`):

```
GET  http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/form-management-permissions?role={role}
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/form-management-permissions
Authorization: Bearer <userToken>
```

**Request body** (same shape for all POST management access endpoints):

```json
{
  "role": "ADMIN",
  "allowed": true
}
```

**Response:**

```json
{
  "data": {
    "tenantId": "tnt_1vzaisck",
    "appId": "OnlineStore",
    "role": "ADMIN",
    "allowed": true
  }
}
```

> ⚠️ **Lockout warning:** If no role has `form-management-permissions: true`, all permission management at runtime becomes read-only — no logged-in user can change any permissions or access rights. The only way to recover is for the tenant to use the UMA portal (`/uma-app/` endpoints), since the tenant always has full access regardless of role settings.

---

#### User Permissions (Runtime)

Returns the permissions for the currently logged-in user. The server resolves the user's role from the Bearer token — the frontend does not need to know or pass the role explicitly. No special permission required — available to any logged-in user.

> ⚠️ **Response wrapper rule — read before writing any code:**
> - Auth and permissions endpoints (UMA-APP / UMA-Dashboard services) → response is wrapped: `response.data.fieldName`
> - Form data endpoints (`/uma/apps/{appId}/form/...`) → response is NOT wrapped: `response.items`, `response.totalCount` directly
>
> Never assume all endpoints follow the same pattern. Always check which service the endpoint belongs to.

```
GET http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/permissions
Authorization: Bearer <userToken>
```

**Response** (wrapped in `data`, same as all other endpoints):

```json
{
  "data": {
    "role": "ADMIN",
    "permissionsMap": {
      "Order": {
        "actions": ["READ", "WRITE", "DELETE"]
      },
      "Category": {
        "actions": ["READ", "WRITE", "DELETE"],
        "scope": "ALL"
      }
    },
    "fullAccess": false,
    "readOnly": false,
    "allowManagePermissions": true,
    "allowManageSignUpPolicy": true,
    "allowManageUsers": true,
    "updatedAt": "2026-03-26T23:32:26.803-04:00",
    "version": 74,
    "_id": "69bb3d8657ab26a3ac417c9b"
  }
}
```

| Response Field | Description |
|---|---|
| `role` | The role assigned to the logged-in user |
| `permissionsMap` | Map of `formId → Permissions`. Only forms the user has access to are included. |
| `fullAccess` | If `true`, user has unlimited access to all forms. `permissionsMap` will be empty. |
| `readOnly` | If `true`, user can only READ across all forms regardless of `permissionsMap`. |
| `scope` | Only present in a `permissionsMap` entry when the form has `ownerScoped: true`. `OWN` or `null` = own records only. `ALL` = all records. |
| `allowManagePermissions` | If `true`, user can access the role permissions management page and grant/revoke all management rights for any role. Corresponds to the **Role & Permission** (`form-management-permissions`) right. |
| `allowManageSignUpPolicy` | If `true`, user can view and edit the sign-up policy page. Corresponds to the **Sign Up Policy Management** (`sign-up-policy-management-permissions`) right. |
| `allowManageUsers` | If `true`, user can manage app users. Corresponds to the **User Management** right. |
| `updatedAt` | ISO timestamp of the last permissions update. |
| `version` | Optimistic lock version of the permissions record. |
| `_id` | Internal record ID. |

**Agent use:** Call this immediately after login during frontend initialization. This is the single source of truth for everything the UI needs — form-level permissions, management page access, and the user's role. Do not make additional calls to determine what the user can see or do.

---

#### Frontend: Management Page Implementation

Use the following pattern to initialize and save each management page. The `GET /uma/apps/{appId}/permissions` response (already fetched during app init) determines which pages are accessible.

---

**Permissions Management Page** — requires `allowManagePermissions: true`

To initialize, fetch the selected role's permissions:

```
GET http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/roles-permissions?role={selectedRole}
Authorization: Bearer <userToken>
```

Cross-join with schemas already loaded from `GET /uma/apps/{appId}/schemas` to build the Form Permissions table:

```ts
for each schema in schemas:
  row = rolePerms.permissionsMap[schema.formId] ?? { actions: [], scope: null }
```

UI element → response field mapping:

| UI element | Response field |
|---|---|
| Sign Up Policy Management toggle | `allowManageSignUpPolicy` |
| User Management toggle | `allowManageUsers` |
| Role & Permission toggle | `allowManagePermissions` |
| Read Only toggle | `readOnly` |
| Full Access column | `fullAccess` |
| Per-form Read / Write / Delete checkboxes | `permissionsMap[formId].actions` |
| Per-form Scope dropdown | `permissionsMap[formId].scope` — only show when schema has `ownerScoped: true`, otherwise show `—` |

To save, POST the full updated permissions object:

```
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/roles-permissions
Authorization: Bearer <userToken>
```

```json
{
  "role": "ADMIN",
  "version": 74,
  "allowManageSignUpPolicy": true,
  "allowManageUsers": true,
  "allowManagePermissions": true,
  "readOnly": false,
  "fullAccess": false,
  "permissionsMap": {
    "Category": { "actions": ["READ", "WRITE", "DELETE"], "scope": "ALL" },
    "Customer":  { "actions": ["READ", "WRITE", "DELETE"] },
    "Product":   { "actions": ["READ", "WRITE", "DELETE"] }
  }
}
```

Rules:
- `version` is required — take it from the GET response to avoid `VERSION_CONFLICT`
- Send the **full** `permissionsMap` — omitting a form removes its permissions
- `scope` only applies when the form schema has `ownerScoped: true`; omit or null it otherwise
- `fullAccess: true` overrides all `permissionsMap` entries — table checkboxes are irrelevant when this is on

---

**Sign-Up Policy Page** — requires `allowManageSignUpPolicy: true`

To initialize, fetch the sign-up policy:

```
GET http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy
Authorization: Bearer <userToken>
```

UI element → response field mapping:

| UI element | Response field |
|---|---|
| Registration Type (Public / Domain Restricted) | `type`: `PUBLIC` / `DOMAIN` |
| Allowed Domains chips | `allowedDomains` — only relevant when `type: DOMAIN` |
| Registration Status (Open / Closed) | `status`: `OPEN` / `CLOSE` |
| Roles list | `roles` |
| Default role selector | `defaultRole` |

To save:

```
POST http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/iam/sign-up-policy
Authorization: Bearer <userToken>
```

```json
{
  "version": 12,
  "type": "DOMAIN",
  "status": "OPEN",
  "allowedDomains": ["tetonsoft.com"],
  "defaultRole": "MANAGER",
  "roles": ["MANAGER", "ADMIN"]
}
```

Rules:
- `version` is required — take it from the GET response
- If `type` is `DOMAIN`, `allowedDomains` must be non-empty — returns `SIGN_UP_POLICY_DOMAIN_MISSING` otherwise
- `defaultRole` must exist in `roles`

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
| `FILE` | File stored inline as Base64 in MongoDB (max 5 MB decoded) |
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
| `fieldDisplays` | object | optional | Display labels keyed by language code (e.g., `{ "en": "Full Name" }`) |
| `allowMultiple` | boolean | `false` | Whether the field accepts an array of values |
| `required` | boolean | `false` | Whether the field is structurally required |
| `validateRequired` | boolean | `false` | Whether the server validates the value is non-null on save |
| `deprecated` | boolean | `false` | Soft-delete flag. Set by server when field is omitted from a PUT. |
| `semantic` | object | required | Semantic metadata (see Section 6). Always populate — never omit. |

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

⛔ **Always send ISO format (`yyyy-MM-dd`) to the server when writing records — regardless of what display format the field is configured with.** `datePattern` controls the output format only (how the server returns the value on read). It does not affect what the server expects on write.

| Scenario | Server returns (read) | Client must send (write) |
|---|---|---|
| Field has a pattern | Value in pattern format (e.g. `"2026/03/17"`) | ISO string `"2026-03-17"` |
| Field has no pattern | ISO string `"2026-03-17"` | ISO string `"2026-03-17"` |

`datePattern` available values:

| Value | Example |
|-------|---------|
| `ISO_LOCAL_DATE` | `2026-03-17` |
| `SLASH_DATE` | `2026/03/17` |
| `DASH_DATE` | `17-03-2026` |
| `SLASH_DAY_FIRST` | `17/03/2026` |
| `DASH_MONTH_FIRST` | `03-17-2026` |
| `SLASH_MONTH_FIRST` | `03/17/2026` |
| `LONG_DATE` | `Tuesday, Mar 17, 2026` |
| `MEDIUM_DATE` | `Tue, 17 Mar 2026` |

---

### 5.5 DATE_TIME

Input value: `"yyyy-MM-dd'T'HH:mm:ss"` (e.g., `"2025-01-15T09:30:00"`). Stored as UTC.

```json
{
  "fieldType": "DATE_TIME",
  "dateTimePattern": "ISO_LOCAL_DATE_TIME"
}
```

⛔ **Always send ISO format (`yyyy-MM-ddTHH:mm:ss`) to the server when writing records — regardless of display format.** `dateTimePattern` controls output format only.

| Scenario | Server returns (read) | Client must send (write) |
|---|---|---|
| Field has a pattern | Value in pattern format (e.g. `"2026/03/17 02:30 PM"`) | ISO string `"2026-03-17T14:30:00"` |
| Field has no pattern | ISO string `"2026-03-17T14:30:00"` | ISO string `"2026-03-17T14:30:00"` |

`dateTimePattern` available values:

| Value | Example |
|-------|---------|
| `ISO_LOCAL_DATE_TIME` | `2026-03-17T14:30:00` |
| `DATE_TIME_24H_1` | `2026-03-17 14:30:00` |
| `DATE_TIME_24H_2` | `2026/03/17 14:30` |
| `DATE_TIME_24H_3` | `17-03-2026 14:30` |
| `DATE_TIME_24H_4` | `2026-03-17T14:30:00` |
| `DATE_TIME_AMPM_1` | `2026-03-17 02:30 PM` |
| `DATE_TIME_AMPM_2` | `2026-03-17 02:30:00 PM` |
| `DATE_TIME_AMPM_3` | `2026/03/17 02:30 PM` |
| `DATE_TIME_AMPM_4` | `2026/03/17 02:30:00 PM` |
| `DATE_TIME_AMPM_5` | `03-17-2026 02:30 PM` |
| `DATE_TIME_AMPM_6` | `03/17/2026 02:30 PM` |
| `DATE_TIME_AMPM_7` | `17-03-2026 02:30 PM` |
| `DATE_TIME_AMPM_8` | `17/03/2026 02:30 PM` |
| `DATE_TIME_AMPM_9` | `2026-03-17T02:30:00 PM` |
| `DATE_TIME_AMPM_10` | `Tuesday, Mar 17, 2026 02:30 PM` |
| `DATE_TIME_AMPM_11` | `Tue, 17 Mar 2026 02:30:00 PM` |

---

### 5.6 TIME

Input value: `"HH:mm:ss"` (e.g., `"14:30:00"`). Stored as UTC.

```json
{
  "fieldType": "TIME",
  "timePattern": "HH_MM_SS"
}
```

⛔ **Always send ISO format (`HH:mm:ss`) to the server when writing records — regardless of display format.** `timePattern` controls output format only.

| Scenario | Server returns (read) | Client must send (write) |
|---|---|---|
| Field has a pattern | Value in pattern format (e.g. `"02:30 PM"`) | ISO string `"14:30:00"` |
| Field has no pattern | ISO string `"14:30:00"` | ISO string `"14:30:00"` |

`timePattern` available values:

| Value | Example |
|-------|---------|
| `ISO_LOCAL_TIME` | `14:30:00` |
| `HOURS_MINUTES_24H` | `14:30` |
| `HOURS_MINUTES_AMPM` | `02:30 PM` |
| `HOURS_MINUTES_SECONDS_AMPM` | `02:30:00 PM` |

---

### 5.7 BOOLEAN

No additional attributes. Value must be `true` or `false`.

---

### 5.8 EMAIL

No additional attributes beyond base. Value is stored as-is; no server-side format validation beyond type acceptance.

---

### 5.9 URL

No additional attributes beyond base.

---

### 5.10 FILE

Stores file content **directly in MongoDB** as a Base64-encoded object. There is no external file storage — the file data lives in the record.

No additional schema-level attributes beyond base. The field definition requires no extra configuration:

```json
{
  "fieldId": "attachment",
  "fieldType": "FILE",
  "fieldDisplays": { "en": "Attachment" },
  "validateRequired": false
}
```

**Record-level value** — when inserting or updating a record, provide the file as:

```json
{
  "data": {
    "attachment": {
      "bytesBase64": "<base64-encoded file content>",
      "contentType": "application/pdf"
    }
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `bytesBase64` | string | yes | Base64-encoded file content. Must be valid Base64. |
| `contentType` | string | yes | MIME type (e.g., `"image/png"`, `"application/pdf"`). Must be non-blank. |

**Constraints:**
- Maximum file size: **5 MB** after Base64 decoding. Larger files return `FORM_VALIDATE_FAIL`.
- Both `bytesBase64` and `contentType` must be present and non-blank when `validateRequired` is `true`.
- `allowMultiple: true` allows an array of file objects per record.

**Storage note:** Because files are embedded in MongoDB documents, this field type is best suited for small files (documents, images, attachments under 5 MB). For larger files, use an external storage service and store the URL in a `URL` field instead.

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

Input value when creating a record — integer `centAmount`, or full object:

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

Output example: `"ORD-0000001"`

**Filtering:** Use the formatted string value in filter conditions. The server automatically converts it to the stored numeric value before querying.

```json
{ "field": "orderNumber", "operator": "EQUALS", "value": "ORD-0000001" }
```

Range queries — use `GREATER_THAN_OR_EQUAL` and `LESS_THAN_OR_EQUAL` together for an inclusive range:

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

**Record-level value:** Provide the referenced record's ObjectId string (24-char hex):

```json
{
  "data": {
    "customer": "64a1b2c3d4e5f6a7b8c9d0e1"
  }
}
```

When `allowMultiple: true`, provide an array of ObjectId strings.

**Response format in LIVE mode:** The field value is returned as an array regardless of `allowMultiple`. Always access the first element as `record.data.fieldId[0]`.

**`fieldMode` and `id`:** Regardless of `fieldMode`, the `id` of the referenced record is always returned. Use `PARTIAL` when only a few fields are needed for display — the `id` is always available for a follow-up fetch of the full record.

| `fieldMode` | What is returned |
|---|---|
| `FULL` | All fields from the referenced record + `id` |
| `PARTIAL` | Only `selectedFieldIds` + `id` |

**LINKED vs RELATED:**

| Feature | LINKED | RELATED |
|---|---|---|
| `dataMode` | ✓ LIVE / SNAPSHOT | — not supported |
| `fieldMode` | ✓ FULL / PARTIAL | ✓ FULL / PARTIAL |
| `selectedFieldIds` | ✓ when PARTIAL | ✓ when PARTIAL |
| `allowMultiple` | ✓ | — not supported |
| Writable | ✓ | ✗ read-only |

Use `LINKED` when you need control over data freshness (`dataMode`) or multi-select. Use `RELATED` for a simple back-reference.

---

### 5.15 RELATED

Inverse side of a LINKED relationship. **Server-computed — never stored on the record itself.** The server resolves it at query time by finding records in the referenced form whose LINKED field points back to the current record. **Read-only — do not include in record `data`.**

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

The `id` of each back-referenced record is always returned regardless of `fieldMode`.

**Response format:** RELATED fields are returned as an **array** of back-referenced records. For example, fetching a Doctor record with a RELATED field `patients` returns all Patient records whose `doctor` LINKED field points to that Doctor's `id`. The Doctor record itself never stores a patient list — the server resolves it at read time.

**Response data shape — LINKED vs RELATED:**

The API returns referenced record data in two possible shapes depending on field type:

| Shape | Example | Typical field |
|---|---|---|
| Nested | `{ "id": "abc", "data": { "fullName": "Tom" } }` | LINKED |
| Flat | `{ "_id": "abc", "fullName": "Tom" }` | RELATED |

Always handle both shapes when reading referenced record values.

**Display name resolution:**

When displaying field labels from a referenced record, resolve the display name in this order:
1. `fieldDisplays['en']`
2. `fieldDisplays['EN']`
3. Raw `fieldId` as fallback

---

### 5.16 EMBED

Embeds an inline sub-schema within the field. The embedded data is stored inside the parent record — not as a separate form. Do not create a separate schema for embedded data; provide the schema inline in `embeddedFormSchema`.

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

**Record-level value:** Provide an object matching the embedded schema fields:

```json
{
  "data": {
    "address": {
      "street": "123 Main St",
      "city": "Springfield"
    }
  }
}
```

When `allowMultiple: true`, provide an array of such objects.

---

## 6. Semantic Metadata

### 6.1 The `semantic` Object

Present at three levels: App, Schema, and Field. All three levels use the same structure.

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

**If a requested change would violate the recorded semantic intent:**

The agent **must not proceed silently**. It must prompt the user:

> *"The requested change conflicts with the recorded semantic intent of [schema/field]. The `evolutionNotes` state: '[content]'. This change may break [reason]. Do you want to proceed and update the semantic metadata to reflect the new intent?"*

Only proceed after explicit user confirmation. After confirmation, update `evolutionNotes` to reflect the new intent.

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
