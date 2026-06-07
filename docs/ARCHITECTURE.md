# Arquitectura de lyr-auth

## Visión general

`lyr-auth` es un Identity Provider (IdP) centralizado que implementa OIDC para que múltiples aplicaciones administrativas (los "admins" de cada empresa cliente) deleguen la autenticación, gestión de usuarios, recuperación de credenciales y auditoría.

```
┌────────────────────────────────────────────────────────────────────┐
│                       DOKPLOY (servidor)                            │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  admin-empresa-a │  │  admin-empresa-b │  │  admin-empresa-c │   │
│  │  :3001           │  │  :3002           │  │  :3003           │   │
│  │  (Node/Next.js)  │  │  (Node/Express)  │  │  (Node/Next.js)  │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                     │             │
│           │ OIDC (Authorization Code + PKCE)          │             │
│           ▼                     ▼                     ▼             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │            AUTHENTIK (IdP) - auth.tudominio.com              │  │
│  │                                                                │  │
│  │  Tenant: empresa-a                                            │  │
│  │  ├─ Users: [user1@a.com, user2@a.com]                         │  │
│  │  ├─ Groups: [admins, editors]                                 │  │
│  │  └─ Applications:                                             │  │
│  │     └─ admin-a (OIDC client, client_id, secret)              │  │
│  │                                                                │  │
│  │  Tenant: empresa-b                                            │  │
│  │  └─ Applications: admin-b                                     │  │
│  │                                                                │  │
│  │  Tenant: empresa-c                                            │  │
│  │  └─ Applications: admin-c                                     │  │
│  │                                                                │  │
│  │  Services globales: flows, MFA, audit log, recovery           │  │
│  └──────────────────────────┬─────────────────────────────────────┘  │
│                             │                                       │
│  ┌──────────────────────────▼─────────────────────────────────────┐ │
│  │  PostgreSQL (existente) - DB: authentik                       │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Redis (cache, rate limiting, sessions) - interno al stack     │ │
│  └────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

## Conceptos fundamentales

### Tenant

En Authentik, un **tenant** agrupa identidad + configuración para una organización aislada. En nuestro modelo, **un tenant = una empresa cliente**.

Cada tenant tiene:
- Sus propios usuarios
- Sus propios grupos
- Sus propias applications (clientes OIDC)
- Su propio branding (logo, color, título)
- Su propio subdominio opcional (ej: `auth.empresa-a.tudominio.com`)

### Application (cliente OIDC)

Es el **cliente OIDC** que tu admin (la app) usa para pedir autenticación. Cada admin es una Application distinta en Authentik.

Configuración de una Application:
- `name`: nombre legible (ej: "Admin Empresa A")
- `slug`: identificador URL-safe
- `client_id` y `client_secret`: credenciales
- `redirect_uris`: URLs válidas para recibir el callback de Authentik
- `client_type`: `confidential` (con secret, para backend) o `public` (sin secret, para SPA)
- `scopes`: qué información pedir (`openid`, `profile`, `email`, `lyr_tenant`)

### Provider (protocolo)

Es la **implementación del protocolo** que usa la Application. Para nuestro caso: **OAuth2/OpenID Provider**.

El Provider está asociado a una Application y define:
- El flow: `implicit`, `authorization_code`, `hybrid`
- Los scopes disponibles
- Los property mappings (qué claims van en el token)

### Property Mapping

Define qué información del usuario se incluye en el `id_token` y/o `access_token`. Por default, Authentik expone:
- `sub` (user id)
- `email`, `email_verified`
- `name`, `preferred_username`, `nickname`
- `groups`

Vamos a agregar uno custom:
- `lyr_tenant`: slug del tenant (de la URL del request, o del grupo del user)

## Flujo de login (Authorization Code + PKCE)

```
┌─────────┐                ┌─────────────┐                ┌──────────────┐
│ Browser │                │ Admin (App) │                │  Authentik   │
└────┬────┘                └──────┬──────┘                └──────┬───────┘
     │                            │                              │
     │  1. Click "Login"          │                              │
     ├───────────────────────────▶│                              │
     │                            │                              │
     │  2. Redirect to Authentik  │                              │
     │     /authorize?...         │                              │
     │     + code_challenge       │                              │
     │◀───────────────────────────┤                              │
     │                                                          │
     │  3. GET /authorize?...                                     │
     ├─────────────────────────────────────────────────────────▶│
     │                                                          │
     │  4. Login page (user/password)                            │
     │◀─────────────────────────────────────────────────────────┤
     │                                                          │
     │  5. Submit credentials                                    │
     ├─────────────────────────────────────────────────────────▶│
     │                                                          │
     │  6. Validates, creates session                            │
     │                                                          │
     │  7. Redirect back to app with ?code=xxx                  │
     │◀─────────────────────────────────────────────────────────┤
     │                            │                              │
     │  8. Follow redirect        │                              │
     ├───────────────────────────▶│                              │
     │                            │                              │
     │                            │  9. POST /token             │
     │                            │     code + code_verifier     │
     │                            ├─────────────────────────────▶
     │                            │                              │
     │                            │  10. id_token (JWT)         │
     │                            │      access_token           │
     │                            │      refresh_token          │
     │                            │◀─────────────────────────────┤
     │                            │                              │
     │  11. Set session cookie    │                              │
     │  Redirect to dashboard     │                              │
     │◀───────────────────────────┤                              │
     │                                                          │
