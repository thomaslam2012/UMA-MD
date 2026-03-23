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
