# lyr-auth

Hub de identidad centralizado multi-tenant. Reemplaza N módulos de seguridad con un único Identity Provider (Authentik) al que se conectan todos los admins vía OIDC.

## Estado actual

**Fase 1 — Deploy de Authentik en Dokploy.**

Plan completo: [PLAN.md](./PLAN.md).

## Estructura

```
lyr-auth/
├── docker/                    # Authentik en Docker Compose
├── packages/
│   ├── lyr-auth-sdk/          # SDK Node/TS (Fase 3)
│   └── lyr-auth-cli/          # CLI de onboarding (Fase 5)
├── examples/
│   └── admin-reference/       # Admin Next.js de ejemplo (Fase 4)
└── docs/                      # Arquitectura, onboarding, runbook (Fase 6)
```

## Quickstart

### Prerrequisitos

- Servidor con Dokploy
- Dominio apuntando a tu servidor (ej: `auth.tudominio.com`)
- TLS configurado (Let's Encrypt vía Dokploy)
- PostgreSQL accesible (reutilizamos el existente)
- DNS wildcard o subdominios por admin configurados (ej: `admin-a.tudominio.com`, `admin-b.tudominio.com`)

### Deploy

1. Copiar `docker/.env.example` a `docker/.env` y completar valores
2. En Dokploy, crear nuevo proyecto desde `docker/docker-compose.yml`
3. Configurar dominio `auth.tudominio.com` apuntando al servicio `authentik-server`
4. Levantar y esperar a que el primer migrate termine
5. Entrar a `https://auth.tudominio.com/if/flow/initial-setup/` y crear el primer usuario admin

Detalles completos en [docker/README.md](./docker/README.md).

## Stack

- **IdP**: [Authentik](https://goauthentik.io/) (self-hosted, OIDC, multi-tenant)
- **DB**: PostgreSQL
- **SDK cliente**: `openid-client` + `jose` + `iron-session`
- **Tests**: Vitest
- **Monorepo**: pnpm workspaces
- **Deploy**: Docker Compose en Dokploy

## Documentación

- [PLAN.md](./PLAN.md) — Plan completo con fases, decisiones y riesgos
- [docker/README.md](./docker/README.md) — Deploy de Authentik paso a paso
- [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) — Arquitectura detallada (Fase 6)
- [docs/ONBOARDING.md](./docs/ONBOARDING.md) — Cómo agregar una empresa nueva (Fase 6)
- [docs/RUNBOOK.md](./docs/RUNBOOK.md) — Operación y recovery (Fase 6)
