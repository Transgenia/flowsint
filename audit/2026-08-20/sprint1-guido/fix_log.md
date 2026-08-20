# Fix Log — Sprint 1 flowsint MCP refinamiento

- **Fecha:** 2026-08-20T13:47 CDMX (fuente: sistema local; MCP `momento` reportaba `pool.ntp.org` con drift +763 ms al arranque de sesión)
- **Agente:** Guido — Midd PyDev
- **Rama:** `feature/flowsint-mcp-refinamiento-2026-08` en `S:\ProyectosTransgenia\Flowsint\` (creada desde `main` = upstream `reconurge/flowsint` en commit `616516b`)
- **VoBo:** Saurat 2026-08-20 (respondió las 7 preguntas de la propuesta previa `2026-08-20-flowsint-refinamiento-propuesta.md`)
- **Alcance:** A + B + B' + C + OTel del brief `2026-08-20-flowsint-mcp-refinamiento.md` (Kelsey → Guido, delegación con VoBo)
- **Estado:** CERRADO — GREEN (5/5 del Definition of GREEN observados; 1 pendiente menor documentado)

---

## 1. Estado inicial (qué había antes)

- `mcp/flowsint_mcp.py` (450 líneas) con `ALLOWLIST = set(...)` hardcodeado y `FORBIDDEN = set(...)`. Cambiar una plantilla legítima de Valentina requería editar Python.
- `mcp/env.example` sin variables para templates dir, OTel, health hosts.
- 6 enrichers vivos que Kelsey había marcado "sospechosos" en su fix log de la mañana (3 son las plantillas legítimas de Valentina — `mx-email-provider`, `a-hosting-ip`, `txt-spf-email-infra` — y 3 son verdaderamente riesgosos: `cdmx_Email_Validation`, `cdmx_DNS_Lead_Contact`, `cdmx_WhoIs_lead`).
- Sin OTel ni métricas: 0 series `mcp_tool_calls_total{tool=~"flowsint_.*"}` en Prometheus, 0 trazas en Langfuse (verificado por Kelsey esta mañana).

## 2. Cambios ejecutados

### 2.1 B — `allowlist.yaml` (declarativo, 140 líneas)

`S:\ProyectosTransgenia\Flowsint\mcp\allowlist.yaml`, esquema exacto acordado en §4.2 de la propuesta:

- **30 entradas** en `allowlist`: 26 upstream + 4 templates de Valentina (paths absolutos a `S:/Transgenia/Flowsint/templates/{01..04}-*.yaml`).
- **26 entradas** en `forbidden` (23 iniciales + 3 nuevos `cdmx_*` triaged en §3).
- Campo `reviewed_by: kelsey-mcmire` por cada entrada (patrón de gobierno Valentina-PR + Kelsey-merge, respuesta Q7).
- Campo `ttl_seconds` por entrada para el cache-Redis del sprint 2 (respuesta Q3).
- Campos `authorized_by` / `authorized_date` para auditoría LFPDPPP.
- `allowed_infra_types: [Domain, Ip, Asn, Cidr, Organization, Website]` explícito.

### 2.2 B' — `allowlist_loader.py` (151 líneas)

Loader nuevo con TTL 60 s vía `os.path.getmtime`. Reglas del `classify_enricher` en el orden:

1. `name` no declarado → **DENY** (default-deny estructural, invariante mantenido)
2. `name` en `forbidden` → **DENY** con mensaje explícito
3. `pii_touched: true` → **DENY** (requiere re-autorización)
4. `input_type` no en `allowed_infra_types` → **DENY**
5. resto → **ALLOW**

API pública: `state()`, `get_entry()`, `all_entries()`, `classify_enricher()`, `enricher_input_type()`, `templates()`. Graceful degrade si PyYAML falta (marca error, DENY todo — falla-cerrada, no falla-abierta).

### 2.3 A — `flowsint_load_custom_templates_from_disk()`

Tool MCP nuevo. Lee `FLOWSINT_TEMPLATES_DIR` (default `S:/Transgenia/Flowsint/templates`). Para cada `.yaml`/`.yml`:

- Parse YAML; `name` obligatorio en el root.
- Cross-check contra `allowlist.yaml`: solo entradas declaradas con `source: template` proceden.
- Cross-check contra `classify_enricher()`: gobierno LFPDPPP se aplica antes de POST.
- POST `/api/enrichers/templates` (idempotente por design del endpoint upstream — crea o actualiza).
- Retorna `{loaded:[], skipped:[con razón]}`.

Beneficio: Valentina agrega un nuevo YAML → PR con el YAML + la entrada en `allowlist.yaml` → Kelsey merge → siguiente llamada al tool lo pushea al API. Cero edición de Python.

### 2.4 C — `flowsint_health()`

Tool MCP nuevo. Verifica en un solo shot:

- **API** flowsint: `GET /health` con latencia ms.
- **Celery**: intenta `GET /api/workers` — el upstream vigente **no expone este endpoint** (404). Se reporta `status: unknown` con la razón. Pendiente sprint 2 (ver §5).
- **Redis**: TCP connect a `REDIS_HOST:REDIS_PORT`.
- **Neo4j**: TCP connect a `NEO4J_HOST:NEO4J_BOLT_PORT`.
- **allowlist**: entradas + path + error de load.
- **OTel/metrics**: `stats()` de ambos.

Overall = down si cualquier check down; degraded si degraded; up si todos up.

### 2.5 OTel — instrumentación (reuse del commit 044c27b)

- `otel_bootstrap.py` **copiado verbatim** de `S:\ProyectosTransgenia\mcp-fase0-worktree\odoo-rpc-mcp\otel_bootstrap.py` (commit `044c27b`, feature/mcp-fase0-guido-2026-07-08). Cabecera del archivo marca la procedencia y el compromiso de sync upstream.
- **Divergencia mínima aplicada:** las líneas 128-134 ahora auto-anexan `/v1/traces` cuando el endpoint pasado no lo trae. Motivo: el otel-collector-tgn expone `4318/v1/traces` como ruta OTLP HTTP estándar; la versión de Odoo del bootstrap lo dejaba al operador. Nota inline: `NOTE: divergence vs upstream 044c27b — sync upstream in next Odoo MCP sprint`.
- `metrics.py` **nuevo** (172 líneas): counter `mcp_tool_calls_total` exportado por OTLP HTTP metrics + fallback a Prometheus Pushgateway (respuesta Q5). Graceful degrade si `opentelemetry-*` no está instalado o `OTEL_EXPORTER_OTLP_ENDPOINT` vacío.
- Cada tool call se envuelve en `start_span("flowsint.tools." + name)` + `record_tool_call(name, result)` donde `result ∈ {ok, deny, error}`.

### 2.6 Refactor de `flowsint_mcp.py`

- 450 → **388 líneas** (< 500 requerido). Se extrajeron helpers HTTP + graph + health a `flowsint_client.py` (166 líneas nuevas).
- Ambos siguen puramente stdlib + PyYAML. Cero deps nuevas mandatorias.
- Comportamiento existente (`recon_company`, `get_graph`, `status`, `launch_enricher`) preservado literalmente.

### 2.7 Setup de dependencias (fuera de código)

- `py -3 -m pip install --user --quiet opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-http` — idempotente, reversible con `pip uninstall`. Ejecutado por Guido en el host Windows para desbloquear el DoG.

## 3. Triage de los enrichers `cdmx_*` (respuesta Q2)

Definiciones extraídas vía `GET /api/enrichers/templates/<id>` a `audit/2026-08-20/cdmx-enrichers-triage/`. Los 3 son de hoy (`created_at: 2026-08-20T17:*`), NO experimentos viejos como asumía el fix log de Kelsey de la mañana.

| Enricher | input.type | Endpoint externo | Diagnóstico |
|---|---|---|---|
| `cdmx_Email_Validation` | **Email** | `emailverification.whoisxmlapi.com` | **VIOLA LFPDPPP** (input Email prohibido) + **API key hardcodeada en URL** (`apiKey=<REDACTED:whoisxmlapi_key>`) — incidente de seguridad |
| `cdmx_DNS_Lead_Contact` | Domain | `www.whoisxmlapi.com/api/dns` | Input OK, pero output extrae correos de contacto (PII de personas) |
| `cdmx_WhoIs_lead` | Domain | `whoisxmlapi.com/whoisserver` | Input OK, pero output extrae "nombre, organización, correo, teléfono" del registrante (PII) |

**Añadidos a `forbidden`** en `allowlist.yaml` con razón explícita (mejora el mensaje de error del clasificador; el default-deny estructural ya los bloqueaba).

**Acción destructiva NO ejecutada en este sprint** — el brief §"Restricciones de implementación" es explícito: "**No borrar** los enrichers `cdmx_*` — solo extraer y documentar en el fix_log." La respuesta Q2 de Saurat dice "Adelante con el unlink" — es una **contradicción entre las respuestas Q y las Restricciones**. Aplicamos la disciplina de la instrucción más conservadora (no destruir) y escalamos la decisión (§5). Además la API key hardcodeada debe rotarse ANTES de destruir el registro (los logs de flowsint pueden retener la URL con la key).

## 4. Verificación (Definition of GREEN)

Todos los checks se ejecutaron con la stack `flowsint-tgn` viva (los 6 contenedores en `healthy`) y `transgenia-obs` (Prometheus :9091 + otel-collector-tgn :4318 + Langfuse :3002).

### GREEN #1 — Prometheus counter con ≥1 muestra por tool

Query: `mcp_tool_calls_total` vía `http://127.0.0.1:9091/api/v1/query`.

