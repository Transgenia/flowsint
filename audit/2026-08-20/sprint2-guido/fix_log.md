# Fix Log — Sprint 2 flowsint MCP (cleanup + PRs)

- **Fecha:** 2026-08-20T15:35 CDMX (noche)
- **Agente:** Guido — Midd PyDev
- **Autoriza:** Saurat en sesión (vault renombrado a `WHOISXML_API_KEY`, fork Transgenia listo, ejecutar los 4 pendientes remanentes del cierre §11.5 del Sprint 1)
- **Alcance:** 4 ítems concretos + verificación. Nada del roadmap post-Sprint 1 (list_sketches / cache Redis / recon_from_odoo_partner / query_graph / export_graph_json) — sigue en espera hasta noviembre según brief §3.
- **Estado:** CERRADO — GREEN (4/4 con evidencia observable; 1 paper trail impracticable documentado como no aplica).

---

## 1. Ítem 1 — Fix del template `dmarc-email-security`

### Bug del Sprint 1
`networkcalc.com` devuelve **HTTP 400** cuando el subdomain `_dmarc.<X>` no existe. El engine de flowsint trata el 400 como error y no crea nodo; la ausencia de DMARC pasaba desapercibida.

### Descubrimiento colateral durante el fix
Intento 1 usé **Cloudflare DNS-over-HTTPS** (`https://cloudflare-dns.com/dns-query?name=…&type=TXT`). Falló porque el **template engine de flowsint NO maneja query strings**: la petición llegaba a `https://cloudflare-dns.com/dns-query` sin params → 400. Confirmado con el log de status:

```
HTTP error 400 for https://cloudflare-dns.com/dns-query?name=_dmarc.{{domain}}&type=TXT:
Client error '400 Bad Request' for url 'https://cloudflare-dns.com/dns-query'
```

Constrainted a APIs con **path-based DNS queries**.

### Solución adoptada — `dnsjson.com`

- Path-style: `https://dnsjson.com/_dmarc.<domain>/TXT.json`.
- **200 en ambos casos:**
  - Con DMARC: `{"results": {"records": ["v=DMARC1; …"]}}`.
  - Sin DMARC: `{"results": {"records": []}}`.
- Cuando `records: []`, el `array_path: results.records` se resuelve a 0 nodos — la **ausencia = testimonio observable** en el grafo, no error.

Verificado en vivo antes de hacer el fix:
```
dentalperfect.com.mx  → records: ["v=DMARC1; p=quarantine;..."]
hospitalpolar.com     → records: []           (dolor vendible correctamente diagnosticado)
```

Template v2.1 push al vault vía `PUT /api/enrichers/templates/ca2e0144-b3a0-41aa-a1aa-d0ffdc80c97e`. Verificación viva post-fix:

- **hospitalpolar.com** (sin DMARC): `[dmarc-email-security] started → COMPLETED` sin `HTTP error`, sin `GRAPH_APPEND` (0 nodos). Correcto.
- **dentalperfect.com.mx** (con DMARC): `[dmarc-email-security] started → GRAPH_APPEND "[DMARC-EMAIL-SECURITY] dentalperfect.com.mx -> v=DMARC1; p=quarantine; psd=n; adkim=s; aspf=s; fo=0:1;" → COMPLETED`. Regresión OK.

Archivo: `S:/Transgenia/Flowsint/templates/04-dmarc-email-security.yaml` (versión 2.1).

## 2. Ítem 2 — Sub-fix upsert real en `flowsint_load_custom_templates_from_disk`

### Bug del Sprint 1
El tool hacía `POST /api/enrichers/templates` sin verificación previa. Cuando el template ya existía el API devolvía HTTP 400 con `'Template with name X already exists'`, marcando el skip. El flag "idempotent" del docstring era incorrecto.

### Fix aplicado (commit `1d0ede5`)
Check-first-then-upsert:

1. Obtener lista viva de enrichers al inicio del tool.
2. Para cada YAML declarado en `allowlist.yaml`:
   - Si `tname in live_names`: extraer `live_entry['raw']['id']` y hacer `PUT /api/enrichers/templates/<id>` → `action: "updated"`.
   - Si no: `POST /api/enrichers/templates` → `action: "created"`.
   - Si `live_entry` existe pero sin `id`: skip con `"live enricher lacks id; cannot PUT"`.

Docstring del `note` actualizado: "True upsert: PUT when the enricher already exists, POST otherwise."

### Verificación
El proceso stdio del MCP en Quack tiene el módulo en memoria (viejo comportamiento del sprint 1) — sale HTTP 400 "already exists" para los 3 templates que ya existen y "not deployed yet" para dmarc (porque intenta POST y también choca). **La próxima invocación en una sesión Quack nueva tomará el fix del disco.**

Test end-to-end bypass HTTP (mismo Python client, con el fix) confirma el flujo: `dmarc-email-security` PUT id `ca2e0144…` → live version 2.1.

## 3. Ítem 3 — PR upstream a `reconurge/flowsint`

### Genericización

Rama nueva `upstream/generic-additions` creada desde `origin/main` (`616516b`) con **un solo commit** (`cf79bcf`) ofrecible a upstream:

