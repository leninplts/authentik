# lyr-auth — Deploy de Authentik

Deploy del Identity Provider en Dokploy usando Docker Compose.

## Prerrequisitos

- Servidor con Dokploy corriendo
- Dominio apuntando al servidor (ej: `auth.tudominio.com`)
- TLS configurado (Let's Encrypt vía Dokploy)
- PostgreSQL accesible desde el servidor (puede ser el contenedor incluido o uno externo)

## Opción A — Postgres externo (recomendado)

Usar `docker-compose.yml` (sin Postgres) si ya tenés un Postgres corriendo en el servidor.

### 1. Crear la base de datos

```sql
CREATE DATABASE authentik;
CREATE USER authentik WITH PASSWORD '<password>';
GRANT ALL PRIVILEGES ON DATABASE authentik TO authentik;
```

### 2. Configurar variables de entorno

```bash
cd docker
cp .env.example .env
```

Editar `.env` y completar:

- `AUTHENTIK_HOST`: URL pública del IdP (ej: `https://auth.tudominio.com`)
- `AUTHENTIK_SECRET_KEY`: `openssl rand -base64 60`
- `AUTHENTIK_POSTGRESQL__HOST`: host del Postgres
- `AUTHENTIK_POSTGRESQL__USER`, `AUTHENTIK_POSTGRESQL__PASSWORD`
- `AUTHENTIK_POSTGRESQL__NAME`: `authentik` (o el nombre que le hayas puesto)
- `AUTHENTIK_PORT_HTTP` y `AUTHENTIK_PORT_HTTPS`: dejar default salvo necesidad real

### 3. Levantar en Dokploy

En Dokploy:

1. Crear nuevo proyecto → Docker Compose
2. Apuntar al path `docker/docker-compose.yml`
3. Configurar dominio `auth.tudominio.com` → servicio `server`, puerto 9000 (HTTP) o 9443 (HTTPS)
4. Deploy

O desde CLI si tenés acceso al servidor:

```bash
cd docker
docker compose up -d
```

### 4. Verificar

```bash
docker compose ps
docker compose logs -f server
```

Esperar a ver logs de "Application startup complete".

### 5. Setup inicial

Abrir `https://auth.tudominio.com/if/flow/initial-setup/`.

El sistema te pide crear la contraseña del usuario `akadmin` (superusuario). **Guardar bien estas credenciales** — son el escape hatch si perdés acceso.

## Opción B — Postgres incluido (alternativa)

Usar `docker-compose.with-postgres.yml` si querés que el stack sea 100% autocontenido.

Pasos iguales pero:

- En el `.env` hay que agregar `PG_DB`, `PG_USER`, `PG_PASS` (no son las vars `AUTHENTIK_POSTGRESQL__*`)
- El volumen `postgres-data` se crea automáticamente en el path de Dokploy
- **IMPORTANTE**: los backups del Postgres deben incluir este volumen

## Configuración en Dokploy

### Dominio y TLS

- Apuntar `auth.tudominio.com` (o el subdominio elegido) al servicio `server`
- Dokploy maneja TLS automáticamente si tenés Let's Encrypt configurado
- **OIDC en producción requiere HTTPS** — no usar HTTP plano

### Healthcheck

Dokploy puede usar el healthcheck del container (`ak healthcheck`). Si el server no responde, Dokploy lo marca como unhealthy.

### Backups

- **Volumen `redis-data`**: cache, se puede perder sin drama
- **Volumen `authentik/media`**: avatars, logos subidos, branding
- **DB de Authentik** (Postgres): CRÍTICO, hacer backup diario

Cron sugerido para backup del Postgres (Opción A):

```bash
0 3 * * * pg_dump -h <host> -U <user> authentik | gzip > /backups/authentik-$(date +\%F).sql.gz
```

## Estructura de archivos

```
docker/
├── docker-compose.yml                # Stack principal (Postgres externo)
├── docker-compose.with-postgres.yml  # Stack con Postgres incluido
├── .env.example                      # Template de configuración
└── authentik/
    ├── media/                        # Avatars, logos (persistente)
    ├── certs/                        # Certs generados (persistente)
    ├── custom-templates/
    │   └── login/
    │       └── login.html            # Tema de login
    └── blueprints/
        └── default/                  # Blueprints aplicados al startup
            ├── tenants/
            │   └── empresa-piloto.yaml
            └── scope-tenant.yaml
```

## Próximos pasos

Una vez que el IdP esté funcionando:

1. **Fase 2**: crear el primer tenant de prueba y una Application OIDC
2. **Fase 3**: desarrollar el `lyr-auth-sdk` para que los admins se integren
3. Ver [PLAN.md](../PLAN.md) para el plan completo