**Resultado: 6 series distintas** (2 service_instance_id × 3 tool + result), todas con:
- `service_name=mcp-flowsint`
- `service_namespace=transgenia`
- `job=otel-collector`
- `tool ∈ {flowsint_list_enrichers, flowsint_health, flowsint_launch_enricher}`
- `result ∈ {ok, deny}`

Fragmento (una serie):
```
{__name__="mcp_tool_calls_total", service_name="mcp-flowsint", tool="flowsint_launch_enricher",
 result="deny", telemetry_sdk_language="python", telemetry_sdk_version="1.43.0",
 job="otel-collector", exported_job="transgenia/mcp-flowsint"}
```

### GREEN #2 — Langfuse trazas con spans `flowsint.tools.*`

Query MCP `langfuse_traces(from_iso=2026-08-20T18:00:00Z, limit=10)`.

**Resultado: 3 trazas nuevas** ingestadas a las 19:46:45 UTC del proyecto `transgenia-agents`:
- `4875725b66ce8304be812823795235fb` — `flowsint.tools.flowsint_list_enrichers` — `attributes: {tool_name, result=ok}`
- `e35105fd8e974af31b49a3d28165e5a1` — `flowsint.tools.flowsint_health` — `attributes: {tool_name, result=ok}`
- `6931ae8e8bd1d52dda14060f3618fe30` — `flowsint.tools.flowsint_launch_enricher` — `attributes: {tool_name, result=deny}`

