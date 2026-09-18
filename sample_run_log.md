# Sample run log — real output, not illustrative

Both examples below are real outputs from the workflow running against live logs pulled from Cloudlab-1 (`kubectl logs`, Open5GS Core + srsRAN RAN, namespace `open5gs`). Nothing here is synthetic or hand-written to look plausible.

## Run 1 — early test, base workflow (2026-08-13, `--since=1h`)

This run was used to validate the agent's reasoning before the ServiceNow/S3 extension existed. Output was written to `ai_log_analysis_report.md` on Cloudlab-1 via the "Append report (SSH)" node.

```json
{
  "severity": "WARNING",
  "component": "AMF",
  "event_summary": "AMF no pudo resolver el host 'lab-open5gs-scp-sbi' al arrancar (04:36:54), reintentó registro contra el NRF y se recuperó solo en ~21s (04:37:15).",
  "probable_root_cause": "Condición de carrera en el arranque: el pod de AMF llegó a estar Ready antes de que el Service DNS del SCP estuviera resoluble (típico cuando no hay initContainer/readiness gate entre AMF y SCP). No hubo impacto a usuarios.",
  "recommended_action": "Si el patrón se repite en cada restart, agregar un initContainer o readiness probe en el deployment de AMF que espere a que el Service DNS de SCP responda antes de iniciar el proceso. No es urgente: se autorresuelve por retry/backoff.",
  "requires_human_approval": true
}
```

The agent also flagged, in its own free-text reasoning before the structured output, that AMF and MongoDB showed exactly 15 restarts each in the same window — a correlation worth checking manually, even though it didn't rise to a reportable severity on its own. MME and RAN logs in the same window showed only normal 3GPP lifecycle procedures (Attach request/complete, Mobile Reachable timer expiry, Implicit Detach) — correctly *not* flagged.

## Run 2 — full end-to-end, extended workflow (2026-08-14, `--since=24h`)

This run exercised the complete pipeline: agent → `Edit Fields` (shared `report_key`) → ServiceNow incident creation → S3 evidence upload via Floci.

```
## WARNING - MME - 2026-08-13T01:37:16.093-04:00
**Resumen:** Flap de conexión S1AP seguido de procedimiento Attach/Detach normal;
eNB se desconectó en 10.42.1.10, posteriormente se reconectó desde 10.42.1.15.
UE 999700000000002 ejecutó Attach exitoso, transitó a ECM-IDLE tras UE Context
Release, y fue finalmente desconectado por expiración de Implicit Detach Timer
(26 minutos tras Mobile Reachable timer).

**Causa raíz probable:** El MME registró 'connection refused' desde
eNB-S1[10.42.1.10] a las 23:24:50, indicando que el eNB/pod previo cerró o no
estaba disponible (reinicio/redeploy NFV típico). Inmediatamente después (5
segundos) el eNB se reconectó desde una nueva IP (10.42.1.15), evidenciando un
flap de la interfaz S1-MME. El procedimiento 3GPP Attach/Detach subsecuente es
completamente normal: ECM-IDLE tras inactividad, Mobile Reachable Timer (T3413)
y Implicit Detach Timer (T3422) — sin evidencia de fallo de autenticación, S6a,
S11 o plano de usuario.

**Acción recomendada:** Investigar el evento de desconexión del eNB original;
correlacionar con logs del orquestador (K8s); si es lab/desarrollo, no requiere
acción correctiva; si es producción, evaluar Session Continuity/X2 handover.

**Requiere aprobación humana:** true
```

`report_key` computed for this event: `MME-20260813-013716` (component sanitized, joined with a `yyyyLLdd-HHmmss` timestamp — see the "Edit Fields" node in `workflow_export.json`).

**Independent verification, outside of n8n's own execution log:**

- ServiceNow: the incident appeared in the instance's incident list, correctly linked via `cmdb_ci` to the "Open5GS MME" Configuration Item — confirmed by opening the record directly in the ServiceNow UI, not just trusting the n8n node's "success" status.
- S3 (Floci): confirmed the object existed and was readable, from a terminal outside of n8n:
  ```bash
  AWS_ENDPOINT_URL=http://192.168.0.17:4566 AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test \
    aws s3 ls s3://open5gs-noc-reports/

  AWS_ENDPOINT_URL=http://192.168.0.17:4566 AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test \
    aws s3 cp s3://open5gs-noc-reports/MME-20260813-013716.md -
  ```
  Both the incident's evidence reference and the actual S3 key matched exactly — the point of computing `report_key` once, upstream of the branch, rather than independently on each branch.

## What's notable across both runs

In both cases, the agent distinguished the one real anomaly in the log window from the normal 3GPP procedures surrounding it (Attach, idle timers, RACH) in the same combined log block — rather than flagging every WARNING-adjacent string it saw. That's the specific failure mode a generic LLM without the `lookup_3gpp_pattern` tool and the domain-specific system prompt tends to fall into (see the earlier self-hosted-LLM experiment referenced in the README, where that confusion did happen).
