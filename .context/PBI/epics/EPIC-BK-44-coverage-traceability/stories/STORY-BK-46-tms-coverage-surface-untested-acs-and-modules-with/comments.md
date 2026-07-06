# Comments for BK-46

[View in Jira](https://jira.upexgalaxy.com/browse/BK-46)

---

### Carlos Alberto Chiavassa - 6/26/2026, 7:08:00 AM

## Shift-Left QA — Refinement completado (2026-06-26)

Análisis pre-sprint realizado. Artefacto local:
`.context/PBI/epics/EPIC-BK-44-coverage-traceability/stories/STORY-BK-46-surface-untested-acs/shift-left-refinement.md`

***Resultado:*** 3 ACs originales expandidos a 9 sub-scenarios. 20 outlines ATP DRAFT identificados.
Infraestructura faltante: coverage view route + API endpoint (ambos nuevos). Data model del SUT soporta la feature.

---

### 3 preguntas bloqueantes — requieren respuesta del PO antes de iniciar dev

***Q1 — ¿Qué significa "never run"?***
¿Es `atcs.status = 'unrun'` (estado puntual en DB)? ¿O se necesita una tabla de historial de ejecuciones?
Si un ATC corrió, fue editado y se reseteó a `unrun`, ¿sigue apareciendo en el filtro "not run"? Impacta la query central de AC2.

***Q2 — ¿"Fully covered" = ATC vinculado, o ATC vinculado Y ejecutado?***
AC3 dice "executed test coverage." ¿Un módulo donde todos los ACs tienen ATCs pero todos están en `unrun` es "fully covered"?
O ¿deben tener `status ≠ unrun`? Impacta el threshold del indicador de AC3.

***Q3 — Multi-ATC por AC en filtro "not run": ¿unión o intersección?***
Si un AC tiene 2 ATCs y uno es `pass` y otro es `unrun`, ¿aparece en el filtro?
Propuesta QA: NO (si al menos uno corrió, el AC tiene cobertura ejecutada).
Impacta el predicado SQL del filtro de AC2.

---

6 preguntas adicionales (MEDIO/BAJO) documentadas en el artefacto local.
ATP DRAFT completo en campo Acceptance Test Plan (ATP) de este ticket.


---

### Carlos Alberto Chiavassa - 6/27/2026, 6:09:31 PM

PO Decisions — Q1, Q2, Q3 resolved (2026-06-27)

Q1 — "Not run" definition
"Not run" = atcs.status = 'unrun' (current point-in-time value). Distinguishing "never executed historically" from "reset to unrun" is out of MVP scope; if needed, a separate execution-history story.

Q2 — "Fully covered" definition
"Fully covered" = ATC linked AND executed (status != 'unrun'). A module with all ACs linked to unrun ATCs is NOT fully covered. Note: "covered" (was it run?) and "healthy" (did it pass?) are separate axes — this view measures coverage only.

Q3 — Multi-ATC per AC: union rule
An AC with N ATCs appears in "not run" if at least ONE linked ATC is 'unrun'. One executed ATC does not neutralize pending coverage on the others.

Story unblocked for estimation. Q4-Q9 remain open.

---


_Synced from Jira by sync-jira-issues_
