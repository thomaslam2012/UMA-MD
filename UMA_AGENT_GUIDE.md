# UMA Agent Guide

**Audience:** AI coding agents (Claude Code, Cursor, etc.)
**Purpose:** Operate UMA autonomously as a semantic data model layer during development.
**Base URL:** `http://{UMA_HOST}:{UMA_PORT}/uma`

---

> ## ⛔ MANDATORY — DO NOT SKIP
>
> Before making any change to any schema, field, or data record — and before writing any plan, proposal, or code — you MUST complete all of the following steps. There are no exceptions.
>
> **Step 1.** Read `uma.env` and authenticate (Section 1.1).
>
> **Step 2.** Call `GET /uma/apps/{appId}/schemas` and read the full response. This is not optional. Do not proceed until you have it.
>
> **Step 3.** For each schema that the user's request touches, call `GET /uma/apps/{appId}/schemas/{formId}` and read:
> - Every field's `fieldId`, `fieldType`, and `semantic.purpose`
> - `semantic.evolutionNotes` at both schema level and on every field
> - `semantic.constraints` at both schema level and on every field
>
> **Step 4.** Internally complete the following Conflict Check. Do not skip it. Do not show it to the user.
>
> ```
> [INTERNAL — NOT SHOWN TO USER]
> Schemas read: [list every schema fetched]
> Fields with protected evolutionNotes: [list each field and its note, or "none"]
> Conflicts found: YES / NONE
> If YES — conflict detail: [explain exactly what conflicts and why]
> ```
>
> ⛔ **HARD STOP — you are not permitted to write a plan, proposal, or any code until the Conflict Check above is complete. No exceptions.**
>
> **Step 5.** If Conflicts found is NONE — proceed silently to Phase 2. Do not mention the conflict check to the user.
>
> **Step 6.** If Conflicts found is YES — show the user only a plain-language warning. Follow these rules strictly:
>
> - No technical field names (`dentist`, `fieldId`, `LINKED`, etc.) — use the display name or plain noun instead
> - No system/code terms (`deprecated`, `field`, `schema`, `link`, `logic`, `per-dentist`, `derive`) — describe the business effect instead
> - No mention of how the system works internally — only describe what the user would see or lose
>
> **Avoid → Use instead:**
> | Avoid | Use instead |
> |-------|-------------|
> | "the dentist field" | "the dentist assigned to each appointment" |
> | "double-booking prevention" | "making sure the same dentist isn't booked twice at the same time" |
> | "per-dentist logic" | "how the system tracks each dentist's schedule" |
> | "field would need to be retired" | "existing appointments would lose their dentist information" |
> | "derive the dentist through the assistant" | "look up the dentist from the assistant's profile" |
> | "linked / direct link" | "connected" or just describe what it does |
> | "existing records" | "appointments already in the system" |
> | "snapshot" | "a saved copy at the time of booking" |
> | "integrity" | describe specifically what breaks (e.g. "so the calendar stays accurate") |
>
> Example format:
>
> > ⚠️ Before I proceed, I need to flag something:
> > Right now, every appointment is connected to a specific dentist — this is how the calendar knows whose schedule to show and how the system prevents the same dentist being booked twice at the same time. Removing this would mean [plain business consequence]. Do you want to proceed?
>
> Wait for explicit user confirmation before continuing.
>
> **Step 7.** Only after the Conflict Check is complete and any conflicts are confirmed may you proceed to Phase 2.
>
> **This applies to ALL requests — new features, edits, refactors, and "quick fixes." Planning from memory or assumptions is not allowed.**

---

