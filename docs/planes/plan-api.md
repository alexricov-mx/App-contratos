# Plan de trabajo — `contratos-api`

> Leer antes: [contrato-api.md](../contrato-api.md) (fuente de verdad), [requerimientos.md §6, §9, §11](../requerimientos.md), [casos-de-uso.md](../casos-de-uso.md) (CU-S*, CU-P*).

## Stack
.NET 10 · ASP.NET Core Minimal APIs · **Vertical Slice Architecture** · EF Core 10 + Npgsql · FluentValidation · ASP.NET Core Identity (hash) + JWT propio · Serilog · OpenAPI + Scalar · xUnit + Testcontainers (PostgreSQL) · Docker.

## Convenciones VSA
- Una carpeta por caso de uso: `Features/Clientes/Registrar/{Endpoint.cs, Request.cs, Validator.cs, Handler.cs}`. Sin capas de Services/Repositories genéricos.
- Cada slice registra su endpoint con un método de extensión (`MapRegistrarCliente`). Registro automático de slices por reflexión o lista explícita en `Program.cs`.
- Lógica compartida solo en `Common/` (auth, errores, auditoría, almacenamiento, conexión de BD).
- Errores como ProblemDetails con `code` (contrato-api.md §1).

---

## A1 · Base, BD y autenticación (día 1, mañana)

| # | Tarea | Aceptación |
|---|---|---|
| A1.1 | Solución, `CLAUDE.md`, `Dockerfile` multi-stage, CI (build + test + imagen a GHCR) | `docker build` OK |
| A1.2 | **Conexión dinámica de BD** (`Common/BaseDatos`): `IProveedorConexion` que lee `conexion-bd.json` cifrado (Data Protection con llaves en volumen) de `/app/config`; si no existe, usa `ConnectionStrings__Contratos` de entorno. `ContratosDbContext` creado vía `IDbContextFactory` + `DbDataSource` reemplazable en caliente | Prueba: cambiar el proveedor apunta a otra BD sin reiniciar |
| A1.3 | Modelo EF completo (requerimientos §11): UUID como PK, `jsonb` para Delta y alcance, `created_at/updated_at/deleted_at/version` (token de concurrencia), filtros globales de borrado lógico, snake_case | Migración inicial |
| A1.4 | Solo características compatibles con PostgreSQL administrado (sin extensiones; `gen_random_uuid()` no requerido: UUID se genera en .NET) | Revisión |
| A1.5 | Migraciones como **EF bundle** incluido en la imagen; al arrancar aplica migraciones pendientes si `MIGRAR_AL_INICIO=true` | Contenedor arranca contra BD vacía |
| A1.6 | Semilla: superadmin (desde variables de entorno), empresa, 2 productos, plantillas semilla (Delta de `contratos_pdf` D2.11, copiadas como JSON) | — |
| A1.7 | `Auth/Login`, `Auth/Refresh` (rotativo, guardado con hash), `Auth/Logout`, `Auth/2faVerificar` (TOTP) + políticas por rol | Pruebas de integración |
| A1.8 | Middleware: ProblemDetails, auditoría de escrituras, **modo mantenimiento** (503 + `Retry-After` en escrituras), `/health` | Pruebas |
| A1.10 | **Entorno local** según [entorno-local.md](../entorno-local.md): reutilizar `deploy/docker-compose.dev.yml` (ya existe, PostgreSQL con BD `contratos`; entorno con **Podman**), perfil `Development` (puerto 5080, CORS `http://localhost:5090`, usuarios semilla), carpeta `.local/` ignorada, `scripts/dev.ps1` | `.\scripts\dev.ps1` levanta BD + API y `/scalar` abre en el navegador |
| A1.9 | `IAlmacenamientoArchivos` con implementación `Volumen` (cifrado AES-GCM en reposo, ruta `/app/archivos`) y `GET /archivos/{id}` auditado | Pruebas |

## A2 · CRM, catálogo y plantillas (día 1, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| A2.1 | `Clientes/*` (listar con filtros, obtener, crear, actualizar con `version`, borrar lógico, identificación multipart) + cálculo `estadoRegistro`/`camposFaltantes` idéntico a `contratos_modelos` D1.4 | Pruebas con los mismos casos que D1.4 |
| A2.2 | `Clientes/Fusionar` (reasigna eventos, contactos, interacciones) | Prueba |
| A2.3 | `Contactos/*`, `Interacciones/*` | Pruebas |
| A2.4 | `Productos/*` con regla OFFLINE/ONLINE–periodicidad | Pruebas |
| A2.5 | `Plantillas/*` según contrato-api.md §7: borrador, validar, publicar (versión inmutable), versiones, duplicar, retirar, `variables`. Validador C# con **las mismas reglas** que D2.2 | Pruebas con los mismos casos que D2.2 |
| A2.6 | `Usuarios/*`, `Empresa/*` | Pruebas |

