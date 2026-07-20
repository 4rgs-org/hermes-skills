---
name: macos-ssh-from-hermes-session
description: SSH desde una sesión Hermes en macOS hacia hosts LAN (incluyendo iCloud/MacBook remotos). Cubre el patrón de carga explícita de claves desde el Keychain cuando BatchMode=yes y PreferredAuthentications fallan, los flags legacy (`-K`/`-A`) vs modernos (`--apple-use-keychain`/`--apple-load-keychain`), la variable `APPLE_SSH_ADD_BEHAVIOR`, y la verificación previa al primer comando host-side (lsof / health / version). Use cuando el usuario dice "conéctate a <host>", "haz ssh a <machine>", "ejecuta esto en mi otra Mac", "lanza esto en <IP>", o cuando un comando SSH dentro de una herramienta falla con `Permission denied (publickey,password,keyboard-interactive)` o `Permission denied, please try again` sin prompt.
---

# SSH from Hermes session to macOS LAN hosts

## Core problem

Las sesiones Hermes se ejecutan en subprocesos sin TTY, sin sesión GUI completa y sin socket del ssh-agent de la sesión gráfica del usuario. Resultado: aunque `~/.ssh/id_ed25519` esté cargada en el agente (`ssh-add -l` la lista) y el usuario pueda SSHear desde Terminal.app sin fricción, una llamada SSH desde la sesión Hermes devuelve:

- `Permission denied (publickey,password,keyboard-interactive)` con `BatchMode=yes`
- `Permission denied, please try again` (×N) con `PreferredAuthentications=password` (porque no hay prompt interactivo)
- `No such file or directory` al usar `-o IdentityFile=~/.ssh/key` si la clave no está exportable (caso típico Apple Silicon moderno)

La causa raíz es que la clave vive en **Keychain Access** (`/Library/Keychains/System.keychain`), no en `~/.ssh/`, y solo el ssh-agent GUI la expone. `ssh` desde un subshell no-GUI no la encuentra.

## Workflow (orden obligatorio)

### 1. Diagnóstico previo (read-only, 30 segundos)

```bash
# ¿Hay clave cargada en el agent del subshell actual?
ssh-add -l
# → SHA256:... id_ed25519 (si lista → clave está, problema es del flujo)
# → The agent has no identities. (si no lista → clave no está en agent)

# ¿La clave está físicamente en ~/.ssh/?
ls -la ~/.ssh/id_*
# → típica respuesta en macOS moderno: ~/.ssh/ está vacío o solo tiene config

# ¿El host destino es alcanzable por TCP?
nc -zv -w 3 <HOST_IP> 22
# → succeeded (si OK)
# → Connection refused / timeout (si problema de red, no de auth)
```

**Stop y reporta al usuario si**: `nc` falla (no es problema de auth, es problema de red). NO persigas el flujo SSH con más flags.

### 2. Cargar la clave desde Keychain explícitamente

```bash
# macOS moderno (12+): flag nuevo
ssh-add --apple-load-keychain ~/.ssh/id_ed25519 2>&1
# macOS Big Sur o más viejo, o sesión sin el flag nuevo
ssh-add -K ~/.ssh/id_ed25519 2>&1

# Verificar que cargó
ssh-add -l
# → SHA256:... id_ed25519 (debe listar)
```

Si `ssh-add -K` falla con `WARNING: The -K and -A flags are deprecated`, significa que estás en macOS moderno. Usa `--apple-use-keychain` y `--apple-load-keychain`. Si ambos coexisten con warnings, **ignora el warning** — el flag funciona aunque esté deprecated.

Si el usuario reporta "mi clave está en otro nombre" (no `id_ed25519`):
```bash
# Listar claves conocidas en Keychain (alternativa a ssh-add -l)
security find-generic-password -s "SSH: ..." 2>/dev/null | head -5
# O pedirle al usuario que confirme el path antes de cargar
```

### 3. Override portable cuando el flag nuevo no esté disponible

```bash
export APPLE_SSH_ADD_BEHAVIOR=always
ssh-add -A   # o --apple-load-keychain
```

