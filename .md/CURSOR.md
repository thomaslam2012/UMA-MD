# UMA Frontend Generation Workflow

Build a frontend app backed by UMA. Follow the three-phase workflow strictly.

## Normative sources (AI / Cursor)

- This document is **domain-agnostic**: it does **not** assume any specific business (rentals, CRM, health, etc.). The **`appId`**, **`formId`** values, and **schemas** you create in Phase 2 define the domain. Generated UI **MUST** be driven only by those schemas and by **permissions**, not by assumed entities.
- **Cursor / agents:** also obey **`.cursor/rules/*.mdc`** (always applied in this repo), including **`ui-quality.mdc`**, and **`UMA-COMPLIANCE-CHECKLIST.md`**. Before claiming Phase 3 or frontend work is complete, produce a **checklist status table** (every applicable row: Done / N/A + reason). Do not treat “working demo” as complete if checklist items are unmet.
- **Single source of truth:** If checklist or rules appear to conflict with this file, **`CLAUDE.md` wins**; update the checklist or rules to match, do not improvise API or workflow behavior.

## Execution Guardrails (Mandatory)

1. MUST follow this file strictly with phase gates.
2. MUST execute in order: Phase 1 -> Phase 2 -> Phase 3.
3. MUST ask user for UMA API key before any code changes or scaffolding.
4. MUST STOP if credentials are missing and wait for user input.
5. MUST summarize outputs after each phase and wait for explicit user approval before continuing.
6. MUST NOT scaffold frontend until app and schemas are successfully created in UMA.

## Base URLs

```
UMA           = http://localhost:8080
UMA-DASHBOARD = http://localhost:8081
UMA-APP-AUTH  = http://localhost:8082
```

## Auth Header (all UMA/DASHBOARD calls after login)

```
Authorization: Bearer <jwt>
```

---

## Phase 1 — Developer Login (get JWT for Phases 2)

Prompt user for their API key, then:

**POST** `UMA-DASHBOARD/uma-dashboard/auth/sessions`
```json
{ "authType": "API_KEY", "apiCredentials": "<api_key>" }
```
Response:
```json
{
  "data": {
    "userId": "...", "tenantId": "...", "token": "<JWT>",
    "tokenType": "Bearer", "userType": "OWNER|ADMIN|AI|APPS"
  }
}
```
Store: `jwt`, `tenantId`. Use `Authorization: Bearer <jwt>` for all Phase 2 calls.

---

## Phase 2 — Design Data Model

### Create App
**POST** `UMA/uma/apps`
```json
{
  "appId": "my-app",
  "appName": "My Application",
  "description": "optional"
}
```
Response: AppInfoDto with same fields + `id`, `createdAt`, `updatedAt`, `deprecated`, `version`.
Store: `appId`.

### List Schemas
**GET** `UMA/uma/apps/{appId}/schemas?includeDeprecated=false`
Response: `{ "items": [...FormSchemaDto], "count": N, "totalCount": N, "page": 0, "pageSize": 0 }`

### Create Schema
**POST** `UMA/uma/apps/{appId}/schemas/{formId}`
```json
{
  "appId": "my-app",
  "formId": "customers",
  "description": "optional",
  "ownerScoped": false,
  "fields": {
    "<fieldId>": { ...FormField... }
  }
}
```
`appId` + `formId` in body must match path params.
Response: FormSchemaDto (201).

### Update / Delete / Restore Schema
- **PUT**   `UMA/uma/apps/{appId}/schemas/{formId}` — same body as POST (200)
- **DELETE** `UMA/uma/apps/{appId}/schemas/{formId}` — soft delete (204)
- **PATCH**  `UMA/uma/apps/{appId}/schemas/{formId}/restore` — restore (204)

### Field Operations
- **DELETE** `UMA/uma/apps/{appId}/schemas/{formId}/fields/{fieldId}` — soft delete field (204)
- **PATCH**  `UMA/uma/apps/{appId}/schemas/{formId}/fields/{fieldId}/restore` — restore field (204)

---

## Schema Field Types

All fields share these base properties:
```json
{
  "fieldId": "name",
  "fieldType": "...",
  "required": false,
  "validateRequired": false,
  "allowMultiple": false,
  "deprecated": false,
  "fieldDisplays": { "en": "Display Name" }
}
```

### TEXT
```json
{ "fieldType": "TEXT", "fieldId": "name", "regexPattern": "optional regex", "length": 100 }
```