## A3 · Sincronización y eventos (día 2, mañana)

| # | Tarea | Aceptación |
|---|---|---|
| A3.1 | `Sync/ClientesPush`: multipart, resolución por campo según `actualizadoEn`, resultados `APLICADO/FUSIONADO/RECHAZADO`, conflictos a auditoría | Pruebas de conflicto |
| A3.2 | `Sync/ClientesPull` y `Sync/Catalogo` con marca `desde` y paginación; catálogo solo versiones `PUBLICADA` | Pruebas |
| A3.3 | `Sync/FirmaVendedor` | Prueba |
| A3.4 | `Eventos/Recibir` (contrato-api.md §5.1): validación, **recalcular SHA-256**, idempotencia por `id` (mismo hash → 200; otro hash → 409), transacción única (cliente + identificación + evento + productos + firmas + archivos), `recibidoEn`, efectos por tipo (seguimiento, suscripción) | Pruebas: nuevo, reenvío, hash distinto, PDF corrupto, cliente incompleto, ENTREGA actualiza seguimiento |
| A3.5 | `Eventos/Estado`, `Eventos/Listar`, `Eventos/Obtener` | Pruebas |
| A3.6 | Registrar `ultimaConexion` por dispositivo (para el tablero) | — |

## A4 · Seguimiento, pagos e infraestructura (día 2, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| A4.1 | `Seguimiento/*` y tareas; `entregado-manual` con justificación (ADMIN) | Pruebas |
| A4.2 | `Pagos/*` (multipart con comprobante) y `Suscripciones/*` (suspender/reactivar/cancelar) | Pruebas |
| A4.3 | `Tablero` con los contadores de contrato-api.md §6 | Prueba |
| A4.4 | `Integridad/*` | Prueba |
| A4.5 | `Infraestructura/Bd`: obtener, **probar** (conectar + leer `__EFMigrationsHistory` → VACIO/COMPATIBLE/DESACTUALIZADO/INCOMPATIBLE), **cambiar** como operación en segundo plano (`BackgroundService` + tabla/archivo de operaciones): mantenimiento → (si MIGRAR) `pg_dump \| pg_restore` con cliente `postgresql-client` incluido en la imagen → aplicar migraciones → conteos → guardar config anterior → cambiar `DbDataSource` → salir de mantenimiento; `revertir` | Prueba de integración con **dos** contenedores PostgreSQL (Testcontainers) |
| A4.6 | `Infraestructura/Respaldos`: listar y lanzar respaldo inmediato | Prueba |
| A4.7 | Exigir `X-Token-Elevado` en `/infraestructura/*` | Prueba |

## A5 · Despliegue en OVH (día 2, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| A5.1 | `deploy/docker-compose.yml`: `caddy` (TLS automático para `api.` y `portal.`), `api` (volúmenes `config`, `archivos`, `llaves`), `portal` (imagen de `contratos-portal`), `postgres:17` (volumen, **sin puerto publicado**), `respaldos` | `docker compose config` válido |
| A5.2 | `deploy/.env.example` con todas las variables documentadas | — |
| A5.3 | `deploy/backups/`: `pg_dump` diario + tar de archivos → OVH Object Storage (S3, `rclone` o `aws-cli`), retención 30 días | Respaldo manual exitoso |
| A5.4 | `deploy/db-migracion/`: script manual equivalente al flujo A4.5 para emergencias | — |
| A5.5 | `deploy/README.md`: preparar VPS (Docker, firewall solo 22/80/443, usuario sin root), primer arranque, actualizar, restaurar respaldo | Seguido paso a paso en el VPS |
| A5.6 | Despliegue en el VPS OVH y prueba `/health` por HTTPS | H4 |

## Comandos de verificación
```
dotnet build
dotnet test
docker build -t contratos-api .
docker compose -f deploy/docker-compose.yml up -d
curl https://api.<dominio>/api/v1/health
```