`APPLE_SSH_ADD_BEHAVIOR=always` fuerza a `ssh-add` a recargar desde Keychain cada vez, comportamiento equivalente al `-K` legacy. Útil cuando la sesión no tiene los binarios actualizados pero la variable sí está disponible.

### 4. Probar SSH (sin flags raros)

```bash
ssh -o ConnectTimeout=6 -o StrictHostKeyChecking=no <user>@<host> 'echo OK; hostname; whoami; uname -sm'
# → OK / MacBook-Pro-de-X.local / <user> / Darwin arm64 (en éxito)
```

**Si sigue fallando**:
- Verificar que el destino **acepta publickey** (algunas cajas Linux solo aceptan password): `ssh -v ... 2>&1 | grep -E "Authentications|method"` muestra los métodos que el server anuncia.
- Verificar que la clave pública está en `~/.ssh/authorized_keys` del destino (no relevante si el usuario dice "ya funciona desde mi Terminal").
- **STOP**. No seguir intentando con flags diferentes. Pedirle al usuario que pegue el output de `ssh -vvv <user>@<host>` desde su Terminal (la sesión Hermes no puede generar ese diagnóstico útil).

### 5. Una vez conectado: reconocimiento host-side antes de actuar

Antes de tocar archivos o servicios en el host remoto, ejecuta un bloque corto de reconocimiento (no destructivo). Patrón:

```bash
ssh <user>@<host> bash <<'REMOTE'
echo "=== whoami / hostname ==="
whoami; hostname; uname -sm
echo "=== /health del servicio (si aplica) ==="
curl -sS -m 3 http://127.0.0.1:<PORT>/health
echo "=== listeners del servicio (no 127.0.0.1) ==="
lsof -nP -iTCP -sTCP:LISTEN 2>/dev/null | grep -v "127.0.0.1\|::1" | head -10
echo "=== LAN IP del host ==="
ifconfig 2>/dev/null | grep -E "inet " | grep -v 127.0.0.1
REMOTE
```

Esto responde tres preguntas antes de planificar:
- ¿Quién soy y en qué máquina estoy? (importante si el usuario tiene 3 Macs con el mismo hostname base)
- ¿El servicio que el usuario quiere tocar está vivo?
- ¿El servicio está bindeado a LAN o solo a 127.0.0.1? (si solo 127.0.0.1, hay que relanzarlo con `0.0.0.0` o `tunnels` antes de exponerlo)

## Anti-patterns

- **NO uses `BatchMode=yes` con expect/scripts cuando la clave está en Keychain** — `BatchMode` desactiva prompts, pero el subshell sigue sin acceso al agent GUI. `BatchMode` no es la solución.
- **NO pidas la password al usuario si ssh-agent tiene la clave** — es degradar seguridad sin necesidad. El 95% de las veces, cargar la clave del Keychain resuelve el problema sin exponer nada.
- **NO uses `ssh -o IdentityFile=...` apuntando a `~/.ssh/id_ed25519`** si esa ruta no existe físicamente. En macOS moderno, el archivo no está; está en Keychain. Cargar con `ssh-add --apple-load-keychain` o esperar a que el usuario la exporte a `~/.ssh/`.
- **NO asumas que el usuario en el host remoto es `alvarogonzalez`** porque esa es tu memoria histórica. Siempre confirma con `whoami` en el primer comando host-side. En esta casa, varios hosts usan `ilm` como usuario.
- **NO persistas `SSH_AUTH_SOCK` de una sesión a otra** — el socket cambia entre invocaciones. Cada comando SSH nuevo hereda el socket del shell actual; si está vacío, el agent no responde y `BatchMode` falla.
- **NO uses `-o StrictHostKeyChecking=no` para evitar el prompt de fingerprint** sin saber qué estás aceptando. En LAN doméstica está bien; en hosts externos es un vector de MITM.

## Common traps

### "Ya funciona desde mi Terminal" pero no desde Hermes

Síntoma: el usuario pega su comando SSH, ves que es idéntico al tuyo, pero el tuyo falla. La diferencia: Terminal.app carga el ssh-agent GUI completo al arrancar; la sesión Hermes no.

