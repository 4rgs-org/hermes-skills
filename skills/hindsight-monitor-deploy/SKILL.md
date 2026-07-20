---
name: hindsight-monitor-deploy
description: Despliega el dashboard interactivo de Hindsight (grafo de entidades con layout tipo red neuronal + info-card lateral con highlight de conexiones, nodos, entidades, tags, timeseries) en un host de la LAN via LaunchAgent. Usar cuando se necesite exponer el grafo de Hindsight en `http://<host>:<port>/hindsight_dashboard.html`, o cuando el usuario pida "monitor de Hindsight", "dashboard interactivo de memory", "ver el grafo de entidades", "card al clickear un nodo", "highlight de conexiones", "layout tipo red neuronal", o refiera la URL `http://<host>:8765/hindsight_dashboard.html`.
---

# Hindsight Monitor — Deploy & Operate

Dashboard HTML self-contained servido por Python stdlib `http.server` con un endpoint `POST /api/reload` que regenera el snapshot contra la API de Hindsight. Bind a `0.0.0.0` para LAN. Persistencia via macOS LaunchAgent (`KeepAlive`).

## Arquitectura

```
[Browser LAN] --> http://<host>:8765/hindsight_dashboard.html
                        |
                        v
              [hindsight_dashboard_server.py]  (Python stdlib, ThreadingHTTPServer, CORS, /api/reload)
                        |
              POST /api/reload → subprocess.run(hindsight_dashboard.py)
                        |
                        v
              [hindsight_dashboard.py] (fetch 6 endpoints, render HTML)
                        |
                        v
              [~/.hermes/hindsight-dashboard/]
                ├── hindsight_dashboard.html  (~400KB self-contained, payload embebido)
                └── hindsight_snapshot.json   (~2MB full data)

                       Y

              [Hindsight API @ 127.0.0.1:8888]
                ├── /v1/default/banks/{bank}/stats
                ├── /entities?limit=500
                ├── /entities/graph?limit=300&min_count=2    ← vista principal interactiva
                ├── /graph?limit=1000                         ← nodos memoria
                ├── /tags?limit=100
                └── /stats/memories-timeseries?period=30d
```

## Archivos

- `~/.hermes/scripts/hindsight_dashboard.py` — generador (snapshot + HTML self-contained)
- `~/.hermes/scripts/hindsight_dashboard_server.py` — servidor HTTP con `/api/reload`
- `~/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist` — LaunchAgent KeepAlive
- `~/.hermes/hindsight-dashboard/{hindsight_dashboard.html, hindsight_snapshot.json, server.out.log, server.err.log, index.html}`

## Deploy (pasos verificados)

1. **Validar acceso al host remoto**:
   ```bash
   ssh -o StrictHostKeyChecking=no ilm@192.168.1.110 'echo OK'
   ```
   **Trap**: si el usuario SSH **no es el mismo que tu Mac local** (ej: tu Mac es `alvarogonzalez` y el host es `ilm@192.168.1.110`), `BatchMode=yes` falla con `Permission denied (publickey,password,keyboard-interactive)` aunque el ssh-agent tenga la clave cargada. La clave está en el agente pero el socket no se reenvía al subproceso. Antes de batch SSH, correr:
   ```bash
   ssh-add --apple-load-keychain    # macOS — carga claves del Keychain al agente
   ```
   (También `ssh-add -A` funciona pero está deprecado en macOS modernos.) Después de eso, SSH batch contra el host funciona normal. Si aun así falla, el host no tiene la publickey instalada — pedir al usuario o copiar con `ssh-copy-id`.

2. **Verificar Hindsight API arriba**:
   ```bash
   curl -sS http://<host>:8888/health    # → {"status":"healthy","database":"connected"}
   curl -sS http://<host>:8888/version   # → api_version: 0.8.x
   ```

3. **Copiar scripts al host**:
   ```bash
   scp hindsight_dashboard.py hindsight_dashboard_server.py <user>@<host>:~/.hermes/scripts/
   ssh <user>@<host> '~/.hindsight-venv/bin/python -m py_compile ~/.hermes/scripts/hindsight_dashboard.py ~/.hermes/scripts/hindsight_dashboard_server.py'
   ```

4. **Crear `index.html` con redirect** (el server no autoredirect a `hindsight_dashboard.html`):
   ```bash
   ssh <user>@<host> 'cat > ~/.hermes/hindsight-dashboard/index.html << "EOF"
   <!doctype html><html lang="es"><head>
   <meta charset="utf-8"><title>Hindsight Monitor</title>
   <meta http-equiv="refresh" content="0;url=hindsight_dashboard.html">
   </head><body><p>→ <a href="hindsight_dashboard.html">dashboard</a></p></body></html>
   EOF'
   ```

