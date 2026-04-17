# UMA Workflow Router

At the start of every session, check the working directory to determine context:

## If an existing app has already been built

Look for any of these indicators:
- A `src/` directory exists
- A `package.json` exists
- Frontend source files are present (`.tsx`, `.ts`, `.jsx`, `.js` project files)

→ **Read `CLAUDE-maintenance.md` and follow those instructions.**

## If this is a new / empty project

No frontend source files or project structure exists yet.

→ **Read `CLAUDE-new-app.md` and follow those instructions.**

---

Do this check automatically at the start of every session before responding to any request.