### NUMERIC
```json
{ "fieldType": "NUMERIC", "fieldId": "age", "decimalPlaces": 2, "range": "0,150" }
```
`range` format: `"min,max"`.

### BOOLEAN
```json
{ "fieldType": "BOOLEAN", "fieldId": "active" }
```
Accepts: `true`, `false`, `"true"`, `"false"`, `1`, `0`.

### DATE
```json
{ "fieldType": "DATE", "fieldId": "birthDate", "datePattern": "ISO_LOCAL_DATE" }
```
Input: `"yyyy-MM-dd"` string.
`datePattern` values: `ISO_LOCAL_DATE`, `SLASH_DATE`, `DASH_DATE`, `SLASH_DAY_FIRST`, `DASH_MONTH_FIRST`, `SLASH_MONTH_FIRST`, `LONG_DATE`, `MEDIUM_DATE`

### DATE_TIME
```json
{ "fieldType": "DATE_TIME", "fieldId": "createdAt", "dateTimePattern": "ISO_LOCAL_DATE_TIME" }
```
Input: `"yyyy-MM-dd'T'HH:mm:ss"`.
`dateTimePattern` values: `ISO_LOCAL_DATE_TIME`, `ISO_LOCAL_DATE`, `ISO_LOCAL_TIME`, `DATE_TIME_AMPM_1`–`DATE_TIME_AMPM_11`, `DATE_TIME_24H_1`–`DATE_TIME_24H_4`

### TIME
```json
{ "fieldType": "TIME", "fieldId": "startTime", "timePattern": "ISO_LOCAL_TIME" }
```
Input: `"HH:mm:ss"`.
`timePattern` values: `ISO_LOCAL_TIME`, `HOURS_MINUTES_24H`, `HOURS_MINUTES_AMPM`, `HOURS_MINUTES_SECONDS_AMPM`

### EMAIL
```json
{ "fieldType": "EMAIL", "fieldId": "email" }
```

### URL
```json
{ "fieldType": "URL", "fieldId": "website" }
```

### MONEY
```json
{
  "fieldType": "MONEY", "fieldId": "price",
  "currencyCode": "USD", "fractionDigits": 2,
  "allowCurrencyOverride": false, "allowFractionDigitsOverride": false
}
```
`currencyCode` values: `USD`, `EUR`, `JPY`, `GBP`, `AUD`
Record value: `{ "centAmount": 1999, "currencyCode": "USD", "fractionDigits": 2 }`

### FILE
```json
{ "fieldType": "FILE", "fieldId": "attachment" }
```
Max 5 MB. Record value: `{ "bytesBase64": "<base64>", "contentType": "image/png" }`

### SEQUENCE
```json
{
  "fieldType": "SEQUENCE", "fieldId": "invoiceNumber",
  "prefix": "INV", "separator": "-", "length": 8
}
```
**`length` = total chars of output string** (prefix + separator + digits). `INV-0001` = 8.
`length - prefix.length() - separator.length()` must be > 0 or `SCHEMA_FIELD_TYPE_ERROR`.
Values auto-incremented; system formats as `{prefix}{separator}{zero-padded-number}`.

### UNIQUE
```json
{
  "fieldType": "UNIQUE", "fieldId": "trackingCode",
  "prefix": "TRK", "separator": "-", "length": 10
}
```
Same `length` rule as SEQUENCE. Generates random (not sequential) numeric suffix automatically; no input needed when writing records.

### LINKED (reference with embedded data)
```json
{
  "fieldType": "LINKED", "fieldId": "customer",
  "refFormId": "customers",
  "dataMode": "LIVE",
  "fieldMode": "FULL",
  "selectedFieldIds": [],
  "unique": false
}
```
- `dataMode`: `LIVE` (fetch at read time) | `SNAPSHOT` (capture at write time)
- `fieldMode`: `FULL` (all fields) | `PARTIAL` (only `selectedFieldIds`)
- `selectedFieldIds` must be empty when `FULL`, required when `PARTIAL`

Write: provide ObjectId(s) of referenced record(s).
Read response value:
```json
[{ "id": "<objectId>", "_formId": "customers", "name": "Alice", "age": 30 }]
```

### RELATED (reference without embedded data)
```json
{
  "fieldType": "RELATED", "fieldId": "category",
  "refFormId": "categories",
  "fieldMode": "FULL",
  "selectedFieldIds": []
}
```
No `dataMode`, no `unique`. Same `fieldMode`/`selectedFieldIds` rules as LINKED.

### EMBED (inline nested schema)