5. **Generar snapshot inicial** (genera HTML sin servidor):
   ```bash
   ssh <user>@<host> "~/.hindsight-venv/bin/python ~/.hermes/scripts/hindsight_dashboard.py --api-url http://127.0.0.1:8888 --bank hermes --out-dir ~/.hermes/hindsight-dashboard"
   ```

6. **Crear e instalar LaunchAgent** (`<home>/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist`):
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0"><dict>
     <key>Label</key><string>ai.ilm.hindsight-dashboard</string>
     <key>ProgramArguments</key>
     <array>
       <string><HOMEDIR>/.hindsight-venv/bin/python</string>
       <string>-u</string>
       <string><HOMEDIR>/.hermes/scripts/hindsight_dashboard_server.py</string>
       <string>--host</string><string>0.0.0.0</string>
       <string>--port</string><string>8765</string>
       <string>--out-dir</string><string><HOMEDIR>/.hermes/hindsight-dashboard</string>
       <string>--api-url</string><string>http://127.0.0.1:8888</string>
       <string>--bank</string><string>hermes</string>
     </array>
     <key>WorkingDirectory</key><string><HOMEDIR>/.hermes/hindsight-dashboard</string>
     <key>RunAtLoad</key><true/>
     <key>KeepAlive</key><true/>
     <key>StandardOutPath</key><string><HOMEDIR>/.hermes/hindsight-dashboard/server.out.log</string>
     <key>StandardErrorPath</key><string><HOMEDIR>/.hermes/hindsight-dashboard/server.err.log</string>
   </dict></plist>
   ```
   ```bash
   ssh <user>@<host> 'launchctl unload ~/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist 2>/dev/null; launchctl load ~/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist'
   ```

7. **Verificar LAN + KeepAlive**:
   ```bash
   # bind
   ssh <user>@<host> 'lsof -nP -iTCP:8765 -sTCP:LISTEN'   # → *:8765 (LISTEN)
   # curl LAN
   curl -sS -o /dev/null -w "%{http_code} bytes=%{size_download}\n" http://192.168.1.110:8765/hindsight_dashboard.html
   # resilience
   ssh <user>@<host> 'launchctl stop ai.ilm.hindsight-dashboard; sleep 2; launchctl start ai.ilm.hindsight-dashboard; sleep 2; lsof -nP -iTCP:8765 -sTCP:LISTEN'
   ```

## Interactividad del grafo

- **Click en un nodo** → card lateral derecha muestra:
  - Título + swatch de color (color por fact_type)
  - Menciones totales, ID completo, grado
  - **Tipo**: chip de color (world/experience/observation/mixto/sin datos)
  - **Breakdown**: conteos por fact_type (ej. experience 633 · world 110)
  - **Conexiones directas** (40 + "+N más") ordenadas por peso, cada una con dot del color de su tipo
  - **Examples**: hasta 5 facts de muestra con texto, fecha, tipo y tags
- **Colores por fact_type**:
  - 🟢 verde `#50fa7b` = experience (eventos)
  - 🔵 azul `#4f9cf9` = world (entidades)
  - 🟠 ámbar `#ffb86c` = observation (derivados)
  - 🟣 violeta `#bd93f9` = mixto (sin tipo dominante claro)
  - ⚪ gris `#7e8a9a` = sin datos
- **Fact_type derivado** por el generador Python: muestrea `/memories/list?limit=10000` (90% del bank), agrupa por entidad y elige el `fact_type` dominante (>60% = ese tipo, sino `mixto`). Cobertura 100% de los nodos del grafo.
- **Filtros chip**: click en cualquier chip para toggle (world/experience/observation/mixto/sin datos). Combinables. Click `aristas: 300` no es filtro, es info.
- **Highlight automático**: el nodo seleccionado queda destacado en su color (sin crecimiento de tamaño — se controla manualmente), los demás nodos se atenúan a gris con opacity 0.30, las aristas conectadas se resaltan en dorado.
- **Click en neighbor** (chip de la card) → re-selecciona ese nodo y mueve el viewport al centro con `network.moveTo({position:{x,y}})` (NUNCA `focus()`).
- **ESC** o **botón ×** o **click en canvas vacío** → limpia la selección + restaura todos los nodos a su estado base + `network.fit()` para zoom natural.
- **Filtros** (sliders min-menciones/min-co-ocurrencia + chips de tipo) → limpian la selección porque invalidan el subgrafo.
- **API expuesta** para integraciones y testing: `window.__selectNode(id)`, `window.__clearSelection()`, `window.__hindsight.{network, allNodes, allEdges, nodeById, neighborsOf, DATA}`.

## Verificación de deploys visuales