> ## ⛔ MANDATORY — BEFORE GENERATING ANY DELETE HANDLER
>
> Before writing any code that deletes a record, you MUST complete all of the following steps. There are no exceptions.
>
> **Step 1.** Fetch all schemas with `GET /uma/apps/{appId}/schemas` and identify every schema that has a LINKED field pointing to the form being deleted.
>
> **Step 2.** If no other schema has a LINKED field pointing to this form — proceed to generate the delete code.
>
> **Step 3.** If any schema has a LINKED field pointing to this form — read the field-level `semantic` on that LINKED field to understand the relationship in domain terms.
>
> **Step 4.** Ask the user in plain language what should happen to those related records. Follow these rules strictly:
> - No technical field names, no system terms (`LINKED`, `field`, `schema`, `formId`, etc.)
> - Describe only the real-world relationship and consequence
> - Use the `semantic.meaning` and `semantic.purpose` of the LINKED field to form the question
>
> Example:
> > *"When a cart is deleted, what should happen to the items inside it — should they be deleted too, or kept?"*
>
> **Step 5.** Wait for explicit user confirmation before generating any delete code.
>
> **Step 6.** Generate the delete logic based on the user's answer:
> - If user says delete them: generate code that deletes child records first, then the parent.
> - If user says keep them: generate code that deletes the parent only.
>
> **Step 7.** Record the user's decision in `semantic.evolutionNotes` on the LINKED field so future agents do not need to ask again.
>
> ⛔ **HARD STOP — you are not permitted to generate any delete handler until this check is complete and the user has confirmed the intended behavior. No exceptions.**

---

## Table of Contents

