# Comments for BK-101

[View in Jira](https://jira.upexgalaxy.com/browse/BK-101)

---

### Automation for Jira - 6/10/2026, 8:37:25 PM

Hola, Carlos Alberto Chiavassa!✨
Espero que te encante trabajar aquí en el ***NUEVO Workspace de UPEX GALAXY***!👾🚀

Esta ***Feature**** ha sido ****deployada**** al Ambiente ****QA**** en el presente ****Sprint***.
Ahora, luego de haber estimado la ***User Story**** en el ****Sprint Planning***, 
***usted has sido asignado a esta US por el Equipo de Desarrollo***, para proceder con la Actividad de Testing correspondiente.
Estamos confiados de que gracias a tus habilidades y conocimientos, podrás cumplir con la entrega!

🚩Para saber cómo realizar esta actividad y más, tienes muchas guías y documentación en ***Confluence*** de UPEX o UPEX Galaxy (importante): 

- [Documentación UPEX-DOCU (Recomendado para leer sobre Buenas Prácticas)](https://docu.upexgalaxy.com)
- [Documentación GALAXY-DOCU (Recomendado para leer el Flujo de Trabajo Paso a Paso)](https://academy.uppexgalaxy.com)

💬 Por favor, comunicate con el Staff de UPEX si sospechas de algún error de acceso o algo que impida tu progreso. Rápidamente lo resolveremos!

✨ AL INFINITO Y MÁS TESTING! 🪐

Saludos,
Atte. 
***Elyer Maldonado***
***Workspace Manager / UPEX's CEO***.

---

### Carlos Alberto Chiavassa - 6/11/2026, 10:44:02 AM

Tus decisiones de PO para BK-101:

1. Cada workspace se muestra con el nombre como título y el rol como subtítulo. Ejemplo: "Acme QA" / "Rol: Admin"
2. El workspace activo se marca con badge "Activo" y borde visual diferenciado
3. Esta US es solo lectura — sin acciones de cambio, salida ni administración de miembros

---

### Carlos Alberto Chiavassa - 6/11/2026, 5:24:45 PM

## QA Session Completed — BK-101

***Verdict******:*** CONDITIONAL PASS
***Tested******:*** 2026-06-11 | Environment: staging | QA: chiavassa-bunkai-qa-engineering

### Results at a glance

| Result | Count |
| --- | --- |
| PASS | 5 |
| BLOCKED (infra/data) | 3 |
| CONDITIONAL PASS | 1 |
| BUGS FILED | 0 |

***Tested scope (API + DB)******:*** passes cleanly. All auth boundaries, RLS guards, and DB consistency checks behave correctly.

### Passed ✅

- P-02 Bearer token workspace list — workspace array complete, all fields present
- N-01 Unauthenticated 401 — both `/me` and `/workspaces` correctly reject
- N-02 Non-member 403 — `active-workspace` correctly blocks and preserves state
- B-01 Oldest workspace fallback — correct (single workspace; needs ≥2 to be non-trivial)
- I-01 DB cross-validation — API set exactly matches DB `workspace_members` active set

### Blocked (no defect)

- P-01, P-03: Magic-link-only auth prevents headless cookie-session testing → needs Playwright+email flow
- N-03: QA bot has no suspended membership in staging → needs DB seed
- N-04: No invited memberships exist in staging → needs seed or dev clarification on invite table

### API Gap (open item for dev)

`role` field not returned by `GET /api/v1/me` — only `owner*user*id` available. Role labels (AC-1) cannot be asserted until API is extended.

### Deferred

All UI outlines deferred — BK-87 (Settings Hub) not deployed to staging as of today.

***Next steps******:*** QA bot infra setup → re-run P-01, P-03, N-03, N-04. BK-87 deploy → re-run all UI outlines. API `role` extension → re-run AC-1 role labels.

---


_Synced from Jira by sync-jira-issues_
