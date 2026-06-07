# Plan: lyr-auth — IdP centralizado multi-tenant

## Objetivo

Centralizar el módulo de seguridad (login, usuarios, MFA, recovery, audit) de todos los admins en un único Identity Provider. Cada admin (app propia) se integra vía OIDC contra este IdP. Cada empresa cliente es un tenant dentro del IdP.

## Decisiones arquitectónicas

| Aspecto | Decisión | Razón |
|---|---|---|
| IdP | **Authentik** self-hosted | Liviano, UI moderna, OIDC estándar, multi-tenant nativo, encaja con Dokploy |
| Protocolo | **OIDC Authorization Code + PKCE** | Estándar, seguro, no requiere secretos en el cliente (recomendado para apps web) |
| DB | **Postgres existente** | Reutilizamos infra, schema dedicado para Authentik |
| Hosting | **Dokploy + Docker Compose** | Stack actual, no agrega complejidad |
| SDK | **`openid-client` v6+** + **`jose`** + **`iron-session`** | Estándar, mantenido, tipado, edge-friendly |
| Monorepo | **pnpm workspaces** | Liviano, suficiente para 3 packages |
| MFA | **No en MVP** | Configurable en Authentik para activar después |

## Componentes del sistema

```
lyr-auth/
├── docker/                          # Authentik en Dokploy
├── packages/
│   ├── lyr-auth-sdk/                # SDK Node/TS
│   └── lyr-auth-cli/                # CLI de onboarding
├── examples/
│   └── admin-reference/             # Admin Next.js de ejemplo
└── docs/
    ├── ARCHITECTURE.md
    ├── ONBOARDING.md
    └── RUNBOOK.md
```

## Mapa tenant → application → admin

```
Authentik
├── Tenant: empresa-a
│   ├── Users: [user1@empresa-a.com, user2@empresa-a.com, ...]
│   ├── Groups: [admins, editors, viewers]
│   └── Applications:
│       └── admin-a (OIDC client)
│           └── redirect_uri: https://admin-a.tudominio.com/api/auth/callback
├── Tenant: empresa-b
│   ├── Users: [...]
│   └── Applications:
│       └── admin-b (OIDC client)
└── ...
```

## Fases

### Fase 1 — Deploy de Authentik (actual)
- Monorepo inicializado
- Docker Compose con Authentik server + worker
- Blueprints de flows default
- Tema de login base
- Documentación de deploy

### Fase 2 — Tenant piloto
- Tenant `empresa-piloto` creado
- Primera Application OIDC configurada
- Login verificado con curl + navegador

### Fase 3 — `lyr-auth-sdk`
- Wrapper sobre `openid-client`
- Middlewares Express y Next.js
- Validación de JWT con JWKS
- Manejo de sesión con cookies firmadas
- Tests con Vitest + Authentik en docker

### Fase 4 — Admin de referencia
- Next.js 15 App Router
- Login, callback, dashboard protegido, logout
- Server actions con verificación de sesión
- Ejemplo de extracción de claims

### Fase 5 — `lyr-auth-cli`
- Comandos: `tenant create`, `app register`, `user invite`
- Usa service account token de Authentik
- Idempotencia y output claro

### Fase 6 — Documentación y operación
- `docs/ARCHITECTURE.md`
- `docs/ONBOARDING.md` (cómo agregar empresa nueva)
- `docs/RUNBOOK.md` (operación, recovery, backups)

## Conceptos clave

- **Tenant en Authentik** = empresa cliente (usuarios, grupos, apps, branding aislados)
- **Application en Authentik** = tu admin que consume el IdP (cliente OIDC)
- **Authorization Code + PKCE** = flow seguro para apps web
- **JWKS endpoint** = claves públicas de Authentik para verificar JWTs sin llamada de red
- **Refresh token rotation** = Authentik rota refresh tokens; persistir cifrados

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Punto único de falla (si cae Authentik, todos los admins sin login) | Backups automáticos del Postgres, healthchecks en Dokploy, runbook de restore |
| Vendor coupling con Authentik | OIDC estándar: si mudamos a Keycloak/Auth0, los clientes no se enteran |
| Secretes expuestos | `.env` en Dokploy secrets, nunca en repo, `AUTHENTIK_SECRET_KEY` rotable |
| Migración futura de usuarios | Authentik tiene API REST para bulk import; passwords: forzar reset en primera migración (no se pueden migrar hashes) |

## Out of scope (MVP)

- MFA / passkeys (configurable en Authentik, activar después)
- Identity brokering (Google/Microsoft login)
- SAML
- HA / replicación de Postgres
- Webhooks de eventos
- Branding custom por tenant (tema default al inicio)