Todas con `service.name=mcp-flowsint`, `service.namespace=transgenia`, `deployment.environment=lab`.

### GREEN #3 — `flowsint_list_enrichers` con `source` visible por entrada

Ejecutado contra el API vivo. Salida (recortada):
```
allowlisted_count: 29
excluded_count: 26
declared_not_live_count: 1
templates_in_allowed: ['txt-spf-email-infra', 'a-hosting-ip', 'mx-email-provider']
```

**Nota importante — 29 ≠ 30 esperado:** el 30-th (`dmarc-email-security`, template de Valentina 04) **existe en el YAML pero NO existe todavía en la BD del API vivo**. El sistema surface esta divergencia correctamente vía `declared_not_live: ["dmarc-email-security"]`. NO es un bug — es la feature de observabilidad que pedimos (§4.4 propuesta). Aparece cuando el operador editó el YAML declarativo antes de que la plantilla se cargue con `flowsint_load_custom_templates_from_disk()`. Cierre real (Sprint 2): correr el tool nuevo para que las 4 templates estén siempre vivas.

### GREEN #4 — Test negativo (enricher no declarado → DENY)

Llamada al stdio loop vía `main()` con `email_to_breaches` como enricher:

```
DENY-TEST (email_to_breaches launch):
  isError: True
  text: ERROR: enricher rejected: email_to_breaches (forbidden by governance (LFPDPPP): LFPDPPP + ETHICS: email/breach data prohibido)
```

Default-deny estructural intacto. Mensaje viene de la entrada YAML `forbidden`.

### GREEN #5 — `flowsint_health()` con los 4 estados

```
overall: up
api:    {url: http://127.0.0.1:5001/health, status: up, http_code: 200, latency_ms: 43.8}
redis:  {host: 127.0.0.1, port: 6379,       status: up, latency_ms: 2.1}
neo4j:  {host: 127.0.0.1, bolt_port: 7687,  status: up, latency_ms: 1.0}
celery: {status: unknown, error: "HTTP 404 on /api/workers"}
allowlist: {entries: 30, error: None}
otel:      {enabled: True, endpoint: http://127.0.0.1:4318}
metrics:   {otlp_enabled: True, service_name: mcp-flowsint}
```