- [What UMA Is](#what-uma-is)
- [1. Authentication](#1-authentication)
  - [Making curl API Calls — Required Format](#making-curl-api-calls--required-format)
  - [Before You Start — Resolve Configuration Values](#before-you-start--resolve-configuration-values)
  - [1.1 Development-Time Auth](#11-development-time-auth-ai-agent--uma)
  - [1.2 Runtime Auth](#12-runtime-auth-generated-app--uma-app)
    - [1.2.1 Sign Up](#121-sign-up)
    - [1.2.2 Verify Email](#122-verify-email)
    - [1.2.3 Resend Verification Email](#123-resend-verification-email)
    - [1.2.4 Password Reset Request](#124-password-reset-request)
    - [1.2.5 Password Reset](#125-password-reset)
    - [1.2.6 Login](#126-login)
- [2. Reference](#2-reference)
- [3. Error Handling](#3-error-handling)
- [7. AI Agent Workflow](#7-ai-agent-workflow)
  - [7.0 Master Build Workflow](#70-master-build-workflow)
    - [Phase 1.5 — Read Existing Data Model](#phase-15--read-existing-data-model-existing-projects-only)
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

### Making curl API Calls — Required Format

Claude Code blocks any `Bash` command that contains literal newlines, even with pre-approved permissions. **Never inline multiline JSON in a curl `-d` flag.** Instead, always write the JSON body to a temp file first:

```bash
cat > /tmp/uma_payload.json << 'ENDJSON'
{
  "appId": "MyApp",
  "formId": "Customer"
}
ENDJSON
curl -s -X POST "http://$UMA_HOST:$UMA_PORT/uma/apps/MyApp/schemas/Customer" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data @/tmp/uma_payload.json
```

This pattern avoids the newline safety prompt on every API call. Use `/tmp/uma_payload.json` for every request, overwriting it each time.

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

**204 No Content:** Sections 1.2.2, 1.2.3, 1.2.4, and 1.2.5 return `HTTP 204` with no response body on success. Do not call `res.json()` on these responses — guard against it: `if (res.status === 204) return {}`.

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

## 2. Reference

For full API endpoint details, field types, field attributes, and semantic metadata rules, read **`UMA_REFERENCE.md`**.

Read `UMA_REFERENCE.md` when you need to:
- Make a specific API call and need the exact request/response format (Section 2)
- Choose a field type for a schema field (Section 4)
- Configure field attributes (Section 5)
- Understand semantic metadata structure and evolution rules (Section 6)

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
| 400 | `INVALID_FORMAT` | Value does not match expected format | Check field type format rules in UMA_REFERENCE.md Section 5. |
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
| 400 | `SCHEMA_DATE_ERROR` | DATE field `datePattern` value is invalid | Use a valid `datePattern` value from UMA_REFERENCE.md Section 5.4. |
| 400 | `SCHEMA_DATE_TIME_ERROR` | DATE_TIME field `dateTimePattern` value is invalid | Use a valid `dateTimePattern` value from UMA_REFERENCE.md Section 5.5. |
| 400 | `SCHEMA_TIME_ERROR` | TIME field `timePattern` value is invalid | Use a valid `timePattern` value from UMA_REFERENCE.md Section 5.6. |
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

## 7. AI Agent Workflow

### 7.0 Master Build Workflow

When the user gives a prompt to build any app or software, execute the following sequence in order. The sections referenced in each step provide the detailed rules.

---

**Phase 1 — Setup**

1. Read `uma.env` from the project root (Section 1 "Before You Start"). If the file is missing or any required key is absent, stop and ask the user before proceeding.
2. Authenticate with UMA-Dashboard (Section 1.1) → store `devToken` and `tenantId`.
3. Check `uma.env` for `UMA_APP_ID` and `UMA_TENANT_ID`:
   - **If both are present:** This project has an existing app. Use `UMA_APP_ID` as the `appId` and `UMA_TENANT_ID` as the `tenantId` for all subsequent operations. Skip app creation in Phase 3. **Before doing anything else, read the existing data model** (Phase 1.5 below).
   - **If absent:** This is a new project. Proceed through all phases normally.

---

**Phase 1.5 — Read Existing Data Model (existing projects only)**

**Do this before planning any change.** Never propose modifications without first knowing what already exists in UMA.

```
GET http://{UMA_HOST}:{UMA_PORT}/uma/apps/{appId}/schemas
```

For each schema returned:
- Read all field `fieldId`, `fieldType`, and `semantic.purpose` values.
- Read `semantic.evolutionNotes` at both schema and field level.
- Note which fields are `deprecated: true` (do not include in plans — they are soft-deleted).
- Note LINKED and RELATED relationships between schemas.

Only after reading all existing schemas, proceed to Phase 2 with full knowledge of the current state. **Do not design or propose any changes based on assumptions — base everything on what the GET returns.**

---

**Phase 2 — Design the Data Model**

4. Interpret the user prompt to identify entities, relationships, and field types (Section 7.2). Read `UMA_REFERENCE.md` Section 4 and 5 for field type options and attributes.
5. For each entity, design its schema: choose appropriate `fieldType` for each field and plan a fully populated `semantic` block at both schema and field level (Section 7.3). Never leave `semantic` empty.
   - For **existing schemas**: plan only the delta — new fields to add, fields to modify, semantics to update. Do not re-create schemas that already exist.
   - For **new schemas**: design the full schema from scratch.
6. Identify LINKED/RELATED dependencies and determine creation order — referenced schemas must exist before schemas that link to them.

---

**Phase 2.5 — Present Data Model for User Confirmation**

Before creating anything in UMA, present the designed data model to the user for confirmation. **Do not show raw JSON.** Display it as human-readable tables so non-technical users can understand and verify it.

**Language rules — translate all technical terms to business language:**

| Technical term | Use instead |
|----------------|-------------|
| `SEQUENCE` | Auto-generated ID |
| `TEXT` | Text |
| `BOOLEAN` | Yes/No |
| `LINKED → FormName` | Links to [plain name] (e.g., "Links to Dentist") |
| `RELATED` | Auto-populated from [plain name] |
| `EMBED` | Inline group of fields |
| `MONEY` | Currency amount |
| `DATE_TIME` | Date and time |
| `EMAIL` | Email address |
| `URL` | Web link |
| `FILE` | File attachment |
| `UNIQUE` | Auto-generated unique code |
| `allowMultiple: true` | Can select multiple |
| `validateRequired: true` / Required: Yes | Required |
| `deprecated` | Removed |
| schema / form | [entity name] (e.g., "Patient record", "Appointment") |

Never use `fieldId`, `formId`, `appId`, `fieldType`, or any JSON key names when talking to the user.

For each schema, show:

```
## [Plain entity name — e.g. "Appointment" not "AppointmentForm"]
What it stores: [semantic.purpose in one plain sentence]

| Field          | Type                    | Required | What it's for                         |
|----------------|-------------------------|----------|---------------------------------------|
| Product Code   | Auto-generated ID       | No       | System ID assigned automatically      |
| Product Name   | Text                    | Yes      | The name of the product               |
| Price          | Currency amount         | Yes      | Selling price in USD                  |
| Category       | Links to Category       | Yes      | The category this product belongs to  |
```

Use the `fieldDisplays.EN` value as the field name in the table, not the `fieldId`.

Then ask:
> *"Here is the data model I've designed for your app. Does this look right? Let me know if you'd like to add, remove, or change anything before I build it."*

Only proceed to Phase 3 after the user confirms. If the user requests changes, update the design and re-present the tables before asking again.

---

**Phase 3 — Build the Data Model in UMA**

7. `POST /uma/apps` — create the app with full `semantic`. If it already exists (`DUPLICATED_APPS`), use the existing app. Read `UMA_REFERENCE.md` Section 2.1 for the exact request format.
   - **After the app is successfully created**, write `UMA_TENANT_ID` and `UMA_APP_ID` into `uma.env`:
     ```
     UMA_TENANT_ID=tnt_1f8e2jur
     UMA_APP_ID=ProductManagement
     ```
     This ensures future sessions know which app belongs to this project and do not create a new one.
8. `POST /uma/apps/{appId}/schemas/{formId}` — create each schema with full `semantic` on the schema and on every field. Follow the creation order from step 6. Read `UMA_REFERENCE.md` Section 2.2 for the exact request format.
9. Insert test data to verify the schema works (Section 7.5).
10. Record what was built in `semantic.evolutionNotes` at schema and app level (Section 7.8).

---

**Phase 4 — Determine Runtime Auth**

11. Determine whether the app requires end-user signup and login:
    - **If the user prompt explicitly mentions login, signup, or user accounts:** Implement UMA-APP signup and login in the generated code (Sections 1.2.1–1.2.6). The token from the login response (`data.token`) is used as the Bearer token for all runtime `/uma/*` calls.
    - **If the user prompt explicitly says no login is needed:** Follow the warning and confirmation flow in Section 1.2. Do not embed credentials until the user explicitly confirms. If confirmed, generate code that fetches a dev-time token on load and add a security warning comment in the code.
    - **If the user prompt does not mention login or signup at all:** Always ask before proceeding:
      > *"Does your app need users to sign up and log in? UMA-APP supports this automatically with no extra setup required. If you say no, I'll need to embed credentials directly in the code which is not secure for production use."*
      Wait for the user's answer before continuing.

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
13. Implement UI or handlers for all entities: list, create, edit, delete, as appropriate for the domain. For delete handlers, follow the mandatory delete check above.
14. Use `POST .../records:query` for any search, filter, or paginated views (Section 7.6). Read `UMA_REFERENCE.md` Section 2.3 for query format details.

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
3. **Check `evolutionNotes` on relevant schemas and fields.** If the request violates any notes, prompt the user before continuing (see `UMA_REFERENCE.md` Section 6.3).
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

**`evolutionNotes` must be specific and protective — not generic.** Vague notes like "do not modify without care" are useless. Every schema's `evolutionNotes` must explicitly answer:

1. **Which fields are workflow-critical?** Name them and explain why. Example: *"appointmentDate is required by the calendar view — do not remove or rename. status drives the booking state machine — do not change its allowed values without updating the UI."*
2. **Are new fields safe to add freely?** State whether future fields must be optional. Example: *"New fields must not use validateRequired: true — existing records will not have them and will fail validation."*
3. **What schemas depend on this one?** Name any LINKED or RELATED relationships. Example: *"Appointment schema has a LINKED field pointing to this schema — changes to formId or key field names will break that link."*
4. **What changes require user confirmation?** Be explicit. Example: *"Do not deprecate patientId — it is the join key used across three schemas. Do not change roomId fieldType — it is SEQUENCE and immutable."*

Apply the same standard to individual field `evolutionNotes` for any field that is a key, a join reference, or drives UI or workflow logic.

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
1. REQUIRED FIRST STEP — GET /uma/apps/{appId}/schemas/{formId}
   Do not proceed until you have the full response in hand.
   Do not rely on memory from earlier in the session — always fetch fresh.
   Read semantic.evolutionNotes at schema level and on every field.
   Read semantic.constraints at schema level and on every field.
   If you skip this step and make changes based on assumptions,
   you risk silently breaking existing data and workflows.

2. Internally complete the Conflict Check (do not show to user):

   [INTERNAL — NOT SHOWN TO USER]
   Schemas read: [list]
   Fields with protected evolutionNotes: [list each field and its note, or "none"]
   Conflicts found: YES / NONE
   If YES — conflict detail: [explain]

   ⛔ HARD STOP — do not proceed until this check is complete.

3. If Conflicts found is NONE — proceed silently. Do not mention the conflict check.
4. If Conflicts found is YES — show the user only a plain-language warning. No technical field names, no system terms, no internal notes. Describe only what the user would see or lose in business terms. Follow the language rules in Step 6 of the mandatory block above.

   Wait for explicit user confirmation before continuing.
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

Read `UMA_REFERENCE.md` Section 2.3 for full query, filter, sort, and aggregation format.

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
