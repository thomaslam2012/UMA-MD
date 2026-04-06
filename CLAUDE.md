# UMA Frontend Generation Workflow

Build a frontend app backed by UMA. Follow the three-phase workflow strictly.

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
{ "fieldType": "USER_ID", "fieldId": "ownerId" }
```
Auto-populated with current user's ID.

---

## Phase 3 — Generate Runtime Frontend Code

By this point `tenantId`, `appId`, and all `formId`s are known. Embed them as env vars / constants in generated code.

### App User Authentication (UMA-APP-AUTH)

All UMA-APP-AUTH requests must include `tenantId` and `appId` in the request body (POST) or query params (GET). The service extracts them via `TenantContextFilter`.

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

### Dynamic UI — Schema + Permission Driven

After login, call these two endpoints to drive all UI rendering:

**GET** `UMA/uma/apps/{appId}/schemas`
**GET** `UMA/uma/apps/{appId}/permissions`

Permission response:
```json
{
  "data": {
    "role": "admin",
    "permissionsMap": {
      "customers": { "actions": ["READ","WRITE","DELETE"],"scope": "ALL|OWN","visible": true}
    },
    "fullAccess": false,
    "readOnly": false,
    "allowManageUsers": true,
    "allowManagePermissions": false,
    "allowManageSignUpPolicy": false
  }
}
```
- `actions`: `READ`, `WRITE`, `DELETE`
- `scope`: `ALL` (all records) | `OWN` (own records only)
- `fullAccess: true`: full access to all forms. `readOnly: true`: read-only access to all forms, `permissionsMap` is `{}` (empty).
- `visible`: controls whether the form shows in the UI sidebar and dashboard.

**Generate UI dynamically from schema fields + permission map.** If schema changes (new fields, new forms, changed permissions), UI must reflect it without code regeneration.

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
**POST** `UMA/uma/apps/{appId}/form/{formId}/records:query`
```json
{
  "appsId": "my-app",
  "formId": "customers",
  "page": 0,
  "pageSize": 10,
  "sort": [{ "field": "data.name", "direction": "ASC|DESC" }],
  "filter": {
    "operator": "AND",
    "conditions": [
      { "field": "data.age", "operator": "GREATER_THAN", "value": 18 },
      { "field": "data.status", "operator": "EQUALS", "value": "active" }
    ]
  }
}
```
Filter field paths: `data.<fieldId>` for record fields; system fields (`_id`, `createdAt`, etc.) by direct name.

**Filter operators:** `EQUALS`, `NOT_EQUALS`, `GREATER_THAN`, `GREATER_THAN_OR_EQUAL`, `LESS_THAN`, `LESS_THAN_OR_EQUAL`, `IN`, `LIKE`
**Group operators:** `AND`, `OR` — use with `"conditions": [...]` instead of `"field"`/`"value"`

Optional aggregation (replaces items list in response):
```json
{ "aggregation": { "type": "COUNT|SUM|AVG|MIN|MAX", "field": "data.age" } }
```

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

When `true`, all record reads/queries for non-superusers are automatically filtered to `createdBy == currentUserId`. Enables row-level security per user.

---

## Implementation Notes

### SEQUENCE field `length`
Must be the **total length** of the generated output string (prefix + separator + digits). E.g. `STU-0001` = 8 chars → `length: 8`. Setting `length: 4` when prefix+separator already takes 4 chars fails with `SCHEMA_FIELD_TYPE_ERROR`.

### TypeScript `verbatimModuleSyntax`
Always use the `type` keyword for type-only imports: `import { type FormEvent }`, `import { type ReactNode }`.

### The permissions endpoint omits `permissionsMap` entirely for admin/full-access users. Always type it as optional (`permissionsMap?: Record<string, FormPermission>`) and access with optional chaining (`permissions.permissionsMap?.[formId]`). Check `fullAccess` first before any map lookup.

### LINKED/RELATED fields — write vs read
- **Write (create/update):** send a plain ObjectId string in `data`.
- **Read (list/get response):** field comes back as a resolved array `[{ "id": "...", "_formId": "...", ...fields }]`.
- **Edit form pitfall:** `initial.data` is populated from the read response, so LINKED fields already contain resolved arrays. Normalize them back to plain ObjectId strings before submitting, otherwise the PUT sends the full array object instead of an id.

### Update `version` field
Use the record's actual `version` from the GET/list response, not `0`. Sending `0` on update may cause optimistic locking conflicts.

### DATE/TIME write format
Pattern only affects read. Write always ISO: `yyyy-MM-dd`, `yyyy-MM-dd'T'HH:mm:ss`, `HH:mm:ss`. Normalize on edit.

### Use `required` in schema to drive `*` indicators in UI forms.

### On 401 response from UMA, clear auth and redirect to `/login

## Error shape: `{ "error": { "code": "...", "message": "..." } }`. Extract via `err.error.message`, not `err.message`