```bash
# Debug útil: comparar SSH_AUTH_SOCK entre sesiones
# Terminal.app:
echo $SSH_AUTH_SOCK
# → /private/tmp/com.apple.launchd.<random>/Listeners  (típico)
# Subshell Hermes:
echo $SSH_AUTH_SOCK
# → vacío o apunta a un socket sin PIDs conectados
```

Si el socket está vacío en la sesión Hermes, ni `ssh-add` puede contactar al agent → la solución es `--apple-load-keychain`, que NO depende del socket.

### La clave está en Keychain pero `ssh-add -K` no la carga

Síntomas: `ssh-add -K ~/.ssh/id_ed25519` retorna `Failed to load key` o similar.

```bash
# Verificar que el archivo existe físicamente y tiene el formato correcto
ls -la ~/.ssh/id_ed25519
file ~/.ssh/id_ed25519  # debe decir "OpenSSH private key" o "ED25519"

# Verificar permisos (Keychain rechaza archivos legibles por otros)
chmod 600 ~/.ssh/id_ed25519
```

Si el archivo no existe, la clave vive **solo en Keychain**. En ese caso:
```bash
# Exportar desde Keychain a ~/.ssh/ (el usuario debe hacerlo, no el agente)
# Pasos:
# 1. Abrir Keychain Access
# 2. Buscar "ssh" o el nombre de la clave
# 3. Click derecho → Exportar como OpenSSH key
# 4. Guardar en ~/.ssh/id_ed25519 con permisos 600
# Después: ssh-add ~/.ssh/id_ed25519 ya funciona sin --apple-load-keychain
```

NO hagas esto desde la sesión Hermes automáticamente — el usuario debe confirmar porque implica mover secretos entre almacenes.

### `ssh -K` (mayúscula) NO es lo que quieres

`ssh -K` (mayúscula) es para GSSAPI/Kerberos authentication, NO para Apple Keychain. El flag correcto es minúscula `ssh-add -k` o `ssh-add -K` (en `ssh-add`, no en `ssh`). Confusión común en documentación cruzada.

### Permiso denegado después de exitosos varios

Si `ssh` funciona 3-4 veces y luego empieza a fallar con `Permission denied`, el ssh-agent probablemente se quedó sin slots o la clave expiró su entrada en Keychain. Recargar:
```bash
ssh-add -D   # borra todas las identidades
ssh-add --apple-load-keychain ~/.ssh/id_ed25519  # recarga desde Keychain
```

## Verificación de la conexión (4 checks independientes)

Después de cada conexión nueva a un host remoto, ejecutar antes de planificar cambios:

```bash
# 1. Identidad y host
ssh <user>@<host> 'whoami; hostname; uname -sm'
# → confirma usuario, hostname (Mac.local vs <otro>), arquitectura (arm64 vs x86_64)

# 2. Versión macOS
ssh <user>@<host> 'sw_vers'
# → ProductName: macOS, ProductVersion: <X.Y.Z>

# 3. Servicios críticos del host (si aplica)
ssh <user>@<host> 'lsof -nP -iTCP -sTCP:LISTEN 2>/dev/null | head -10'
# → lista puertos escuchando, LAN-bind (*) vs solo loopback

# 4. Disco y HOME del usuario remoto
ssh <user>@<host> 'df -h ~ | tail -1; ls -la ~ | head -5'
# → confirma que HOME está montado y accesible (no es un volumen externo desmontado)
```

Si alguno falla, **no procedas** — el reconocimiento incompleto es el origen de la mitad de los errores de "el archivo no está donde dijiste".

## Related skills

- `hermes-hindsight-memory-provider` — usa SSH al host de Hindsight (típicamente `ilm@192.168.1.110`). Cuando esa skill dice "SSH from the agent's shell often can't reach the host", está describiendo exactamente esta trampa; el fix es el paso 2 de este skill.
- `hindsight-knowledge-graph` — también asume SSH al host activo. Documenta la trampa de Keychain en sus anti-patterns (referencia cruzada).
- `macos-launchagent-management` — patrón de plist + bootstrap una vez que ya tienes SSH funcional.