```json
{
  "fieldType": "EMBED", "fieldId": "address",
  "embeddedFormSchema": {
    "appId": "my-app", "formId": "address-embed",
    "fields": {
      "street": { "fieldType": "TEXT", "fieldId": "street" },
      "city":   { "fieldType": "TEXT", "fieldId": "city" }
    }
  }
}
```

### USER_ID
 ```json
 { "fieldType": "USER_ID" }
 ```
  Stores a userId. Renders as a readonly text input with a lookup icon.

  On lookup click, call:
  GET UMA/uma/apps/{appId}/users?page={page}&pageSize={pageSize}

  Show firstName + lastName in the picker. On select, store userId as the field value.
  On read: returns { userId, firstName, lastName, email, status } instead of a plain string.
  On write: send only the userId string. On edit load, extract userId from the response object and display firstName + lastName as the label.

## Semantic Metadata

  App, schema, and each field support a `semantic` object for documenting business rules and intent:

  ```json
  {
    "intent": "What this entity is for",
    "meaning": "What it represents semantically",
    "purpose": "Business purpose",
    "constraints": ["rule 1", "rule 2"],
    "behavior": "Expected behavior",
    "evolutionNotes": "How it may change over time"
  }
  ```

  Used as a machine-readable spec guard — before implementing new features, consult semantic fields on affected app/schemas/fields to detect conflicts with existing business rules.

  **Always populate `semantic` on every app, schema, and field during Phase 2. Do not leave it empty.**

---

## Phase 3 — Generate Runtime Frontend Code

By this point `tenantId`, `appId`, and all `formId`s are known. Embed them as env vars / constants in generated code.

### App User Authentication (UMA-APP-AUTH)

All UMA-APP-AUTH requests must include `tenantId` and `appId` in the request body (POST) or query params (GET). The service extracts them via `TenantContextFilter`.

Generate these pages:
- `/login` — email + password form. Links to: Sign Up (if policy `status` is `OPEN`), Resend Verification Email, Forgot Password.
- `/sign-up` — check policy first; if `status == CLOSE`, do not show sign-up. Use `defaultRole` from policy response.
- `/verify` — shown after sign-up; code input + resend link.
- `/forgot-password` — email input to request reset code.
- `/reset-password` — code + new password form.

#### Login
**POST** `UMA-APP-AUTH/uma-app/auth/sessions`
```json
{ "authType": "PASSWORD", "email": "...", "password": "...", "tenantId": "...", "appId": "..." }
```
Response:
```json
{ "data": { "userId": "...", "token": "<JWT>", "tokenType": "Bearer", "tenantId": "..." } }
```
Store JWT. Use `Authorization: Bearer <jwt>` for all UMA CRUD calls.

#### Sign Up
Before showing sign-up form, check policy:
**GET** `UMA-APP-AUTH/uma-app/auth/sign-up-policy?tenantId={tenantId}&appId={appId}`
```json
{
  "data": {
    "tenantId": "...", "appId": "...",
    "type": "PUBLIC|DOMAIN",
    "allowedDomains": ["example.com"],
    "status": "OPEN|CLOSE",
    "defaultRole": "member",
    "roles": ["member", "admin"]
  }
}
```
If `status == CLOSE`, do not allow sign-up.

**POST** `UMA-APP-AUTH/uma-app/auth/users`
```json
{
  "firstName": "John", "lastName": "Smith",
  "email": "john@example.com", "password": "secret123",
  "tenantId": "...", "appId": "...", "role": "member"
}
```
Response (201): user object with `status: "PENDING"` (email verification required).

#### Email Verification
**POST** `UMA-APP-AUTH/uma-app/auth/users/verification-codes`
```json
{ "email": "...", "code": "123456", "tenantId": "...", "appId": "..." }
```
Response: 204

**POST** `UMA-APP-AUTH/uma-app/auth/users/verification-codes/resend`
```json
{ "email": "...", "tenantId": "...", "appId": "..." }
```
Response: 204

#### Password Reset
**POST** `UMA-APP-AUTH/uma-app/auth/password-reset-requests`
```json
{ "email": "...", "tenantId": "...", "appId": "..." }
```
Response: 204

**POST** `UMA-APP-AUTH/uma-app/auth/password-resets`
```json
{ "email": "...", "code": "123456", "newPassword": "...", "tenantId": "...", "appId": "..." }
```
Response: 204

---
### Dynamic UI — Schema + Permission Driven UI

Never hardcode fields, forms, nav items, or buttons. All UI derived at runtime.

