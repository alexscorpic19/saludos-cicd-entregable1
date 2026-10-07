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
