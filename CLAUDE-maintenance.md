# UMA Maintenance Workflow

For adding features, fixing bugs, and modifying existing apps. The app, schemas, and auth flows already exist — focus only on what changed.

## CRITICAL RULES — Read First

- **NEVER read frontend code to answer questions about the system, data model, or business logic.** Always go to UMA semantic fields first.
- **NEVER make a data model change without completing the Semantic Pre-Check below.**
- For any UMA call, prompt for the API key first if not already obtained. Do not proceed without it.

## Base URLs
```
UMA           = http://localhost:8080
UMA-DASHBOARD = http://localhost:8081
UMA-APP-AUTH  = http://localhost:8082
```

## App Identity
`appId` and `tenantId` are embedded in the frontend as env vars or constants — read them from the codebase (`.env` or a constants file) before making any UMA call. Do not ask the user for them.

---

## Auth
If any UMA API call is needed, prompt the user for their API key first, then:

**POST** `UMA-DASHBOARD/uma-dashboard/auth/sessions`
```json
{ "authType": "API_KEY", "apiCredentials": "<api_key>" }
```
Store `jwt`, `tenantId`. Use `Authorization: Bearer <jwt>` on all UMA calls. Once obtained, reuse for the rest of the conversation — do not ask for the API key again.

---

## Semantic as Source of Truth

`semantic` is the primary source of truth for business rules and data model intent — always consult it first.

**NEVER read frontend code to answer questions about the system, data model, or business logic.** Always fetch `semantic` from UMA first — this is mandatory, not optional. Frontend code is for confirmation only if semantic fields are unclear or incomplete. Before making any UMA call, prompt for the API key if not already obtained. Do not proceed without it.

**GET** `UMA/uma/apps/{appId}/schemas/{formId}` — read `semantic` at form level and on each field.

---

## Semantic Pre-Check (MANDATORY before any data model change)

Before modifying any schema or field (add / edit / delete / deprecate):

1. **Fetch schema:** `GET UMA/uma/apps/{appId}/schemas/{formId}`
2. **Read `semantic`** at form level and on every affected field. Check the proposed change against:
   - `constraints` — does it violate a stated rule?
   - `behavior` — does it alter expected behavior?
   - `intent` / `purpose` / `meaning` — does it contradict business intent?
   - `evolutionNotes` — does it conflict with documented future direction?
3. **If deleting a form:** fetch all schemas (`GET UMA/uma/apps/{appId}/schemas`) and scan every field across all forms for `LINKED` or `RELATED` fields where `refFormId` matches the target form. If any are found, STOP and warn the user — deleting will break those references. List the affected forms and fields explicitly.
4. **Always report what was found in the semantic fields** — whether or not there is a conflict. Show the relevant `intent`, `constraints`, `behavior`, `purpose`, and `evolutionNotes` to the user.
5. **If conflict:** state exactly which semantic property conflicts and why. Offer at least one alternative. Do not add opinions beyond what the semantic fields say. Wait for user confirmation before proceeding.
6. **If no conflict:** state that the semantic fields show no conflict and it is safe to proceed. Wait for user confirmation before making any change.
7. **Never make a data model change without explicit user confirmation** — even if there is no conflict.

Never skip this check.

---

## Data Model Changes

### Schema Operations
- List: `GET UMA/uma/apps/{appId}/schemas?includeDeprecated=false`
- Update: `PUT UMA/uma/apps/{appId}/schemas/{formId}` — same body as create
- Delete (soft): `DELETE UMA/uma/apps/{appId}/schemas/{formId}`
- Restore: `PATCH UMA/uma/apps/{appId}/schemas/{formId}/restore`

### Field Operations
- Delete (soft): `DELETE UMA/uma/apps/{appId}/schemas/{formId}/fields/{fieldId}`
- Restore: `PATCH UMA/uma/apps/{appId}/schemas/{formId}/fields/{fieldId}/restore`