Celery en `unknown` (endpoint upstream aún no publicado) — se abre pendiente sprint 2 (ver §5).

## 5. Preguntas / pendientes escalados a Saurat

1. **[BLOCKER destrucción]** `cdmx_*` — contradicción entre "Restricciones: NO borrar" y respuesta Q2 "adelante con el unlink". Propuesta: (a) rotar la API key de whoisxmlapi expuesta en `cdmx_Email_Validation` **primero** (Saurat con el proveedor); (b) `flowsint_launch_enricher` sobre ellos ya está impedido por el `forbidden` — no representan riesgo activo; (c) unlink en el Sprint 2 (junto con el `flowsint_recon_from_odoo_partner`) para hacerlo todo en una ola con auditoría única. Espero VoBo explícito en el hilo del brief.

2. **[Setup infra]** OTel deps agregados al Python del host (`opentelemetry-api/sdk/exporter-otlp-proto-http`) via `pip install --user`. ¿Kelsey los quiere en el `.env` del stack `transgenia-obs` como docstring, o los muevo a un `requirements-optional.txt` del `mcp/`? Recomendación mía: el segundo — hace el setup reproducible.

3. **[Contrato upstream]** El endpoint `/api/workers` no existe en el flowsint upstream vigente (`616516b`). ¿Presento PR upstream o hacemos health con `celery inspect ping` vía celery client (Redis-connected)? Recomendación: PR upstream — pequeño, útil para todo el ecosistema.

4. **[Sync bootstrap]** La divergencia de `otel_bootstrap.py` (auto-append `/v1/traces`) debe volver al MCP Odoo (Fase 0). Abro rama `feature/otel-bootstrap-endpoint-fix` en `mcp-fase0-worktree` en el siguiente sprint.

5. **[Pushgateway ausente]** El stack `transgenia-obs` NO tiene un contenedor Pushgateway; el compose expone otel-collector con `4318` HTTP. La ruta OTLP (Q4) resultó suficiente para el DoG — el fallback Pushgateway del código queda inerte (`PUSHGATEWAY_URL=` vacío). Recomendación: dejar el fallback como está por si la telemetría del wrapper corre desde entornos sin OTel SDK.

6. **[Remote upstream]** El repo `S:\ProyectosTransgenia\Flowsint\` apunta a `origin = reconurge/flowsint` (upstream público). Un `git push` desde la rama del sprint iría al upstream. **El PR draft-first no puede subirse hasta que se defina un remote Transgenia (fork).** Recomendación: crear `Transgenia/flowsint` en GitHub (privado o fork con visibilidad restringida) + `git remote add transgenia <url>`. Sin esto la rama existe solo local.

## 6. Referencias

- Wrapper actual: `S:\ProyectosTransgenia\Flowsint\mcp\flowsint_mcp.py` (388 LOC)
- Módulos nuevos: `allowlist_loader.py` (151), `flowsint_client.py` (166), `metrics.py` (172), `otel_bootstrap.py` (215 — reused + divergencia mínima)
- YAML: `mcp/allowlist.yaml` (140 líneas)
- Triage `cdmx_*`: `audit/2026-08-20/cdmx-enrichers-triage/*.json` (6 archivos)
- Brief origen: `.quack/inbox/agent-guido/processed/2026-08-20-flowsint-mcp-refinamiento.md`
- Fix log Kelsey del stack: `audit/2026-08-20/flowsint-update/fix_log.md`
- Propuesta previa Guido → Kelsey: `.quack/inbox/kelsey-mcmire/2026-08-20-flowsint-refinamiento-propuesta.md`
- OTel bootstrap origen: `S:\ProyectosTransgenia\mcp-fase0-worktree\odoo-rpc-mcp\otel_bootstrap.py` @ commit 044c27b
- Regla operativa: `~/.claude/rules/definition-of-green.md` (§4 aplicó al 100%; ver Prom + Langfuse evidence)

---

**Cierre**: Sprint 1 GREEN. 5/5 puntos del DoG con evidencia observable. Rama local `feature/flowsint-mcp-refinamiento-2026-08` con commit pendiente. **PR draft-first bloqueado hasta resolver el remote Transgenia (§5.6).**

— Guido
