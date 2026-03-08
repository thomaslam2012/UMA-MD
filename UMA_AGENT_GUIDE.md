# UMA Agent Guide

**Audience:** AI coding agents (Claude Code, Cursor, etc.)
**Purpose:** Operate UMA autonomously as a semantic data model layer during development.
**Base URL:** `http://{UMA_HOST}:{UMA_PORT}/uma`

---

## Table of Contents

- [What UMA Is](#what-uma-is)
- [1. Authentication](#1-authentication)
  - [Before You Start — Resolve Configuration Values](#before-you-start--resolve-configuration-values)
  - [1.1 Development-Time Auth](#11-development-time-auth-ai-agent--uma)
  - [1.2 Runtime Auth](#12-runtime-auth-generated-app--uma-app)
    - [1.2.1 Sign Up](#121-sign-up)
    - [1.2.2 Verify Email](#122-verify-email)
    - [1.2.3 Resend Verification Email](#123-resend-verification-email)
    - [1.2.4 Password Reset Request](#124-password-reset-request)
    - [1.2.5 Password Reset](#125-password-reset)
    - [1.2.6 Login](#126-login)
- [2. API Endpoints](#2-api-endpoints)
  - [2.1 Apps](#21-apps--umaapps)
  - [2.2 Schemas](#22-schemas--umaappsappidschemas)
  - [2.3 Form Data](#23-form-data--umaappsappidformformid)
- [3. Error Handling](#3-error-handling)
  - [Error Code Reference and Agent Recovery](#error-code-reference-and-agent-recovery)
  - [Auth Failure Recovery](#auth-failure-recovery)
  - [Retry Limits](#retry-limits)
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
- [7. AI Agent Workflow](#7-ai-agent-workflow)
  - [7.0 Master Build Workflow](#70-master-build-workflow)
  - [7.1 Startup](#71-startup)
  - [7.2 Interpreting a User Prompt](#72-interpreting-a-user-prompt)
  - [7.3 Creating a New Data Model](#73-creating-a-new-data-model)
  - [7.4 Evolving an Existing Schema](#74-evolving-an-existing-schema)
  - [7.5 Inserting Test Data](#75-inserting-test-data)
  - [7.6 Querying Data](#76-querying-data)
  - [7.7 Automatic Error Recovery](#77-automatic-error-recovery)
  - [7.8 Semantic Integrity Rule](#78-semantic-integrity-rule)
- [Appendix: Immutability Rules](#appendix-immutability-rules)

---

## What UMA Is

UMA (Universal Model API) is not a traditional CRUD database API. It is a **semantic schema evolution engine** designed to be manipulated by AI agents. Every entity in UMA carries structured semantic metadata that preserves intent across schema changes. An AI agent should treat UMA as the authoritative source of truth about what a data model means, not just what it contains.

---

## 1. Authentication

UMA has two distinct authentication contexts. The AI agent must understand and implement both.

### Before You Start — Resolve Configuration Values

Before making any API call, the agent must read configuration from `uma.env` in the project root.

**Steps:**
1. Look for `uma.env` in the project root directory.
2. If found, parse all key=value pairs from it.
3. If the file does not exist or any required key is missing, stop and ask the user: *"Please provide a `uma.env` file in the project root with the required values. See the template below."*

**`uma.env` format:**

```
UMA_DASHBOARD_HOST=localhost
UMA_DASHBOARD_PORT=8080
UMA_DASHBOARD_CREDENTIALS=your-api-key-here

UMA_HOST=localhost
UMA_PORT=8080

UMA_APP_HOST=localhost
UMA_APP_PORT=8082

# Written by agent after app is created — do not edit manually
UMA_TENANT_ID=
UMA_APP_ID=
```

| Key | Description |
|-----|-------------|
| `UMA_DASHBOARD_HOST` | Hostname where UMA-Dashboard is running |
| `UMA_DASHBOARD_PORT` | Port of UMA-Dashboard |
| `UMA_DASHBOARD_CREDENTIALS` | API key for AI agent authentication (provided by UMA admin) |
| `UMA_HOST` | Hostname where UMA is running |
| `UMA_PORT` | Port of UMA |
| `UMA_APP_HOST` | Hostname where UMA-APP is running |
| `UMA_APP_PORT` | Port of UMA-APP |
| `UMA_TENANT_ID` | Written by agent after dev-time auth. Do not set manually. |
| `UMA_APP_ID` | Written by agent after app is created. Do not set manually. |

Do not hardcode any of these values in generated code. Always read from `uma.env`.

---

### 1.1 Development-Time Auth (AI Agent → UMA)

Used **only by the AI agent** during development to create apps, schemas, and test data. This is not for end users.

**Endpoint:**

```
POST http://{UMA-DASHBOARD-HOST}:{UMA-DASHBOARD-PORT}/uma-dashboard/auth/sessions
Content-Type: application/json
```

**Request body:**

```json
{
  "authType": "API_KEY",
  "apiCredentials": "{UMA-DASHBOARD-CREDENTIALS}"
}
```

**Response:**

```json
{
  "data": {
    "userId": "usr_d2125821",
    "tenantId": "tnt_1f8e2jur",
    "token": "eyJ...",
    "tokenType": "Bearer"
  }
}
```

**Extract and store:**
- `data.token` — used as `Authorization: Bearer <token>` on every `/uma/*` API call.
- `data.tenantId` — store this. It is required when generating the runtime signup UI for the app.

**Environment variables (never hardcode):**

| Variable | Description |
|----------|-------------|
| `UMA_DASHBOARD_HOST` | Hostname of UMA-Dashboard service |
| `UMA_DASHBOARD_PORT` | Port of UMA-Dashboard service |
| `UMA_DASHBOARD_CREDENTIALS` | API key credential string |
| `UMA_HOST` | Hostname of UMA service |
| `UMA_PORT` | Port of UMA service |

**Using the token:**

```
Authorization: Bearer <data.token>
```

Include this header on every `http://{UMA_HOST}:{UMA_PORT}/uma/*` request.

**Auto-retry on 401:**

If any UMA endpoint returns `HTTP 401`, the agent must:
1. Re-call `POST /uma-dashboard/auth/sessions` to obtain a fresh token.
2. Retry the original request with the new token.
3. Do not surface the 401 to the user — it is an internal token refresh.
4. If the retry also returns `401`, stop and surface to user: *"UMA authentication failed. Check UMA_DASHBOARD_CREDENTIALS."*

**Dev-time auth pseudocode:**

```
devToken = null
tenantId = null

function callUMA(method, path, body):
    if devToken is null:
        devToken, tenantId = devAuthenticate()
    response = http(method, "http://" + UMA_HOST + ":" + UMA_PORT + "/uma" + path,
                    Authorization: "Bearer " + devToken, body)
    if response.status == 401:
        devToken, tenantId = devAuthenticate()
        response = http(method, "http://" + UMA_HOST + ":" + UMA_PORT + "/uma" + path,
                        Authorization: "Bearer " + devToken, body)
    return response

function devAuthenticate():
    response = http("POST",
        "http://" + UMA_DASHBOARD_HOST + ":" + UMA_DASHBOARD_PORT + "/uma-dashboard/auth/sessions",
        body={
            "authType": "API_KEY",
            "apiCredentials": env("UMA_DASHBOARD_CREDENTIALS")
        })
    return response.body.data.token, response.body.data.tenantId
```

---

### 1.2 Runtime Auth (Generated App → UMA-APP)

When the AI agent creates an app via `POST /uma/apps`, the following end-user authentication endpoints are **automatically available** on the UMA-APP service. The agent does not need to create or configure them.

**If the user's prompt requires signup, login, or user management**, implement the runtime auth flows in Sections 1.2.1–1.2.6.

**If the user says the app should work without signup or login**, the generated app still needs a Bearer token to call `/uma/*` endpoints at runtime. Without an end-user authentication system in the app, the only available token is the development-time token from Section 1.1, which requires embedding the `UMA_DASHBOARD_CREDENTIALS` API key directly in the application code.

Before proceeding, the agent **must** warn the user and ask for explicit confirmation:

> *"Since your app has no end-user login system, the only way for the app to call UMA is to embed the developer API key credentials directly in the generated code. This is NOT secure — anyone with access to the code or browser can extract the credentials and access your UMA data.*
>
> *UMA-APP already supports user signup and login with no extra setup required on your part. I can implement that instead and your data will be protected.*
>
> *Do you want to proceed with embedded credentials (not secure), or should I implement signup and login?"*

- **If user confirms embedded credentials:** Generate the app so that on load it calls `POST /uma-dashboard/auth/sessions` with the hardcoded `apiCredentials`, stores the returned `data.token`, and uses it as the Bearer token for all `/uma/*` calls. Add a visible comment in the code warning that this is insecure and should be replaced with proper login before production use.
- **If user does not confirm (or asks for login instead):** Implement the full signup and login flow using the UMA-APP endpoints in Sections 1.2.1–1.2.6.

The `tenantId` and `appId` to use in all requests below are already known to the agent at development time:
- `tenantId` — from the dev-time auth response (`data.tenantId`)
- `appId` — the app the agent created via `POST /uma/apps`

Embed these as configuration constants in any generated code.

The **login response token** (`data.token`) is the Bearer token the generated app uses to call `/uma/*` data endpoints on behalf of the logged-in user.

---

#### 1.2.1 Sign Up

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/users
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "password": "userpassword",
  "firstName": "Thomas",
  "lastName": "Lam",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

After successful signup, the server sends a verification email. The user must verify before they can log in.

---

#### 1.2.2 Verify Email

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/users/verification-codes
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "code": "266066",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

The `code` is sent by the server to the user's email after signup. The generated UI should provide an input field for the user to enter it.

---

#### 1.2.3 Resend Verification Email

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/users/verification-codes/resend
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

---

#### 1.2.4 Password Reset Request

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/password-reset-requests
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

The server sends a reset code to the user's email. Use the code in the next step.

---

#### 1.2.5 Password Reset

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/password-resets
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "newPassword": "newpassword",
  "code": "130449",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

The `code` comes from the reset email sent in step 1.2.4. The generated UI should collect email, code, and new password on the same form.

---

#### 1.2.6 Login

```
POST http://{UMA-APP-HOST}:{UMA-APP-PORT}/uma-app/auth/sessions
Content-Type: application/json
```

```json
{
  "authType": "PASSWORD",
  "email": "user@example.com",
  "password": "userpassword",
  "tenantId": "tnt_1f8e2jur",
  "appId": "ChurchManagement"
}
```

**Response:**

```json
{
  "data": {
    "userId": "usr_d78d835a",
    "token": "eyJ...",
    "tokenType": "Bearer",
    "tenantId": "tnt_1f8e2jur"
  }
}
```

Store `data.token` as the session token. Include it as `Authorization: Bearer <data.token>` on every subsequent `/uma/*` data call made on behalf of this user. On `HTTP 401`, prompt the user to log in again.

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

**ID field note:** The App object uses `_id` (with underscore) as its identifier field. Schema and FormData objects use `id` (no underscore). Always use the correct field name when extracting IDs.

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

**Request Body:** Same shape as POST, plus `version`:

```json
{
  "appId": "ProductCatalog",
  "appName": "Product Catalog",
  "version": 1,
  "description": "Manages all product and inventory data",
  "semantic": {
    "intent": "...",
    "meaning": "...",
    "purpose": "...",
    "constraints": [],
    "behavior": "...",
    "evolutionNotes": "..."
  }
}
```

**Note:** `version` must match the current server-side version. Mismatch returns `VERSION_CONFLICT`. Always fetch the app first to get the current `version`.

**Response:** `HTTP 200` with the updated app object.

**Agent use:** Use when the full app record needs to be replaced. Always fetch the current app first to get `version` and existing field values before sending a PUT.

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

**Critical identifier rule:** The key in the `fields` map **must exactly equal** the `fieldId` value inside the field object. They are not inferred from each other — both must be present and identical.

```json
"fields": {
  "email": {          ← map key
    "fieldId": "email",   ← must match the key exactly
    "fieldType": "EMAIL"
  }
}
```

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
- `version` must be included in the body and must match the current server-side version. Mismatch returns `VERSION_CONFLICT`. Omitting it returns `VERSION_IS_MISSING`.
- Field types are **immutable**. Sending a field with the same `fieldId` but a different `fieldType` returns `FIELD_TYPE_CHANGED_NOT_ALLOWED`.
- Fields present in the existing schema but absent from the request body are automatically marked `deprecated: true` (soft-delete). They are not removed.
- Deprecated fields from the existing schema are preserved in the document even if not included in the request.

**Agent use:** Always fetch the current schema first with `GET /uma/apps/{appId}/schemas/{formId}`. Extract the `version` from the response and include it in the PUT body. Include all non-deprecated existing fields plus any new fields.

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

**Purpose:** Restore a soft-deleted schema. This is the **only** PATCH operation on schemas — there is no partial update PATCH for schemas. To update schema content or semantics, use `PUT`.

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

**Notes:**
- `version` must match the current server-side version. Mismatch returns `VERSION_CONFLICT`.
- Do not include SEQUENCE or UNIQUE fields in `data`. They are not regenerated on update; the server will ignore or reject them.
- LINKED and EMBED field format rules are the same as POST (see POST notes above).

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

> **Important:** The query body field is `appsId` (not `appId`). Using `appId` will cause the request to fail silently or return unexpected results.

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
| 400 | `VERSION_CONFLICT` | `version` in request does not match current version | Fetch the resource again (app, schema, or record) to get the current `version`, then retry. |
| 400 | `VERSION_IS_MISSING` | `version` not supplied in update request | Fetch the resource, extract `version`, include it in the request body, then retry. |
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
| `401` / `403` | Wrong credentials | Stop. Surface to user: *"Authentication failed. Check UMA_DASHBOARD_CREDENTIALS environment variable."* Do not retry with the same credentials. |
| `404` | Wrong auth URL | Stop. Surface to user: *"Auth endpoint not found. Check UMA_DASHBOARD_HOST and UMA_DASHBOARD_PORT."* |
| `5xx` | Auth server down | Wait 2 seconds and retry up to 3 times. If still failing, surface to user: *"UMA Dashboard auth server is unavailable."* |

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
| `SEQUENCE` | Auto-incrementing formatted identifier (e.g., `ORD-000001`) |
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
| `required` | boolean | `false` | Client-side/display hint only. The server does **not** reject null values based on this flag alone. Use this to communicate to UI generators that the field should be shown as required. |
| `validateRequired` | boolean | `false` | Server-enforced. When `true`, the server rejects the request if this field is null or absent in `data`. Set `validateRequired: true` whenever the field must genuinely be non-null at write time. |
| `deprecated` | boolean | `false` | Soft-delete flag. Set by server when field is omitted from a PUT. |
| `semantic` | object | **required** | Semantic metadata (see Section 6). Must be populated on creation. Do not leave empty. |

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

No additional schema-level attributes beyond base.

UMA stores file content **directly in MongoDB** — not a URL or external reference. The file is embedded inside the form record.

**Form data input format** — provide a JSON object with two fields:

```json
{
  "data": {
    "attachment": {
      "bytesBase64": "JVBERi0xLjQKJeLj...",
      "contentType": "application/pdf"
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `bytesBase64` | string | yes | Base64-encoded file content |
| `contentType` | string | yes | MIME type (e.g. `"application/pdf"`, `"image/png"`) |

**Server response** — returns the same object structure:

```json
{
  "data": {
    "attachment": {
      "bytesBase64": "JVBERi0xLjQKJeLj...",
      "contentType": "application/pdf"
    }
  }
}
```

**Validation errors:**

| Condition | Error | Message |
|-----------|-------|---------|
| `bytesBase64` is blank | `FORM_VALIDATE_FAIL` | `"bytesBase64 is missing"` |
| `contentType` is blank | `FORM_VALIDATE_FAIL` | `"contentType is missing"` |
| `bytesBase64` is not valid Base64 | `FORM_VALIDATE_FAIL` | `"Invalid base64 format"` |
| Decoded file exceeds 5MB | `FORM_VALIDATE_FAIL` | `"File size exceeds 5MB limit"` |

**Agent use:** When generating UI for FILE fields, the frontend must read the file, Base64-encode it, and send the encoded string with the correct MIME type. Do not send a file path or URL — UMA expects the raw file content encoded as Base64. The 5MB limit applies to the decoded file size.

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
{ "field": "orderNumber", "operator": "EQUALS", "value": "ORD-000001" }
```

Range queries are supported — use `GREATER_THAN_OR_EQUAL` and `LESS_THAN_OR_EQUAL` together for an inclusive range:

```json
{
  "operator": "AND",
  "conditions": [
    { "field": "orderNumber", "operator": "GREATER_THAN_OR_EQUAL", "value": "ORD-000203" },
    { "field": "orderNumber", "operator": "LESS_THAN_OR_EQUAL",    "value": "ORD-000208" }
  ]
}
```

The `IN` operator also accepts a list of formatted strings:

```json
{ "field": "orderNumber", "operator": "IN", "value": ["ORD-000203", "ORD-000205", "ORD-000208"] }
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

Output example: `"USR-482930"` (prefix=3, separator=1, padding=6 → total=10)

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

**RELATED fields are read-only.** Never include them in `data` when creating or updating records. The server populates them automatically by resolving the back-reference from the LINKED form.

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

### 7.0 Master Build Workflow

When the user gives a prompt to build any app or software, execute the following sequence in order. The sections referenced in each step provide the detailed rules.

---

**Phase 1 — Setup**

1. Read `uma.env` from the project root (Section 1 "Before You Start"). If the file is missing or any required key is absent, stop and ask the user before proceeding.
2. Authenticate with UMA-Dashboard (Section 1.1) → store `devToken` and `tenantId`.
3. Check `uma.env` for `UMA_APP_ID` and `UMA_TENANT_ID`:
   - **If both are present:** This project has an existing app. Use `UMA_APP_ID` as the `appId` and `UMA_TENANT_ID` as the `tenantId` for all subsequent operations. Skip app creation in Phase 3. Proceed directly to understanding the user's current request in Phase 2.
   - **If absent:** This is a new project. Proceed through all phases normally.

---

**Phase 2 — Design the Data Model**

4. Interpret the user prompt to identify entities, relationships, and field types (Section 7.2).
5. For each entity, design its schema: choose appropriate `fieldType` for each field and plan a fully populated `semantic` block at both schema and field level (Section 7.3). Never leave `semantic` empty.
6. Identify LINKED/RELATED dependencies and determine creation order — referenced schemas must exist before schemas that link to them.

---

**Phase 2.5 — Present Data Model for User Confirmation**

Before creating anything in UMA, present the designed data model to the user for confirmation. **Do not show raw JSON.** Display it as human-readable tables so non-technical users can understand and verify it.

For each schema, show:

```
## [SchemaName]
Intent:   [semantic.intent]
Purpose:  [semantic.purpose]
Behavior: [semantic.behavior]

| Field ID       | Display Name   | Type      | Required | Purpose                              |
|----------------|----------------|-----------|----------|--------------------------------------|
| productCode    | Product Code   | SEQUENCE  | No       | Auto-generated unique identifier     |
| name           | Product Name   | TEXT      | Yes      | The name of the product              |
| price          | Price          | MONEY     | Yes      | Selling price in USD                 |
| category       | Category       | LINKED    | Yes      | Links to the Category schema         |
```

Then ask:
> *"Here is the data model I've designed for your app. Does this look right? Let me know if you'd like to add, remove, or change anything before I build it."*

Only proceed to Phase 3 after the user confirms. If the user requests changes, update the design and re-present the tables before asking again.

---

**Phase 3 — Build the Data Model in UMA**

7. `POST /uma/apps` — create the app with full `semantic`. If it already exists (`DUPLICATED_APPS`), use the existing app.
   - **After the app is successfully created**, write `UMA_TENANT_ID` and `UMA_APP_ID` into `uma.env`:
     ```
     UMA_TENANT_ID=tnt_1f8e2jur
     UMA_APP_ID=ProductManagement
     ```
     This ensures future sessions know which app belongs to this project and do not create a new one.
8. `POST /uma/apps/{appId}/schemas/{formId}` — create each schema with full `semantic` on the schema and on every field. Follow the creation order from step 6.
9. Insert test data to verify the schema works (Section 7.5).
10. Record what was built in `semantic.evolutionNotes` at schema and app level (Section 7.8).

---

**Phase 4 — Determine Runtime Auth**

11. Determine whether the app requires end-user signup and login — either from the user prompt or by asking the user directly.
    - **Login required:** Implement UMA-APP signup and login in the generated code (Sections 1.2.1–1.2.6). The token from the login response (`data.token`) is used as the Bearer token for all runtime `/uma/*` calls.
    - **No login:** Follow the warning and confirmation flow in Section 1.2. Do not embed credentials until the user explicitly confirms. If confirmed, generate code that fetches a dev-time token on load and add a security warning comment in the code.

---

**Phase 5 — Generate Application Code**

12. Generate the frontend or application code:
    - Embed `tenantId` and `appId` as hardcoded constants (both are known at dev-time).
    - Wire all `/uma/*` data calls with `Authorization: Bearer <token>` using the token determined in Phase 4.
    - **Do not hardcode host/port values in the generated code.** Browser-based apps (React, Vue, etc.) cannot read `uma.env` directly. Instead, generate a `.env` file in the project root using the values from `uma.env`:
      - For **Vite** projects: use `VITE_` prefix
        ```
        VITE_UMA_HOST=<UMA_HOST>
        VITE_UMA_PORT=<UMA_PORT>
        VITE_UMA_APP_HOST=<UMA_APP_HOST>
        VITE_UMA_APP_PORT=<UMA_APP_PORT>
        ```
        Access in code as `import.meta.env.VITE_UMA_HOST`
      - For **Create React App** projects: use `REACT_APP_` prefix
        ```
        REACT_APP_UMA_HOST=<UMA_HOST>
        REACT_APP_UMA_PORT=<UMA_PORT>
        REACT_APP_UMA_APP_HOST=<UMA_APP_HOST>
        REACT_APP_UMA_APP_PORT=<UMA_APP_PORT>
        ```
        Access in code as `process.env.REACT_APP_UMA_HOST`
    - Add the generated `.env` to `.gitignore`. It contains the same values as `uma.env` and should not be committed.
13. Implement UI or handlers for all entities: list, create, edit, delete, as appropriate for the domain.
14. Use `POST .../records:query` for any search, filter, or paginated views (Section 7.6).

---

### 7.1 Startup

```
1. POST http://{UMA_DASHBOARD_HOST}:{UMA_DASHBOARD_PORT}/uma-dashboard/auth/sessions → obtain JWT (see Section 1.1)
2. GET http://{UMA_HOST}:{UMA_PORT}/uma/apps → inspect existing apps and their semantics
3. GET http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/schemas → inspect existing schemas
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

**Always populate `semantic` on creation.** When creating an app, schema, or field, derive `intent`, `meaning`, `purpose`, `behavior`, and `evolutionNotes` from the user prompt and domain context. Do not leave `semantic` empty or omit it. Future schema evolution depends on these notes being accurate from the start. Never submit a `POST /uma/apps`, `POST /uma/apps/{appId}/schemas/{formId}`, or any field definition without a fully populated `semantic` block.

---

**Circular LINKED references** (Schema A links to B, Schema B links back to A):

```
1. POST Schema A — omit the LINKED field that points to B
2. POST Schema B — include the LINKED field pointing to A
3. PUT Schema A — add the LINKED field pointing to B (now that B exists)
```

Never attempt to create both schemas simultaneously with cross-references — the server validates `refFormId` existence at creation time.

### 7.4 Evolving an Existing Schema

```
1. GET /uma/apps/{appId}/schemas/{formId}  → load current schema
2. Read semantic.evolutionNotes on schema and all fields
3. If change is blocked by evolutionNotes → prompt user for confirmation
4. Merge new fields into the existing fields map
5. PUT /uma/apps/{appId}/schemas/{formId}  → submit full updated schema
   (always include existing non-deprecated fields to prevent accidental deprecation)
   (include updated semantic in the PUT body — there is no PATCH for schema semantics)
6. PATCH /uma/apps/{appId}  → update app-level semantic if needed
   (PATCH on schemas is restore-only; schema semantic updates must go through PUT)
```

**Never omit existing fields from a PUT body unless intentionally deprecating them.**

### 7.5 Inserting Test Data

```
1. GET /uma/apps/{appId}/schemas/{formId}  → retrieve field definitions
2. For each field:
   - Use fieldType to generate a valid test value
   - Respect validateRequired: true fields (must not be null)
   - Skip SEQUENCE and UNIQUE fields (server-generates them)
   - Skip RELATED fields (read-only, server-populated)
   - For LINKED fields: first insert referenced records and use their ObjectId strings
   - For EMBED fields: provide an inline object matching embeddedFormSchema.fields
     (do NOT insert separate records — EMBED is not a foreign key)
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
| `VERSION_CONFLICT` | Fetch the current resource (app, schema, or record) to get its `version`, then retry. |
| `VERSION_IS_MISSING` | Fetch the current resource, extract `version`, add to request body, then retry. |
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

After completing any schema evolution, record what changed and why in `semantic.evolutionNotes`. There are two levels:

**Schema-level notes** — include updated `semantic` in the PUT body (no PATCH available for schemas):

```
PUT /uma/apps/{appId}/schemas/{formId}
Content-Type: application/json

{
  "version": <current>,
  "fields": { ... all existing + new fields ... },
  "semantic": {
    "evolutionNotes": "Added phoneNumber (TEXT) in v2 for SMS notifications. Do not remove email — still used as join key."
  }
}
```

**App-level notes** — use PATCH (partial update available for apps):

```
PATCH /uma/apps/{appId}
Content-Type: application/json

{
  "semantic": {
    "evolutionNotes": "v2: Customer schema extended with phoneNumber. Order schema added."
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
| FormData records cannot be restored after delete | DELETE is permanent (hard delete). There is no restore endpoint for records. |
| RELATED fields cannot be written | RELATED is read-only. Including it in `data` on POST/PUT will be ignored or cause an error. |
