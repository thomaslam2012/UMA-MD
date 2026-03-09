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
