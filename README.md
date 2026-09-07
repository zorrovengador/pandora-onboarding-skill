# Pandora Onboarding Skill

Skill para el onboarding de **Pandora White-Label** (Modelo B: cliente = usuario).

Cada usuario obtiene:
- Un **proyecto Composio** propio (Project API Key exclusiva, user_id único)
- Un **perfil Hermes** aislado (memoria, historial, `.env` propio)
- Un **canal Telegram** propio
- **Connect Links OAuth** autoservicio (Gmail, Calendar, Drive, Docs, Sheets, Slides)

El flujo completo, invariantes de seguridad, verificación y pitfalls están en [`SKILL.md`](SKILL.md).

## Uso

1. Crear proyecto Composio (requiere `COMPOSIO_ORG_API_KEY`).
2. Crear perfil Hermes: `hermes profile create <usuario> --no-skills`.
3. Configurar gateway Telegram del perfil.
4. Generar los Connect Links con la key del usuario.
5. Enviar al usuario para que autorice OAuth.
6. Verificar `ACTIVE` y arrancar el gateway.

Ver [`SKILL.md`](SKILL.md) para el detalle completo.
