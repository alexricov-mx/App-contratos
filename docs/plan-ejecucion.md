# Plan de ejecución por fases — App-Contratos

> Versión 0.3 · 2026-09-24 · Referencias: [requerimientos.md](requerimientos.md), [casos-de-uso.md](casos-de-uso.md)

**Principios**
- Cada fase termina con algo que se puede probar de punta a punta.
- La API y la infraestructura van primero porque App y Portal dependen de ellas.
- La **configuración dinámica de la BD** se diseña desde la Fase 1 (conexión leída de configuración externa, esquema compatible con PostgreSQL administrado), aunque la pantalla del Portal llegue al final.
- Mientras el Portal no exista, catálogo, plantillas y usuarios se cargan con **datos semilla** y la documentación OpenAPI.

Proyectos: **API** (.NET 10) · **App** (Flutter móvil) · **Portal** (web) · **Infra** (Docker en VPS).

---

## Fase 0 · Fundamentos e infraestructura (1–2 semanas)

| Proyecto | Tarea | Entregable |
|---|---|---|
| — | Decidir tecnología del Portal (Flutter Web recomendado / Blazor) y proveedor del VPS | Nota de decisión |
| Repo | Estructura `app/`, `portal/`, `api/`, `infra/`, `packages/`, `docs/`; git, CI (build + tests por proyecto) | Repo base |
| Infra | `docker-compose.yml`: proxy (Caddy, TLS), API, PostgreSQL (volumen, sin puerto público), respaldos | Stack levantado en el VPS |
| API | Proyecto .NET 10 Minimal APIs con estructura VSA, EF Core + Npgsql, Serilog, OpenAPI/Scalar, `/health` | API desplegada en contenedor |
| App | Proyecto Flutter con drift (SQLCipher), riverpod, go_router, dio | App compila en el Samsung |
| App | **Prototipo del lienzo con S Pen** (solo stylus, presión, trazo vectorial) en teléfono y tableta | Demo validada |
| Legal | **Enviar [contrato-base.md](contrato-base.md) a revisión de abogado** (en paralelo) | Plantillas validadas |

**Criterio de salida:** la API responde `/health` por HTTPS desde el VPS y el lienzo funciona bien con el S Pen en ambos dispositivos.

---

## Fase 1 · API núcleo (2 semanas)

| Slice / tarea | Detalle |
|---|---|
| **Configuración de BD desacoplada** | Proveedor de cadena de conexión leído de archivo cifrado en volumen (+ variables de entorno como valor inicial); `DbContext` creado vía fábrica que usa la conexión activa; SSL configurable |
| Migraciones | Esquema inicial completo (§11 de requerimientos); migraciones como **EF bundle** ejecutable contra cualquier servidor |
| `Auth/*` | Login, refresh, roles, hash de contraseñas |
| `Usuarios/*` | CRUD básico + semilla de superadmin |
| `Clientes/*`, `Identificaciones/*` | CRUD con estado de registro |
| `Catalogo/*`, `Plantillas/*` | CRUD + versionado de plantillas + semilla de las 2 plantillas base |
| Almacenamiento | `IAlmacenamientoArchivos` con implementación en volumen, cifrado en reposo |
| Auditoría | Registro de escrituras y accesos a archivos sensibles |
| Tests | Pruebas de integración con PostgreSQL en contenedor (Testcontainers) |

**Criterio de salida:** con Scalar/Swagger se puede crear usuarios, clientes, productos y plantillas en el PostgreSQL del VPS.

---

## Fase 2 · App: datos locales, clientes y sincronización CRM (3 semanas)

**Casos de uso:** CU-A01, CU-A02, CU-A03, CU-A04, CU-A05, CU-S02

| Proyecto | Feature |
|---|---|
| App | Esquema SQLite (drift) y **capa de repositorios**: toda escritura va a SQLite |
| App | Login, sesión persistente, PIN/biometría |
| App | Firma del vendedor con S Pen |
| App | Pre-registro rápido y registro completo con **captura de identificación** por cámara |
| App | Pantalla de sincronización: subir clientes, bajar clientes, bajar catálogo y plantillas; marca de última sincronización |
| API | `Clientes/Sync` (push/pull con resolución por campo) y `Catalogo/Sync` (pull) |

**Criterio de salida:** un cliente pre-registrado en modo avión aparece en PostgreSQL tras sincronizar, y un cambio hecho en la BD llega al dispositivo.

---

## Fase 3 · App: evento de venta, PDF y firma (3 semanas)

**Casos de uso:** CU-A06, CU-A07

| Feature | Detalle |
|---|---|
| Asistente de venta | Cliente → productos → ajustes → vista previa |
| Motor de plantillas + PDF | Variables, bloques offline/online, anexo de alcance, folio, paginación |
| Modo presentación | Pantalla bloqueada para el comprador; lectura completa obligatoria; casillas de aceptación |
| Firma con S Pen | Lienzo del prototipo de Fase 0; imagen + trazo vectorial; alternativa con dedo |
| Firma de un solo uso | Ligada al hash; sin reutilización; eliminación si se cancela |
| Confirmación del vendedor | PIN/biometría |
| Evidencias y constancia | Geolocalización, dispositivo, identificación, selfie opcional; SHA-256; hoja de constancia |
| Compartir | WhatsApp/correo desde Android |