After login, fetch these two endpoints to drive all UI rendering:

**GET** `UMA/uma/apps/{appId}/schemas` — relevant response shape for UI generation:

  ```json
  {
    "items": [
      {
        "formId": "...",
        "deprecated": false,
        "fields": {
          "<fieldId>": {
            "fieldId": "...",
            "fieldType": "...",
            "fieldDisplays": { "en": "Label" },
            "required": true,
            "deprecated": false,
            "allowMultiple": false
          }
        }
      }
    ]
  }
  ```
- Skip any form or field where `deprecated: true`
- For `EMBED` fields, recurse into `embeddedFormSchema.fields` using the same structure
- All other properties (`semantic`, `version`, `createdAt`, `id`, `description`, etc.) are not used for UI generation

**GET** `UMA/uma/apps/{appId}/permissions`

Permission response:

    ```json
    {
      "data": {
        "role": "admin",
        "permissionsMap": {
          "customers": { "actions": ["READ","WRITE","DELETE"], "scope": "ALL|OWN", "visible": true }
        },
        "fullAccess": false,
        "readOnly": false,
        "allVisible": false,
        "allowManageUsers": true,
        "allowManagePermissions": false,
        "allowManageSignUpPolicy": false
      }
    }
    ```
Permission fields:

Per-form fields (inside permissionsMap.<formId>):
- actions: READ, WRITE, DELETE
- visible: controls whether the form shows in the UI sidebar and dashboard

Top-level fields (check these first — if true, skip per-form fields):
- fullAccess: true → treat all forms as READ/WRITE/DELETE regardless of permissionsMap
- readOnly: true → treat all forms as READ-only regardless of permissionsMap
- allVisible: true → treat all forms as visible in sidebar/nav regardless of permissionsMap

Rules:
- Fields: always iterate schema.fields
- List: show only if actions includes READ
- Create/Edit: show only if actions includes WRITE
- Delete: show only if actions includes DELETE
- visible is independent of fullAccess/readOnly — allVisible only affects sidebar visibility, not CRUD access
- Nav must be built by filtering permissions.data.permissionsMap where visible === true. Never create a NAV_ITEMS array or any hardcoded form list. Use visible in UMA to control what appears — never hardcode exclusions in frontend code.

---

### UI Quality Standard — Modern, Production-Ready

This project requires a polished UI quality baseline in addition to functional correctness.

- Use a consistent, reusable design system (shared primitives + tokens), not ad-hoc page styling.
- Preserve clear visual hierarchy with consistent typography, spacing, and grouped form/list sections.
- Implement complete UX states: loading, empty, error, success, disabled, and read-only.
- Ensure accessibility baseline: keyboard navigation, visible focus styles, semantic labels, and sufficient contrast.
- Maintain responsive behavior across mobile/tablet/desktop layouts.
- Avoid unstyled native controls as final UI output.

Refer to **`.cursor/rules/ui-quality.mdc`** for normative requirements.

---

### CRUD Endpoints

All require `Authorization: Bearer <app-user-jwt>`.

#### Create Record
**POST** `UMA/uma/apps/{appId}/form/{formId}`
```json
{
  "appId": "my-app", "formId": "customers", "version": 0,
  "data": { "name": "Alice", "age": 30 }
}
```
Response (200): FormDataDto

#### Get Record
**GET** `UMA/uma/apps/{appId}/form/{formId}/records/{recordId}`
Response: FormDataDto (not wrapped)

#### List Records
**GET** `UMA/uma/apps/{appId}/form/{formId}?page=0&pageSize=10&includeDeprecated=false`
Response: `{ "items": [...], "count": N, "totalCount": N, "page": 0, "pageSize": 10 }`

#### Update Record
**PUT** `UMA/uma/apps/{appId}/form/{formId}/records/{recordId}`
Body: same as POST. Response: FormDataDto.

#### Delete Record
**DELETE** `UMA/uma/apps/{appId}/form/{formId}/records/{recordId}` — Response: 200

#### Bulk Delete
**DELETE** `UMA/uma/apps/{appId}/form/{formId}/records`
```json
{ "objectIds": ["<id1>", "<id2>"] }
```
Response: 200

#### Query with Filters

**POST** `/uma/apps/{appId}/form/{formId}/records:query`