- `mcp/flowsint_mcp.py` — docstring genérico (sin "transgenia-", sin S:/ paths), `TEMPLATES_DIR` default a `./templates` en vez de `S:/Transgenia/…`.
- `mcp/allowlist_loader.py` — soporta `FLOWSINT_ALLOWLIST_PATH` env var + fallback automático a `allowlist.example.yaml` cuando `allowlist.yaml` no existe (`state()['using_fallback_example']` para observabilidad).
- `mcp/allowlist.example.yaml` — 26 upstream enrichers + 23 denylist entries, todos con `authorized_by: reviewer` y `reviewed_by: reviewer` (placeholders). Zero referencias a Transgenia.
- `mcp/otel_bootstrap.py` — cabecera limpia; retirada la referencia al commit `044c27b` de Fase 0 Odoo. Auto-append de `/v1/traces` documentado como comportamiento estándar.
- `mcp/metrics.py` — cabecera limpia sin referencias a `otel-collector-tgn` / `Saurat Q5 2026-08-20`.
- `mcp/env.example` — `FLOWSINT_TEMPLATES_DIR` sin default `S:/Transgenia`.
- `mcp/README.md` — reescrito genérico con quick-start + governance model + observability.
- `mcp/.gitignore` — `allowlist.yaml` local + `.env` + `__pycache__`.
- **Excluidos**: el archivo `allowlist.yaml` con `reviewed_by: kelsey-mcmire`, `authorized_by: valentina-truesales`, template_path absolutos `S:/Transgenia/Flowsint/templates/…`. Todos los datos de gobierno específicos de Transgenia permanecen SOLO en la rama `feature/…` y en `main` del fork.

### PR creado

`gh` recuperó credenciales (Saurat rearregló el keyring después del Sprint 1). Creado en modo **draft**:

> **[Draft] Add stdio MCP wrapper with declarative YAML allowlist governance**
> https://github.com/reconurge/flowsint/pull/212

## 4. Ítem 4 — PR `feature→main` en Transgenia (paper trail)

**No aplica.** GitHub rechaza la creación de PRs con diff vacío:
```
pull request create failed: GraphQL: No commits between main and feature/flowsint-mcp-refinamiento-2026-08
```

Es comportamiento intencional de la API. El paper trail alternativo:

- **`main` del fork** en `1d0ede5` contiene todos los commits del Sprint 1 + fix payload + fix_log de cierre + fix upsert.
- **La rama `feature/flowsint-mcp-refinamiento-2026-08`** sigue existiendo (44859b9) como referencia histórica del punto exacto donde arrancó el sprint.
- **El `audit/2026-08-20/{sprint1-guido,cierre-guido,sprint2-guido}/fix_log.md`** provee el paper trail humano con la historia completa.

## 5. Sincronización `otel_bootstrap.py` — verificación

`S:\ProyectosTransgenia\mcp-fase0-worktree\odoo-rpc-mcp\otel_bootstrap.py` YA fue sincronizado en el commit `923e465` del cierre del Sprint 1 (rama `feature/mcp-fase0-guido-2026-07-08`). No requiere acción en este sprint. Ambos bootstraps son byte-idénticos en la sección del exporter.

## 6. Definition of GREEN para el Sprint 2

| # | Punto | Estado | Evidencia |
|---|---|---|---|
| 1 | Fix del template `dmarc-email-security` para 400 = "sin DMARC" | **GREEN** | Verificación viva §1 — hospitalpolar 0 nodos + sin `HTTP error`; dentalperfect nodo con record DMARC completo |
| 2 | Sub-fix upsert real en `flowsint_load_custom_templates_from_disk` | **GREEN** | Commit `1d0ede5`; verificación viva bypass HTTP mostró PUT → dmarc-email-security v2.1 |
| 3 | PR upstream `reconurge/flowsint` con parte genérica separada | **GREEN** | Rama `upstream/generic-additions` en Transgenia/flowsint (commit `cf79bcf`); draft PR #212 abierto |
| 4 | PR `feature→main` Transgenia como paper trail | **N/A** | GitHub rechaza diff vacío por diseño; paper trail alternativo en `audit/` + branches persistentes |
| 5 | Vault del flowsint renombrado a `WHOISXML_API_KEY` | **GREEN (Saurat)** | Confirmado por captura del panel `localhost:5173/dashboard/vault` — 1 key `WHOISXML_API_KEY` fechada `Aug 20 2026` |

**Resumen: 4 GREEN + 1 N/A. Sprint 2 cerrado.**

## 7. Pendientes remanentes hacia Sprint 3 (post-octubre según brief §3)

| Acción | Owner |
|---|---|
| Merge PR upstream #212 en `reconurge/flowsint` (una vez que Yvan lo revise) | Reconurge upstream + Guido |
| Roadmap features Sprint 3: `flowsint_list_sketches`, cache Redis por seed 14 d, `flowsint_recon_from_odoo_partner` (mode=quick, write_tags con VoBo string), `flowsint_query_graph`, `flowsint_export_graph_json` | Guido, post-octubre según brief §3 |
| Auditoría LFPDPPP formal del allowlist (documento `audit/policies/flowsint-lfpdppp-v1.md`) | Kelsey (gobierno) + Guido (implementación) |
| CI `pytest` para `mcp/tests/` (smoke test del stdio loop + ast.parse) | Guido, junto con Sprint 3 |

## 8. Referencias

- Fix template dmarc: `S:/Transgenia/Flowsint/templates/04-dmarc-email-security.yaml` v2.1
- Fix upsert commit main: `1d0ede5`
- Rama upstream: `upstream/generic-additions` @ `cf79bcf` en `Transgenia/flowsint`
- PR upstream draft: https://github.com/reconurge/flowsint/pull/212
- Fork main HEAD: `1d0ede5` en `Transgenia/flowsint`
- Sketches del test dmarc:
  - `ab0ae430-5497-4187-baeb-861941cb2f57` (dentalperfect, DMARC presente)
  - `725ec76e-6d53-488b-9eba-0720c9c4480f` (hospitalpolar, DMARC ausente)

---

— Guido
