# Entregable No. 1 — Diplomado DevOps (Universidad del Rosario)

Aplicación **saludos** de tres capas (Frontend → Backend → Base de datos) desplegada en
**AWS** y automatizada con **GitHub Actions** sobre un **self-hosted runner**.

## Arquitectura

```
Internet → ALB (HTTP 80)
             ├─ frontend-1 (t3.small, subnet pública, corre el runner CI/CD)  host:8080
             └─ frontend-2 (t3.small, subnet pública)                         host:8080
                    └─ /api/* → backend-1 (t3.small, subnet privada)          :5000
                                                  └─ RDS PostgreSQL (privado)  :5432
```

- **VPC** `saludos-vpc` `10.0.0.0/24` con 4 subredes `/26` en 2 AZ
  (2 públicas, 2 privadas), Internet Gateway, NAT Gateway y route tables.
- **Seguridad:** el backend solo acepta 5000 desde el SG de los frontends; el ALB solo
  80 desde Internet; RDS solo 5432 desde el backend; el runner sale a Internet por la
  subred pública.
- La base de datos **no se despliega con CI/CD**: es **RDS gestionado** (el CD despliega
  backend y frontends, en ese orden, esperando la salud del backend antes de continuar).

## Pipelines

| Workflow | Disparador | Jobs |
|---|---|---|
| `ci.yml` | push a `develop` o `main` | Code → Build → Test → Release → Push |
| `cd.yml` | `workflow_run` cuando CI termina en `main`, o manual | Release - Pull → Deploy → Verificación |

- **CI** construye `saludos-database`, `saludos-backend` y `saludos-frontend`, los prueba
  con smoke tests reales (BD con `init.sql`, backend contra BD, frontend + proxy `/api/`)
  y los publica en Docker Hub con **tag único = SHA de git** + `latest`.
- **CD** solo **hace pull** (nunca reconstruye), despliega por SSH y verifica con
  `curl` hasta el endpoint expuesto por el ALB.
- **Rollback:** *Actions → CD → Run workflow* y escribir el SHA de la versión deseada.

## Estructura

```
.github/workflows/   ci.yml, cd.yml
backend/             Flask + psycopg2 (config por variables de entorno)
db/                  PostgreSQL 16 + init.sql (semilla de saludos)
frontend/            nginx + plantilla con proxy /api/ → backend
.env.dev             configuración del ambiente develop (sin secretos)
.env.prod            configuración del ambiente production (sin secretos)
BITACORA.md          registro de errores y soluciones
```

## Despliegue manual (si se necesita sin CI/CD)

```bash
# backend-1 (subnet privada)
docker run -d --name saludos-backend --restart unless-stopped -p 5000:5000 \
  --env-file /home/ubuntu/saludos-backend.env \
  alexunder1985/saludos-backend:latest

# frontend-1 y frontend-2 (subnet publica)
docker run -d --name saludos-frontend --restart unless-stopped -p 8080:80 \
  --env-file /home/ubuntu/saludos-frontend.env \
  alexunder1985/saludos-frontend:latest
```

## Seguridad

- Secretos en **GitHub Secrets** (Docker Hub token, llave SSH en base64) y variables en
  **Actions Variables** (IPs privadas, DNS del ALB).
- La contraseña de RDS **no** está en el repo: vive en `~/saludos-backend.env` de
  `backend-1` (permisos 600) y/o en GitHub Secrets.
- La llave SSH nunca se versiona (`.gitignore` bloquea `*.pem`).
