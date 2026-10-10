# Bitácora de errores — Entregable 1

> Registro de cada error encontrado: qué pasó → mensaje exacto → causa → solución.
> Alimenta la **pregunta 5** de `Entregable1_Respuestas.md`.

Plantilla:

| # | Fecha | Fase | Error / mensaje exacto | Causa | Solución |
|---|---|---|---|---|---|
| 1 |  |  |  |  |  |

---

## 1. Postgres "no existe" durante el smoke test

- **Fecha:** 07/10/2026
- **Fase:** 0 — validación local de los smoke tests de CI
- **Qué pasó:** el job `Test` fallaba al consultar la base de datos recién creada.
- **Mensaje exacto:** `psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL: database "saludos" does not exist`
- **Causa raíz:** `pg_isready` devuelve éxito apenas el servidor temporal de inicialización
  de la imagen `postgres` escucha el socket, **antes** de que se cree la base `saludos`
  y se ejecute `init.sql` (condición de carrera).
- **Solución:** esperar con una consulta real contra la tabla
  (`psql -d saludos -tAc "SELECT count(*) FROM saludos"`) antes de continuar; eso además
  valida que la semilla de `init.sql` se aplicó.

---

## 2.- **Fecha:**
- **Fase:**
- **Qué pasó:**
- **Mensaje exacto:**
- **Causa raíz:**
- **Solución:**
## 2. AWS rechaza nombres de security group que empiezan por `sg-`

- **Fecha:** 08/10/2026
- **Fase:** 1 — Security Groups
- **Mensaje exacto:** `InvalidParameterValue: Value (sg-alb) for parameter GroupName is invalid. Group names may not be in the format sg-*.`
- **Causa raíz:** AWS reserva el prefijo `sg-` para los identificadores de grupo; la guía
  proponía `sg-alb`, `sg-frontend`, `sg-backend`, `sg-rds` como **nombres**.
- **Solución:** crearlos como `saludos-alb`, `saludos-frontend`, `saludos-backend`, `saludos-rds`.

## 3. Lanzamiento de EC2 bloqueado por validación de cuenta

- **Fecha:** 08/10/2026
- **Fase:** 1 — Compute
- **Mensaje exacto:** `PendingVerification: Your request for accessing resources in this region is being validated...`
- **Causa raíz:** AWS valida automáticamente cuentas nuevas antes de permitir más recursos en la región.
- **Solución:** esperar (llega correo de AWS) y reintentar `run-instances`; se resolvió en minutos.

## 4. Clave RDS perdida al limpiarse `/tmp`

- **Fecha:** 10/10/2026
- **Fase:** 2 — Env files
- **Qué pasó:** la contraseña generada se guardó en `/tmp/opencode/rds_password.txt` y el directorio se limpió antes de usarla.
- **Causa raíz:** almacenamiento temporal fuera del home.
- **Solución:** rotar la clave maestra con `rds modify-db-instance --master-user-password ... --apply-immediately`
  y guardarla en `~/.secrets/entregable1/rds_password.txt` (chmod 600); verificar con `psql` desde `backend-1`.

## 5. `scp` falla con "Permission denied" al destino

- **Fecha:** 10/10/2026
- **Fase:** 2 — Llave SSH
- **Mensaje exacto:** `scp: dest open "/home/ubuntu/saludos-key.pem": Permission denied`
- **Causa raíz:** el `scp` de OpenSSH 9 usa SFTP y no pudo crear el archivo en el destino.
- **Solución:** subir por tubería: `ssh ubuntu@fe-1 'umask 077; cat > ~/.ssh/saludos-key.pem' < saludos-key.pem`.

## 6. El job `Verify` del CD no alcanza a `frontend-2` (el más relevante)

- **Fecha:** 10/10/2026
- **Fase:** 3 — CD
- **Mensaje exacto:** `curl: (28) Failed to connect to 10.0.0.98 port 8080 after 132797 ms: Connection timed out`
- **Qué pasó:** `Pull` y `Deploy` salieron verdes, pero `Verificacion` falló al hacer
  `curl http://<fe-2>:8080/` desde el runner (`frontend-1`).
- **Causa raíz:** la regla de `saludos-frontend` permitía 8080 **solo desde `saludos-alb`**;
  el runner sale con su propia IP, no con la del ALB → el firewall descartaba el paquete.
- **Solución:** añadir en `saludos-frontend` la regla *TCP 8080 desde la propia SG*
  (`fe-1 → fe-2`), igual que ya existía para el puerto 22, y relanzar el CD → verde.