### Field Types (reference)
`TEXT` `NUMERIC` `BOOLEAN` `DATE` `DATE_TIME` `TIME` `EMAIL` `URL` `MONEY` `FILE` `SEQUENCE` `UNIQUE` `LINKED` `RELATED` `EMBED` `USER_ID`

Key rules:
- `SEQUENCE` / `UNIQUE` `length` = total output string length (prefix + separator + digits)
- `LINKED` `selectedFieldIds` must be empty when `fieldMode: FULL`
- Always populate `semantic` on new schemas and fields

---

## CRUD

- Create: `POST UMA/uma/apps/{appId}/form/{formId}` → `{ appId, formId, version: 0, data: {...} }`
- Get: `GET UMA/uma/apps/{appId}/form/{formId}/records/{recordId}`
- List: `GET UMA/uma/apps/{appId}/form/{formId}?page=0&pageSize=10`
- Update: `PUT UMA/uma/apps/{appId}/form/{formId}/records/{recordId}` → use actual `version` from record
- Delete: `DELETE UMA/uma/apps/{appId}/form/{formId}/records/{recordId}`
- Query: `POST UMA/uma/apps/{appId}/form/{formId}/records:query`

Query body supports `filter`, `sort`, and `aggregation`. Filter field paths: `data.<fieldId>` for record fields, direct name for system fields (`_id`, `createdAt`, etc.).

---

## Permissions

`GET UMA/uma/apps/{appId}/permissions`

- `actions`: READ, WRITE, DELETE
- `scope`: ALL | OWN
- `visible`: controls sidebar/nav display
- `fullAccess` / `readOnly`: override all per-form actions
- `allowManageUsers` / `allowManagePermissions` / `allowManageSignUpPolicy`

Rules: show nav only if `visible: true`, Create/Edit only if WRITE, Delete only if DELETE, List only if READ.

---

## UI Field Rendering

Never render all fields as plain text inputs:

| Type | Render as |
|---|---|
| BOOLEAN | Toggle / checkbox |
| DATE / DATE_TIME / TIME | Date-time picker |
| MONEY | Currency input with symbol prefix |
| FILE | Dropzone, max 5 MB |
| SEQUENCE / UNIQUE | Readonly — system-generated, never editable |
| USER_ID | Readonly input + lookup icon → user picker (`GET UMA/uma/apps/{appId}/users`). Write: userId string. Read: `{ userId, firstName, lastName, email, status }` |
| LINKED / RELATED | Record picker. Write: plain ObjectId. Read: resolved array. On edit, normalize back to ObjectId before submit. |
| EMBED | Inline nested form section |

Use `required` on fields to show `*` in forms.

---

## Design Patterns

### Enum / Dropdown Fields
UMA has no native enum field type. For fields with a fixed set of allowed values, two approaches:

1. **Static (stable values):** Use `TEXT` + `regexPattern` for backend enforcement + `semantic.constraints` for documentation. Frontend renders a `<select>` driven by parsing `semantic.constraints`.

2. **Dynamic (user-manageable values):** Create a dedicated lookup form (e.g. `activity-types`) with a `name` field, seed it with initial records, and change the referencing field from `TEXT` to `LINKED`. Users manage options by adding/removing records in the lookup form. No schema change needed to add new options.

Prefer the **LINKED lookup table** approach when options are expected to grow or need to be managed by end users.

---

## Gotchas

- **version on update:** use actual record `version`, never `0`
- **LINKED on edit:** read response is a resolved array — extract `.id` before PUT
- **DATE write:** always ISO (`yyyy-MM-dd`, `yyyy-MM-dd'T'HH:mm:ss`, `HH:mm:ss`) regardless of display pattern
- **401:** clear auth and redirect to `/login`
- **Error shape:** `{ "error": { "code": "...", "message": "..." } }` — use `err.error.message`, not `err.message`
- **TypeScript:** `import { type X }` for type-only imports
- **User profile:** `GET UMA/uma/apps/{appId}/user` — unwrap `{ "data": {...} }` before use