Para cualquier deploy de un dashboard interactivo, `curl 200 OK` NO es criterio de éxito. El HTML puede ser válido y contener JS roto, payload vacío, o bugs visuales que solo aparecen al renderizar. Tres capas obligatorias:

1. **curl HTTP estático**: confirma 200 + tamaño plausible (Hindsight ~400-850KB, dashboard simple <200KB, >5MB suele ser trim() olvidado).
2. **`browser_console` assertion programático**: lee estado via `window.__APP_STATE` y asserta propiedades concretas (dataset no vacío, props correctas, payload embebido completo). Requiere exponer el estado en dev (`window.__hindsight = { ... }`) — patrón legítimo estilo React DevTools.
3. **`browser_vision` assertion visual**: pregunta específica sobre composición (colores, layout, labels legibles, highlight funcionando). Captura problemas que el console check no detecta (overflow, nodos gigantes por scaling.max, label invisible).

Saltar cualquiera de las 3 capas deja bugs pasar. Detalle + flujo completo en `references/deployment-verification-pattern.md`.

## Tuning del layout tipo "red neuronal"

Para que el grafo se lea como una visualización de grafo científico (nodos separados, jerarquía clara, cúmulos + periferia), la física del barnesHut necesita ser mucho más "exagerada" que los defaults de vis-network:

```javascript
physics: {
  enabled: true,
  stabilization: { iterations: 500, fit: true, updateInterval: 25 },
  barnesHut: {
    gravitationalConstant: -14000,   // repulsión fuerte (default ~ -8000)
    centralGravity: 0.25,            // cohesión (sin esto, los nodos se van al infinito)
    springLength: 180,               // springs largos
    springConstant: 0.035,           // springs suaves → curvas orgánicas
    damping: 0.5,
    avoidOverlap: 0.8,
  },
},
nodes: {
  shape: 'dot',
  borderWidth: 1,
  size: 14,                          // radio base absoluto (vis-network interpreta como override)
  scaling: { min: 6, max: 18, label: { enabled: true, min: 11, max: 22, maxVisible: 28, drawThreshold: 6 } },
  chosen: false,                     // ← DESHABILITAR: vis-network aplica halo/boost al seleccionar
},
edges: { chosen: false },
```

Y `NODE_SIZE_BOOST = 1.6` aplicado al `value = Math.log2(mentions + 1) * 1.6` para que el rango de radios llene el viewport.

**Recenter SIN zoom al seleccionar** (verificado en vivo, ver Traps abajo): usar `network.moveTo({ position: { x, y } })` en vez de `network.focus(nodeId, { scale: ... })`. El último infla el nodo seleccionado 10x aunque le pases `scale: 1.0`.

## Traps y notas

- **`index.html` obligatorio** — el `SimpleHTTPRequestHandler` no redirige a `hindsight_dashboard.html` por sí solo. Crear manualmente o el `/` da directorio listing o 404.
- **Tamaño del HTML** — el primer build naive embebe todos los `text` completos de 1000 nodos world → ~16MB. Usar `trim_payload_for_browser()` para quedarse solo con `{id, label, date}` (resultado: ~850KB HTML, ~200KB gzip). El snapshot completo (~12MB con memories de muestra) queda en `hindsight_snapshot.json` en disco para debug.
- **`memories/list?limit=10000`** — se necesita ~10k memories para cubrir el 100% de los nodos del grafo de entidades (las 2k primeras solo cubren ~60%). Cuesta ~1.7s; aceptable.
- **vis-network `network.focus()` con cualquier `scale`** infla el nodo seleccionado ~10x — bug verificado en vivo con `scale: 0.32`, `0.85` y `1.0`. Solución: usar `network.moveTo({ position: { x, y } })` que solo translada el viewport sin zoom.
- **vis-network `chosen: true` (default)** aplica halo/border boost en hover/select → si querés control manual total del highlight, deshabilitar con `nodes.chosen: false` y `edges.chosen: false`. Sin esto, el nodo seleccionado crece visualmente al margen de tu `applyHighlight()`.
- **`scaling.max`** alto (>30) hace que nodos grandes se rendericen enormes. Cap a 18-22 para que el grafo no explote.
- **No usar `size` global en nodes a la vez que `value`** — vis-network interpreta `size` como override absoluto del `value`×`scaling`. Si querés tamaños variados, usar solo `value` (sin `size`).
- **Snap full en disco** — `hindsight_snapshot.json` (~12MB) sí conserva el `text` completo de memories para debug/recall; el HTML sólo embebe el resumen.
- **`nohup & disown` desde un script SSH es bloqueado por el security scan** — usar `terminal(background=true, notify_on_complete=true)` con la sesión SSH adentro. Para servidores de larga duración, el patrón LaunchAgent KeepAlive es mejor (sin shell-level background wrappers). Patrón:
  ```bash
  # INCORRECTO: foreground con nohup (security scan lo bloquea)
  ssh ilm@192.168.1.110 'nohup python server.py & disown'   # → denied

  # CORRECTO: background explícito de Hermes
  terminal(command="ssh ilm@192.168.1.110 'python server.py 2>&1'", background=true, notify_on_complete=true)
  # → process_id devuelto; ver logs con process(action="log", session_id=...)
  ```
