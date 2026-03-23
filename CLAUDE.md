# Project Instructions

<!--
  DO NOT run /init on this project.
  Running /init will overwrite this file and the agent will lose the UMA guide.
  If this file is accidentally overwritten, copy CLAUDE.md from the UMA-MD template folder.
-->

This project uses UMA as the backend data layer.

**STOP — before any planning or coding:**
For every user request that involves schemas, data, fields, or features:
1. Read `uma.env` to get `UMA_APP_ID`.
2. Call `GET /uma/apps/{UMA_APP_ID}/schemas` — read the full response.
3. For each schema the request touches, call `GET /uma/apps/{UMA_APP_ID}/schemas/{formId}` and read all `semantic.evolutionNotes` and `semantic.constraints` at schema and field level.
4. Only then propose a plan. Never plan from memory or assumptions.

Read and follow the full guide:

@UMA_AGENT_GUIDE.md

Never hardcode any host, port, or credential values in generated code.

## Semantic Quality Standard

⛔ You are not permitted to submit any `POST` or `PUT` for an app, schema, or field without semantics that answer what breaks without it — not just what it is. Before submitting, apply the questions in Section 7.3 of `UMA_AGENT_GUIDE.md` for that level. Vague notes like "do not modify without care" or "used by the system" must be rewritten before submitting. This is not optional — without high-quality semantics, every future safety check UMA performs is blind.

## React + TanStack Query — Client Cache Security Rules

**Apply these rules only when generating a React frontend that uses TanStack Query.**

TanStack Query's in-memory cache persists across user sessions within the same browser tab. If cache is not cleared on logout, a subsequent user will see the previous user's data even if the API correctly returns 403.

### Rule 1 — Always clear the query cache on logout
Every logout handler must call `queryClient.clear()` before navigating away.
```ts
function handleLogout() {
  logout()
  queryClient.clear()
  navigate('/login')
}
```

### Rule 2 — Clear cache on 401/403 via global interceptor
Implement in the Axios interceptor, not in individual query handlers.
```ts
axiosInstance.interceptors.response.use(
  (res) => res,
  (error) => {
    if (error.response?.status === 401 || error.response?.status === 403) {
      logout()
      queryClient.clear()
      navigate('/login')
    }
    return Promise.reject(error)
  }
)
```

### Rule 3 — Never silently show stale data on query error
Always check `isError` and render an error state — never fall back to stale cached data silently.

### Rule 4 — `queryClient` must be accessible at logout time
Use `useQueryClient()` in the component that owns logout, or pass it into the logout utility. Never call `logout()` without also calling `queryClient.clear()`.

**Checklist before marking any auth feature complete:**
- [ ] `queryClient.clear()` is called in every logout handler
- [ ] A global 401/403 interceptor calls `logout()` + `queryClient.clear()` + redirects to `/login`
- [ ] Every `useQuery` call has an `isError` branch
- [ ] No logout path exists that skips cache clearing

## React + TanStack Table — Records List Patterns

**Apply these rules only when generating a React frontend that uses TanStack Table.**

Always use the query endpoint (`POST .../records:query`) for record lists — not the basic GET list endpoint — because it supports sort, filter, and pagination in the request body.

**TanStack Table must be set to manual mode** — the server handles all data operations:
```ts
manualPagination: true
manualSorting: true
manualFiltering: true
```

**Page reset** — reset to page 0 whenever sort, filter, or deprecated toggle changes:
```ts
useEffect(() => { setPage(0); }, [sorting, globalFilter, showDeprecated]);
```

**Sort mapping** — TanStack Table `SortingState` maps to the API `sort` array:
```ts
const apiSort = sorting.map((s) => ({
  field: s.id,                        // column id must equal fieldId
  direction: s.desc ? 'DESC' : 'ASC',
}));
```

**TanStack Query key** — include all variables that affect the result so any change triggers a refetch:
```ts
queryKey: ['records', appId, formId, showDeprecated, page, apiSort, globalFilter]
```

## React Frontend — Date / Time Field Handling

**Apply these rules only when generating a React frontend that handles DATE, DATE_TIME, or TIME fields.**

### Rendering (read-only display)

| Condition | Display value |
|---|---|
| Field has a pattern | Show value as-is — server already returns it in pattern format |
| No pattern DATE | Show raw ISO: `"2026-03-17"` |
| No pattern DATE_TIME | Show first 16 chars: `"2026-03-17T14:30"` |
| No pattern TIME | Show raw ISO: `"14:30:00"` |

Do not reformat patterned values — the server already returns them formatted.

### Form editing (create / edit)

Use a **text input + hidden native date picker** — do not use `<input type="date">` directly. It renders in browser locale format and ignores the field's pattern.

```
[ text input — shows value in display format ]  [ calendar icon button ]
         hidden <input type="date/datetime-local/time">
         triggered via ref.current?.showPicker()
```

Show a placeholder matching the field's configured format so the user knows what to type:

| Pattern | Placeholder |
|---|---|
| `SLASH_DATE` | `yyyy/MM/dd` |
| `DASH_DATE` | `dd-MM-yyyy` |
| `LONG_DATE` | `Weekday, Mon dd, yyyy` |
| `DATE_TIME_AMPM_1` | `yyyy-MM-dd hh:mm AM/PM` |
| `HOURS_MINUTES_AMPM` | `hh:mm AM/PM` |
| no pattern DATE | `yyyy-MM-dd` |
| no pattern DATE_TIME | `yyyy-MM-ddTHH:mm` |
| no pattern TIME | `HH:mm:ss` |

### Saving (create / update)

Always convert display-format values back to ISO before sending to the server:

```ts
// DATE → yyyy-MM-dd
payload[fieldId] = toHtmlDate(formValue, datePattern);

// DATE_TIME → yyyy-MM-ddTHH:mm:ss
const hv = toHtmlDateTime(formValue, dateTimePattern);
payload[fieldId] = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/.test(hv) ? `${hv}:00` : hv;

// TIME → HH:mm:ss
const hv = toHtmlTime(formValue, timePattern);
payload[fieldId] = /^\d{2}:\d{2}$/.test(hv) ? `${hv}:00` : hv;
```

### Search / filter (date range)

Use `GREATER_THAN_OR_EQUAL` / `LESS_THAN_OR_EQUAL` operators. Send values as-is in the field's display format — matching how the server stores them:

```ts
{ operator: 'GREATER_THAN_OR_EQUAL', field: fieldId, value: dateFrom }
{ operator: 'LESS_THAN_OR_EQUAL',    field: fieldId, value: dateTo   }
```

## React Frontend — LINKED and RELATED Field UI Rules

**Apply these rules only when generating a React frontend.**

### RELATED fields — never show in create/edit forms

RELATED fields are server-computed back-references. The user cannot set them — they are derived automatically. Never render a RELATED field as an input in create or edit forms. Only show them in read-only table or detail views.

In the API response, RELATED fields appear as an **array** of back-referenced records — always iterate over the array when rendering, never treat it as a single value.

### LINKED fields — id is always available

When using `fieldMode: PARTIAL`, the `id` of the referenced record is always returned even if not in `selectedFieldIds`. Use this `id` to fetch the full record when needed — you never lose access to the complete linked record.
