# Agentic AI Log Analyzer — Open5GS Core + srsRAN (Cloudlab-1)

Workflow de n8n con un AI Agent (Claude) que analiza automáticamente logs del Core 4G/5G (AMF, MME) y del RAN (srsenb) en tu lab, clasifica severidad, propone causa raíz y acción recomendada — con un tool de conocimiento 3GPP que el agente puede invocar antes de concluir.

## Arquitectura

```
[Cron cada 5 min]
      |
      v
[SSH: kubectl logs AMF] [SSH: kubectl logs MME] [SSH: kubectl logs RAN]
      |                        |                        |
      v                        v                        v
   [Tag AMF]                [Tag MME]                [Tag RAN]
      \_________________________|________________________/
                                 v
                         [Merge 3 fuentes]
                                 v
                    [Combinar + pre-filtro regex]
                                 v
                    ¿Hay anomalía? --No--> (fin, ahorra tokens)
                                 |
                                Sí
                                 v
                          [AI Agent: Claude]
                    (usa tool lookup_3gpp_pattern
                     + output parser estructurado)
                                 v
                    ¿Severidad > INFO? --No--> (fin)
                                 |
                                Sí
                                 v
                [SSH: append a ai_log_analysis_report.md]
```

**Por qué es "agentic" y no solo un LLM call:** el nodo AI Agent tiene una herramienta (`lookup_3gpp_pattern`) que puede invocar de forma autónoma para contrastar el hallazgo contra una base de conocimiento antes de responder, y un output parser estructurado que fuerza una decisión (severidad, causa raíz, acción) en vez de texto libre. El diseño es explícitamente **human-in-the-loop**: el agente nunca ejecuta remediación, solo la recomienda y marca `requires_human_approval` — coherente con el enfoque que ya describís en tu CV sobre IA en operaciones de red.

## Setup en tu lab (Cloudlab-1)

1. Instalá n8n self-hosted (Docker) si no lo tenés:
   ```
   docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
   ```
2. En n8n, andá a **Credentials → New**:
   - **SSH**: host `Cloudlab-1` (o su IP), usuario `jagustin`, autenticación por password o llave privada.
   - **Anthropic API**: pegá tu API key de Anthropic.
3. Importá `workflow.json` (Workflows → Import from File).
4. Abrí cada nodo SSH y asigná la credencial SSH creada. Abrí el nodo "Anthropic Chat Model" y asigná la credencial Anthropic.
5. Activá el workflow.

> Nota honesta: armé este JSON basado en el formato estándar de exportación de n8n y en los nodos reales de tu cluster (`lab-open5gs-amf`, `lab-open5gs-mme`, `lab-srs-lte-srs-lte-0`, namespace `open5gs`), pero no tengo una instancia de n8n para probarlo end-to-end desde acá. Si al importar te tira algún error de schema, pegámelo tal cual y lo corrijo en el momento.

## Prueba real con tus logs de hoy (12 ago)

Analicé a mano los logs reales que me pasaste — esto es exactamente el tipo de output que produciría el AI Agent una vez conectado:

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

Extra que noté y que el agente también debería marcar: **AMF y MongoDB tienen exactamente 15 restarts cada uno** — vale la pena correlacionar si comparten causa (reinicio de nodo, presión de memoria, etc.).

MME y RAN no mostraron anomalías reales — los "Attach request/complete", "Mobile Reachable timer expired" e "Implicit Detach" son procedimientos 3GPP normales de ciclo de vida de UE, no fallas.

## Cómo hablar de esto en el CV / entrevista

- "Diseñé e implementé un workflow de IA agéntica en n8n que monitorea logs de mi Core 4G/5G (Open5GS) y RAN (srsRAN) en tiempo real, usando un agente basado en Claude con tool-calling y salida estructurada para triage automático de anomalías."
- "El agente está diseñado human-in-the-loop por decisión propia: clasifica severidad y recomienda acción, pero nunca ejecuta cambios sin aprobación — reflejando un enfoque responsable de IA en operaciones críticas de red."
- Podés mostrar el archivo `ai_log_analysis_report.md` generado en tu lab como evidencia tangible en la entrevista.
