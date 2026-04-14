# Agent instructions (Cursor / AI)

This repository implements **UMA-backed frontends** using a **domain-agnostic** specification.

1. Read **`CLAUDE.md`** first — it defines phases, APIs, and required UI behavior for **any** app domain.
2. Follow **`.cursor/rules/`** — workflow gates and frontend implementation rules apply to **every** session.
3. Use **`UMA-COMPLIANCE-CHECKLIST.md`** as the completion gate for Phase 3: report **Done / N/A** for each applicable row before stating work is complete.
4. **Do not** assume a specific industry or hardcode business entities; derive screens from **`schemas`** and **`permissions`** at runtime.
