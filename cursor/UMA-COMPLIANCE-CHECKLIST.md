# UMA frontend compliance checklist (domain-agnostic)

Use this with **`CLAUDE.md`** and **`.cursor/rules/`**. Mark each row **Done**, **N/A** (with reason), or **TODO**. Agents **MUST** report status for every applicable row before claiming Phase 3 complete.

**Domain:** Any. Do not assume a specific business; schemas define entities.

---

## A. Workflow (`CLAUDE.md` — Execution Guardrails + phases)

| # | Requirement |
|---|-------------|
| A1 | Phase 1 → 2 → 3 order; API key obtained before code; no frontend scaffold until app + schemas exist in UMA |
| A2 | Phase output summarized; explicit user approval before next phase |

---

## B. Runtime data loading (after app-user login)

| # | Requirement |
|---|-------------|
| B1 | `GET .../apps/{appId}/schemas` on each login (or session start) |
| B2 | `GET .../apps/{appId}/permissions` on each login |
| B3 | UI derived from responses — no hardcoded form lists / field lists / nav items for app data |

---

## C. Permissions mapping

| # | Requirement |
|---|-------------|
| C1 | Sidebar/nav: `visible`; handle `fullAccess` (no `permissionsMap`); handle `readOnly` |
| C2 | List requires READ; Create/Edit requires WRITE; Delete requires DELETE |
| C3 | `permissionsMap` typed optional; safe access when absent |
| C4 | If UI exposes them: respect `allowManageUsers`, `allowManagePermissions`, `allowManageSignUpPolicy` |

---

## D. Field type rendering (create/edit + display)

| # | Requirement |
|---|-------------|
| D1 | TEXT — text input; honor `length` / `regexPattern` if enforcing client-side |
| D2 | NUMERIC — numeric input; `decimalPlaces` / `range` if enforcing |
| D3 | BOOLEAN — toggle or checkbox |
| D4 | DATE / DATE_TIME / TIME — date/time pickers; writes ISO per spec |
| D5 | EMAIL / URL — appropriate input types |
| D6 | MONEY — currency UI with symbol prefix; read/write `centAmount` object shape |
| D7 | FILE — dropzone; max 5 MB; base64 + contentType payload |
| D8 | SEQUENCE / UNIQUE — readonly; never editable |
| D9 | LINKED / RELATED — record picker for `refFormId`; write id string; read resolves arrays; normalize before PUT |
| D10 | USER_ID — readonly + lookup; IAM users list; picker shows first+last name; store userId; expanded object on read in tables/forms |
| D11 | EMBED — nested inline form for `embeddedFormSchema` |
| D12 | `required` / `fieldDisplays` drive labels and `*` |

---

## E. UMA-APP-AUTH (if implemented in UI)

| # | Requirement |
|---|-------------|
| E1 | Login: `POST .../auth/sessions` with `tenantId`, `appId` |
| E2 | Sign-up: policy `GET` first; block if `CLOSE`; `DOMAIN` email rules |
| E3 | Sign-up user create; verification code; resend |
| E4 | Password reset request + reset with code |
| E5 | All auth requests include `tenantId` + `appId` where required |

---

## F. CRUD + errors

| # | Requirement |
|---|-------------|
| F1 | Create/list/get/update/delete use correct paths and bodies per `CLAUDE.md` |
| F2 | Updates send correct `version` from server |
| F3 | Parse errors via `error.message` inside `error` object |
| F4 | On UMA 401: clear token/session and redirect to `/login` |
| F5 | List responses: handle `{ items, count, totalCount, page, pageSize }` (and wrapped variants if API uses them) |

---

## G. Optional / feature-gated

| # | Requirement |
|---|-------------|
| G1 | If UI offers filters: `POST .../records:query` with documented filter operators |
| G2 | If UI offers bulk delete: `DELETE .../records` with `objectIds` |
| G3 | `ownerScoped` documented for testers; behavior matches UMA row-level rules |

---

## H. TypeScript / code quality

| # | Requirement |
|---|-------------|
| H1 | `import type` for type-only imports |

---

## Phase 2 (schema design) — when agents touch UMA schemas

| # | Requirement |
|---|-------------|
| P1 | `semantic` populated on app, each schema, and each field (not empty) |