**Criterio de salida:** en modo avión se cierra una venta completa y se obtiene el PDF firmado con constancia.

---

## Fase 4 · Envío de eventos App → API (2 semanas) — **MVP de punta a punta**

**Casos de uso:** CU-A08, CU-A09, CU-S01

| Proyecto | Feature |
|---|---|
| API | `Eventos/Recibir` multipart, idempotente por UUID, verificación de hash, transacción única, fecha de recepción, creación de seguimiento/suscripción |
| API | `Eventos/Estado` para confirmar recepción |
| App | Servicio de envío: automático al firmar, reintento con espera progresiva al recuperar conexión (monitoreo de conectividad + tarea en segundo plano) |
| App | Pantalla **Eventos pendientes de envío** con envío manual individual y masivo, motivo del error |
| App | Indicador en la pantalla principal (en línea / sin conexión / N pendientes) |
| App | Política de depuración local de eventos ya enviados |

**Criterio de salida:** una venta firmada sin red se envía sola al volver la conexión; si se fuerza un error, se puede reenviar a mano; reenviar dos veces no duplica.

---

## Fase 5 · Portal: CRM, contactos y eventos (3 semanas)

**Casos de uso:** CU-P01, CU-P02, CU-P03, CU-P04, CU-P07, CU-P08, CU-P09

| Proyecto | Feature |
|---|---|
| Portal | Login, estructura de navegación, roles |
| Portal | Tablero inicial |
| Portal | Clientes: listado, ficha, **bandeja por completar**, fusión de duplicados |
| Portal | **Contactos** e **interacciones** con próximos seguimientos |
| Portal | Eventos de compra: listado, detalle, PDF, constancia, evidencias |
| Portal | Catálogo, plantillas (editor + versionado), usuarios |
| Portal | Verificador de integridad |
| API | Slices `Contactos/*`, `Interacciones/*`, `Eventos/Consultar`, `Tablero/*`, `Clientes/Fusionar` |

**Criterio de salida:** la oficina completa un pre-registro hecho en campo y el cambio llega a la App; se consulta cualquier venta con su PDF.

---

## Fase 6 · Seguimiento de entrega, pagos y suscripciones (2–3 semanas)

**Casos de uso:** CU-A10, CU-P05, CU-P06

| Proyecto | Feature |
|---|---|
| App | Eventos `ENTREGA`, `MODIFICACION`, `CANCELACION` ligados a una venta (con firma nueva) |
| API | `Seguimiento/*`, `Pagos/*`, `Suscripciones/*`; transiciones automáticas al recibir `ENTREGA`/`CANCELACION` |
| Portal | Tablero de seguimiento de entrega: estados, responsable, fecha compromiso, tareas, atrasos |
| Portal | Registro de pagos con comprobante; gestión de suscripciones; alertas |

**Criterio de salida:** una venta pasa de `RECIBIDO` a `ENTREGADO` al firmar el acta en campo, y el tablero muestra entregas atrasadas y pagos vencidos.

---

## Fase 7 · Infraestructura administrable (1–2 semanas)

**Casos de uso:** CU-P10, CU-S03

| Proyecto | Feature |
|---|---|
| API | `Infraestructura/BaseDatos`: probar conexión, verificar esquema, modo mantenimiento, cambio en caliente, verificación, revertir |
| API / Infra | Proceso guiado `pg_dump`/`pg_restore` con progreso |
| Portal | Pantalla de configuración de BD (superadmin + 2FA), almacenamiento y respaldos |
| Infra | Respaldos automáticos a almacenamiento externo; prueba de restauración documentada |
| — | **Simulacro**: migrar a un PostgreSQL administrado de prueba y regresar |

**Criterio de salida:** el simulacro de migración se completa desde el Portal sin perder eventos (la App los encola durante el mantenimiento).

---

## Fase 8 · Refuerzo legal y extras (según prioridad)

| Feature | Valor |
|---|---|
| Verificación OTP del comprador (correo/SMS) | Refuerza la identidad del firmante |
| Constancia **NOM-151** vía Prestador de Servicios de Certificación | Máxima fuerza probatoria |
| Pasarela de pago recurrente (Stripe / Mercado Pago / Conekta) | Cobro automático de suscripciones |
| Facturación CFDI (PAC con API, o integración en un solo sentido con Odoo) | Facturar ventas y suscripciones |
| Almacenamiento de objetos (S3/Blob) | Sacar archivos del VPS |
| Portal del cliente final | Autoservicio |

---

## Resumen de tiempos (estimado, 1 desarrollador)

| Fase | Duración | Acumulado |
|---|---|---|
| 0 · Fundamentos e infraestructura | 1–2 sem | 2 sem |
| 1 · API núcleo | 2 sem | 4 sem |
| 2 · App: datos, clientes, sync CRM | 3 sem | 7 sem |
| 3 · App: venta, PDF, firma | 3 sem | 10 sem |
| 4 · Envío de eventos (**MVP**) | 2 sem | 12 sem |
| 5 · Portal: CRM, contactos, eventos | 3 sem | 15 sem |
| 6 · Entrega, pagos, suscripciones | 2–3 sem | 18 sem |
| 7 · Infraestructura administrable | 1–2 sem | 20 sem |
| 8 · Extras | variable | — |

**MVP en campo con respaldo central:** fin de Fase 4 (~12 semanas).
**Versión 1.0:** fin de Fase 7 (~20 semanas).