- **Server arranca antes que el HTML exista** → `GET /hindsight_dashboard.html` devuelve 404 aunque el archivo esté en disco. `SimpleHTTPRequestHandler` cachea el `os.listdir()` al `bind` y no re-escanea. Solución: o arrancar el server DESPUÉS de generar el snapshot inicial (lo cual ya hace `--only-once` arriba), o `kill -9` y relanzar. **Verificar siempre** con `curl -I http://<host>:8765/hindsight_dashboard.html` después de `launchctl load`.
- **Click en `window.location.protocol === 'file:'`** — si abres el HTML desde `file://`, el `fetch('/api/reload')` resuelve a `file:///api/reload` y CORS lo bloquea. El template debe detectar el protocolo y usar `http://127.0.0.1:8765` como fallback. (Ver `references/hindsight-dashboard/local-server-with-reload.md` del umbrella skill.)
- **Highlight residue al limpiar selección (vis-network)** — si `applyHighlight(null)` retorna `{ id: n.id }` esperando "no-op", vis-network deja pegadas las propiedades (`value`, `color`, `opacity`, `font`) del último highlight. El grafo queda con nodos grises pequeños después de un clear. La rama `null` debe **restaurar explícitamente** el estado base desde los datos originales. Detalle + repro + verificación en `references/vis-network-highlight-residue-bug.md`.
- **Triple verificación en deploys visuales**: para dashboards interactivos servidos vía browser, NO basta con `curl` 200 OK. El criterio de éxito end-to-end debe incluir:
  1. **curl** confirma HTTP status + tamaño plausible (ej. ~400KB HTML).
  2. **`browser_console` assertion programática**: lee estado via `window.__hindsight` y asserta propiedades concretas (cuenta de nodos, fact_types presentes, datos del payload embebido). Esto detecta bugs que el ojo no ve (ej. payload vacío, 0 nodos, fact_type_counts incorrectos).
  3. **`browser_vision` assertion visual**: pregunta específica sobre la composición (colores correctos, layout, labels legibles, highlight funcionando). Captura problemas que el console check no detecta (overflow, nodos gigantes por scaling.max, label invisible).
  Saltar cualquiera de las 3 capas deja bugs pasar. Detalle + flujo completo en `references/deployment-verification-pattern.md`.

## Verificación rápida

```bash
# 1. API up
curl -sS http://192.168.1.110:8888/health
# 2. Dashboard 200 OK
curl -sS -o /dev/null -w "%{http_code}\n" http://192.168.1.110:8765/hindsight_dashboard.html
# 3. Reload regenera en <5s
time curl -sS -X POST http://192.168.1.110:8765/api/reload
# 4. LaunchAgent vivo
ssh ilm@192.168.1.110 'launchctl list | grep ai.ilm.hindsight'
# 5. Listener en LAN
ssh ilm@192.168.1.110 'lsof -nP -iTCP:8765 -sTCP:LISTEN'   # → *:8765
```

## Variables de entorno reconocidas

- `HINDSIGHT_API_URL` (default `http://127.0.0.1:8888`)
- `HINDSIGHT_BANK` (default `hermes`)

Ambos se pueden sobrescribir con `--api-url` / `--bank` o en el plist.

## Comandos operativos

```bash
# Ver logs en vivo
ssh ilm@192.168.1.110 'tail -f ~/.hermes/hindsight-dashboard/server.out.log'

# Reiniciar limpio
ssh ilm@192.168.1.110 'launchctl kickstart -k gui/$(id -u)/ai.ilm.hindsight-dashboard'

# Forzar reload desde shell
ssh ilm@192.168.1.110 '~/.hindsight-venv/bin/python ~/.hermes/scripts/hindsight_dashboard.py'

# Backup snapshot
ssh ilm@192.168.1.110 'cp ~/.hermes/hindsight-dashboard/hindsight_snapshot.json{,.bak.$(date +%Y%m%d_%H%M%S)}'

# Apagar el dashboard (sin desinstalar)
ssh ilm@192.168.1.110 'launchctl unload ~/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist'

# Reactivar
ssh ilm@192.168.1.110 'launchctl load ~/Library/LaunchAgents/ai.ilm.hindsight-dashboard.plist'
```