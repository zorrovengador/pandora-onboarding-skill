---
name: pandora-onboarding
description: "Onboard a Pandora client=user: Hermes + Composio + Telegram."
version: 1.0.0
author: Bizbrain
license: MIT
metadata:
  hermes:
    tags: [pandora, white-label, onboarding, composio, hermes, telegram, multi-tenant, client, user]
---

# Pandora White-Label — Onboarding de cliente=usuario

Procedimiento para crear una **Pandora independiente por usuario** (Modelo B: cliente = usuario). Cada usuario obtiene su propio Hermes profile, su propio proyecto Composio (con Project API Key exclusiva), su canal Telegram y sus Connect Links OAuth autoservicio. Cada Pandora toca SOLO el Gmail/Calendar/Drive de su dueño.

## Invariantes de seguridad

- **Nunca** imprimir API keys, tokens, OAuth codes ni headers MCP.
- Guardar secretos en `/opt/data/profiles/<usuario>/.env` con `chmod 600`.
- La **Organization API Key** de Composio solo vive en `/opt/data/.env` (600), nunca en un perfil ni en logs.
- Cada proyecto Composio = 1 usuario = 1 key = 1 user_id. No reusar keys entre usuarios.
- OAuth lo completa el usuario en Google; el agente jamás escribe credenciales ni códigos.
- Revocación: borrar proyecto Composio del usuario + eliminar perfil Hermes para revocar todo.

## Prerrequisitos

1. `COMPOSIO_ORG_API_KEY` presente en `/opt/data/.env` (solo el proveedor).
2. SDK Composio instalado: `/opt/data/.venv-composio/bin/python` (repo `hermes-composio-self-provisioning`).
3. Binario Hermes: `/opt/hermes/bin/hermes` (export PATH).
4. Token de bot de Telegram por usuario (crear con @BotFather) — o un solo bot para piloto.
5. chat_id del usuario (en Telegram, el bot responde en DM; para piloto, allowlist = chat_id).

## Flujo de onboarding (por usuario)

### 1. Crear proyecto Composio del usuario
```bash
COMPOSIO_ORG_API_KEY=$(grep '^COMPOSIO_ORG_API_KEY=' /opt/data/.env | cut -d= -f2-) \
  python scripts/composio_bootstrap.py create-project --name 'bizbrain-<slug>' \
  --project-key-file /opt/data/profiles/<usuario>/composio-project-key
```
(No imprime la key; la guarda con 600.)

### 2. Crear perfil Hermes aislado
```bash
/opt/hermes/bin/hermes profile create <usuario> --no-skills
```
Escribir en `/opt/data/profiles/<usuario>/.env` (600):
```
COMPOSIO_API_KEY=<key del proyecto del usuario>
COMPOSIO_USER_ID=bizbrain-<slug>
TELEGRAM_BOT_TOKEN=<token del bot del usuario>
```

### 3. Configurar gateway Telegram del perfil
```bash
/opt/hermes/bin/hermes -p <usuario> gateway setup   # interactivo: Telegram, token, allowlist
# o editar config.yaml del perfil con platform.telegram + allowlist
```
Para piloto con un solo bot: allowlist = chat_id del usuario; DM pairing alternativa.

### 4. Generar Connect Links OAuth con la key del usuario
```bash
COMPOSIO_API_KEY=<key del usuario> /opt/data/.venv-composio/bin/python - <<'EOF'
from composio import Composio
import os, json
c = Composio(api_key=os.environ["COMPOSIO_API_KEY"])
s = c.create(user_id=os.environ["COMPOSIO_USER_ID"], mcp=True, manage_connections=False)
for tk in ["gmail", "googlecalendar", "googledrive", "googledocs", "googlesheets", "googleslides"]:
    req = s.authorize(tk)
    print(tk, req.redirect_url)
EOF
```
Salida: lista de links `connect.composio.dev/link/lk_...` (públicos, para el usuario).

### 5. Enviar instrucciones al usuario (email/WhatsApp/Telegram)
Plantilla: autorizar con la cuenta corporativa, en orden, los N links. Checklist de 6 toolkits.

### 6. Verificar ACTIVE
```bash
COMPOSIO_API_KEY=<key> /opt/data/.venv-composio/bin/python - <<'EOF'
from composio import Composio
c = Composio(api_key=os.environ["COMPOSIO_API_KEY"])
resp = c.connected_accounts.list(user_ids=[os.environ["COMPOSIO_USER_ID"]])
for item in resp.items:
    print(item.id, item.status, item.user_id, item.word_id)
EOF
```
Todos los toolkits deben estar `ACTIVE`. Si falta uno, reenviar solo ese link.

### 7. Arrancar la Pandora del usuario
```bash
/opt/hermes/bin/hermes -p <usuario> gateway start
```
El usuario chatea con SU Pandora en Telegram. Verificar con un mensaje de prueba (p.ej. "resume mi correo" con max 2 mensajes, sin escribir).

### 8. Registro del usuario
Guardar en `/opt/data/profiles/<usuario>/usuario.json` (600): nombre, slug, project_id, user_id, chat_id, toolkits activos, fecha.

## Verificación post-onboarding

- `hermes -p <usuario> mcp list` y `mcp test composio` (si se registra MCP).
- `connected_accounts.list` → ACTIVE en los 6 toolkits.
- Prueba de lectura acotada (GMAIL_FETCH_EMAILS max_results=2, include_payload=False).
- Estado del gateway: `hermes -p <usuario> gateway status`.

## Fase 2 (producto)

- **Frontend web tipo ChatGPT:** usar API server OpenAI-compatible de Hermes o TUI gateway JSON-RPC para UI propia en `pandora.mx`/subdominio por cliente.
- **Hermes Desktop:** cada usuario instala app nativa y se conecta a su perfil remoto (multi-connection soportado).
- **Multi-bot Telegram:** un bot por usuario (token por perfil).
- **Facturación:** per-seat con margen (cada usuario consume su propio LLM).
- **Memoria de empresa compartida** (para el caso A): perfil "empresa" + skills compartidos + identidades por usuario.

## Pitfalls conocidos (verificados)

- Los slugs de toolkits son SIN guion bajo: `googlecalendar`, `googledrive`, `googledocs`, `googlesheets`, `googleslides`, `youtube`. `google_calendar` → 404.
- `session.id` es None en SDK 0.21 → usar `session.session_id`.
- `hermes mcp add` es interactivo → correr con PTY y responder Y.
- El MCP de sesión caduca; recrear con `composio_bootstrap.py session` si las tools desaparecen.
- No confiar en el write de Drive para verificar; listar el folder destino.
- Las tools del MCP solo aparecen en sesiones NUEVAS del agente.
- `COMPOSIO_GET_TOOL_SCHEMAS` espera `tool_slugs` (no tool_names).

## Fuentes

- Composio: https://backend.composio.dev/api/v3.1 · docs.composio.dev
- Hermes: https://hermes-agent.nousresearch.com/docs/ (messaging, profiles, multi-connection desktop)
- Repo de provisioning: `/opt/data/hermes-composio-self-provisioning`