**Request body:**
  ```json
  {
    "page": 0,
    "pageSize": 10,
    "sort": [{ "field": "fieldId", "direction": "ASC|DESC" }],
    "filter": {
      "operator": "AND",
      "conditions": [
        { "field": "fieldId", "operator": "EQUALS", "value": "active" },
        { "field": "age", "operator": "GREATER_THAN", "value": 18 }
      ]
    }
  }
```
Field paths:
- Regular fields: fieldId directly (e.g. "dealName")
- LINKED/RELATED sub-fields: "fieldId.subFieldId" (e.g. "owner.name")
- System fields: direct name (e.g. "_id", "createdAt")

Leaf operators: EQUALS, NOT_EQUALS, GREATER_THAN, GREATER_THAN_OR_EQUAL, LESS_THAN, LESS_THAN_OR_EQUAL, IN, LIKE
- LIKE uses regex syntax: ".*searchTerm.*"

Group operators: AND, OR — use "conditions": [...] instead of "field"/"value". Can be nested.
---
## FormDataDto Structure (record response)

```json
{
  "id": "<hex objectId>",
  "appId": "my-app",
  "formId": "customers",
  "version": 1,
  "createdAt": "2024-01-01T00:00:00-05:00",
  "updatedAt": "2024-01-01T00:00:00-05:00",
  "createdBy": "<userId>",
  "updatedBy": "<userId>",
  "deprecated": false,
  "data": { "name": "Alice", "age": 30 }
}
```

## Response Wrappers

| Wrapper | Shape |
|---|---|
| Single item | `{ "data": { ... } }` |
| List | `{ "items": [...], "count": N, "totalCount": N, "page": N, "pageSize": N }` |

## `ownerScoped` on Schema

  When `true`, enables row-level security — records are filtered so users only see their own data. The ownership filter is built automatically from
   the schema structure:

  - `createdBy == currentUserId` — always included
  - USER_ID field present → also matches records **assigned** to the user
  - LINKED field pointing to a form with USER_ID → also matches records **linked** to the user

**Warning:** Without a USER_ID or LINKED field, only the record creator can see their own records (Rule 1 only). If records are created on behalf of
  someone else (e.g. appointments, prescriptions), the form must have a USER_ID field or a LINKED to a form with USER_ID — otherwise the assigned
  user sees nothing.

---

## Implementation Notes

### SEQUENCE field `length`
Must be the **total length** of the generated output string (prefix + separator + digits). E.g. `STU-0001` = 8 chars → `length: 8`. Setting `length: 4` when prefix+separator already takes 4 chars fails with `SCHEMA_FIELD_TYPE_ERROR`.

### TypeScript `verbatimModuleSyntax`
Always use the `type` keyword for type-only imports: `import { type FormEvent }`, `import { type ReactNode }`.

### LINKED/RELATED fields — write vs read
- **Write (create/update):** send a plain ObjectId string in `data`.
- **Read (list/get response):** field comes back as a resolved array `[{ "id": "...", "_formId": "...", ...fields }]`.
- **Edit form pitfall:** `initial.data` is populated from the read response, so LINKED fields already contain resolved arrays. Normalize them back to plain ObjectId strings before submitting, otherwise the PUT sends the full array object instead of an id.

### Update `version` field
Use the record's actual `version` from the GET/list response, not `0`. Sending `0` on update may cause optimistic locking conflicts.

### DATE/TIME write format
Pattern only affects read. Write always ISO: `yyyy-MM-dd`, `yyyy-MM-dd'T'HH:mm:ss`, `HH:mm:ss`. Normalize on edit.

### Use `required` in schema to drive `*` indicators in UI forms.

### On 401 response from UMA, clear auth and redirect to `/login`

### Error shape: `{ "error": { "code": "...", "message": "..." } }`. Extract via `err.error.message`, not `err.message`

### User Profile
  **GET** `UMA/uma/apps/{appId}/user` — response wrapped in `{ "data": { ... } }`, unwrap before use.
  - Display `firstName`, `lastName`, and `role` at the bottom of the sidebar above Sign Out.

### Field Type Rendering
  Each field type requires specific UI treatment — never render all fields as plain text inputs.
  Key types:
  - **USER_ID**: readonly input + lookup icon → user picker (see USER_ID section)
  - **BOOLEAN**: toggle or checkbox
  - **DATE / DATE_TIME / TIME**: date/time picker
  - **MONEY**: currency-formatted input with symbol prefix
  - **FILE**: file upload dropzone, max 5 MB
  - **LINKED / RELATED**: record picker — search and select from referenced form records
  - **SEQUENCE / UNIQUE**: readonly — system-generated, never editable
  - **EMBED**: render as inline nested form section