```

**Validación del JWT** (paso 11+): la app NO confía ciegamente en el token. Lo valida contra el JWKS público de Authentik:

```
GET https://auth.tudominio.com/application/o/<application-slug>/jwks/
```

Devuelve las claves públicas. La app cachea estas claves y verifica la firma del JWT localmente (sin llamada de red en cada request).

**Claims típicos del `id_token`**:

```json
{
  "sub": "abc123def456",
  "email": "user@empresa-a.com",
  "email_verified": true,
  "name": "Juan Pérez",
  "preferred_username": "juan",
  "lyr_tenant": "empresa-a",
  "iss": "https://auth.tudominio.com/application/o/admin-a/",
  "aud": "admin-a-client-id",
  "exp": 1700000000,
  "iat": 1699996400
}
```

## Modelo de seguridad

- **HTTPS obligatorio** en producción (Let's Encrypt vía Dokploy)
- **PKCE** en todos los flows (mitiga authorization code interception)
- **Refresh token rotation**: Authentik emite un nuevo refresh token en cada uso, el viejo queda invalidado
- **Tokens firmados** (RS256), la clave privada nunca sale de Authentik
- **Secret en cliente confidencial**: el `client_secret` de cada admin se guarda en Dokploy secrets, nunca en el repo
- **Logout**: revoke el refresh token en Authentik (RP-initiated logout)
- **Aud**: cada `id_token` lleva el `aud` correcto para el client que lo pidió

## Multi-tenancy en la app cliente

Cuando un admin recibe un token, valida:

1. **Firma**: la firma del JWT es válida (JWKS público de Authentik)
2. **Issuer**: el `iss` corresponde al tenant esperado (o se acepta y se confía en `lyr_tenant`)
3. **Audience**: el `aud` es su propio `client_id`
4. **Expiration**: `exp` no está vencido
5. **Tenant**: el claim `lyr_tenant` coincide con la empresa que el admin está gestionando (defensa en profundidad — la app debe tener su propia noción de tenant)

Si todo OK, extrae `sub`, `email`, `lyr_tenant` y autoriza.

## Stack y decisiones

| Capa | Tech | Justificación |
|---|---|---|
| IdP | Authentik | Multi-tenant nativo, OIDC estándar, Docker, UI moderna |
| DB | PostgreSQL (existente) | Authentik requiere Postgres, schema dedicado |
| Cache | Redis | Sessions, rate limiting, performance |
| Reverse proxy | Dokploy / Traefik | TLS termination, routing |
| SDK | `openid-client` + `jose` | Estándar OIDC, edge-compatible |
| Sesiones cliente | `iron-session` | Cookies firmadas cifradas, sin estado en servidor |

## Por qué OIDC estándar (y no custom)

OIDC es un estándar abierto. Esto significa:

- Si mañana querés migrar a Keycloak, Okta, Auth0 o WorkOS, **los admins no se enteran**. Solo cambia el deploy del IdP.
- Cualquier librería OIDC del ecosistema Node (openid-client, next-auth, passport, etc.) funciona.
- Podés integrar clientes de terceros (apps de partners) sin código custom.
- El ecosistema de OIDC (testing tools, postman, jwt.io) lo podés usar tal cual.

## Out of scope (decisiones conscientes)

- **MFA en MVP**: lo activamos cuando el cliente lo pida, configurable por tenant
- **Identity brokering** (Google/Microsoft login): no hay requerimiento
- **SAML**: no hay terceros
- **HA / replicación Postgres**: para < 5k usuarios, 1 instancia + backups alcanza
- **Branding custom por tenant**: tema default al inicio
