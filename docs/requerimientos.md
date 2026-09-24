# Requerimientos — App-Contratos

> Versión 0.4 · 2026-09-24 · Estado: borrador para revisión

## 0. Decisiones confirmadas

| Tema | Decisión |
|---|---|
| Jurisdicción | **México** |
| Clientes objetivo | **Negocios locales pequeños** (micro y pequeñas empresas, personas físicas con actividad empresarial) |
| Proyectos | **3 proyectos, un repositorio cada uno**: App de campo, API (incluye despliegue) y Portal, más un repositorio de paquetes Dart compartidos. La documentación vive en su propio repositorio (ver §10) |
| App de campo | Flutter móvil, dispositivos **Samsung con S Pen** (teléfono y tableta) |
| BD local (App) | **SQLite** (drift). Todo se guarda primero aquí |
| API | **.NET 10**, **Vertical Slice Architecture**, en **contenedor** en el VPS. **Único punto de acceso a PostgreSQL** |
| Hospedaje | **OVHcloud**: VPS con Docker (v1); a futuro OVH Managed PostgreSQL y OVH Object Storage (S3 compatible) |
| BD central | **PostgreSQL en contenedor** en el VPS (v1), preparada para migrar a un **PostgreSQL administrado** |
| Portal de administración | **Flutter Web**. CRM, eventos de compra, seguimiento de entrega, contactos, **editor de contratos**, configuración |
| Integración App ↔ API | **Envío de eventos** (no sincronización total) + **sincronización de clientes del CRM** y catálogo |
| Firma del comprador | **De un solo uso**: se firma de nuevo en cada evento/documento |
| Identificación | Pre-registro sin identificación; **registro completo exige identificación oficial** |
| Odoo | No en v1 (ver §7.5) |
| Construcción | Desarrollo asistido por Claude a partir de esta documentación, [contrato-api.md](contrato-api.md) y los [planes por repositorio](planes/) |

## 1. Visión

Sistema para vender aplicaciones (offline con pago único y online por suscripción) a negocios locales, en el que:

1. El vendedor, **en sitio y aunque no haya internet**, registra al cliente, arma la venta, genera el contrato en PDF y recaba la firma del comprador con el **S Pen**, todo desde la **App**.
2. Al cerrar la venta, la App envía el **evento de compra** completo a la **API**, que lo guarda en **PostgreSQL**.
3. El equipo de oficina, desde el **Portal**, da seguimiento: CRM, contactos, eventos de compra, entrega, pagos y suscripciones.
4. Los documentos firmados, con su bitácora de evidencias, sirven como **respaldo legal** de la transacción.

## 2. Modelo de negocio

| Tipo de producto | Modalidad de cobro | Contrato |
|---|---|---|
| **Aplicación offline** | **Pago único** (licencia perpetua de uso) | Contrato de desarrollo y licencia |
| **Aplicación online** | **Suscripción obligatoria** (mensual o anual) | Contrato de servicio SaaS |

- **RN-01** Toda aplicación online es de suscripción; no existe pago único para online.
- **RN-02** Toda aplicación offline es de pago único; soporte/actualizaciones posteriores se venden aparte (póliza opcional).
- **RN-03** Cada contrato lleva un **anexo de alcance** con las funcionalidades incluidas; lo que no esté ahí queda fuera.
- **RN-04** Cada contrato incluye limitación de responsabilidad sobre la **calidad de la información** del cliente y sobre el **uso distinto** al pactado (ver [contrato-base.md](contrato-base.md)).
- **RN-05** La firma del comprador es **de un solo uso**: vale solo para el documento del evento en que se capturó.

## 3. Concepto central: el Evento

Un **evento** es cada interacción formal con el comprador en sitio que termina en un documento firmado. Tipos:

| Tipo | Documento que genera | Cuándo |
|---|---|---|
| `VENTA` | Contrato (plantilla A o B) | Cierre de la compra |
| `ENTREGA` | Acta de entrega-recepción | Al entregar/instalar la aplicación |
| `MODIFICACION` | Convenio modificatorio | Cambio de alcance, precio o plazo |
| `CANCELACION` | Convenio de terminación | Fin anticipado del contrato |

Cada evento tiene su propia firma del comprador (RN-05). Los eventos `ENTREGA`, `MODIFICACION` y `CANCELACION` se ligan al evento `VENTA` original.

**Ciclo de vida en la App:**

```
EN_CURSO ──► FIRMADO ──► ENVIANDO ──► ENVIADO
   │                        │
   └─► DESCARTADO           └─► ERROR_ENVIO ──(reintento auto/manual)──► ENVIANDO
```

**Ciclo de vida en el servidor (seguimiento, evento `VENTA`):**

```
RECIBIDO ──► EN_PROCESO ──► LISTO_PARA_ENTREGA ──► ENTREGADO ──► CERRADO
                                                        │
    (cualquier estado) ──► CANCELADO                    └─► (online) suscripción ACTIVA
```

## 4. Actores

| Actor | Usa | Descripción |
|---|---|---|
| **Vendedor de campo** | App | Registra clientes, crea eventos, recaba firmas. |
| **Equipo de oficina** | Portal | Da seguimiento a CRM, contactos, eventos, entregas y pagos. |
| **Administrador** | Portal | Todo lo anterior + catálogo, plantillas, usuarios. |
| **Superadministrador** | Portal | Configuración de infraestructura (conexión a BD, respaldos). |
| **Comprador** | App (en modo presentación) | Revisa y firma. No tiene cuenta. |

## 5. Proyecto 1 — App de campo (Flutter)

### 5.1 Autenticación y perfil
- **RFA-01** Login con correo y contraseña contra la API; sesión persistente para trabajar **offline** después del primer login.
- **RFA-02** Desbloqueo local con PIN o biometría.
- **RFA-03** Captura de la **firma del vendedor** con S Pen (una vez). Se sube a la API y se descarga cifrada a los dispositivos del vendedor. Se estampa solo tras confirmar con PIN/biometría **en cada documento**.

### 5.2 Capa de datos local (SQLite)
- **RFA-10** **Toda escritura va primero a SQLite**, haya o no conexión. La UI nunca espera a la API para guardar.
- **RFA-11** Todos los registros llevan **UUID generado en el dispositivo**, de modo que el mismo ID se conserva en PostgreSQL.
- **RFA-12** Base local **cifrada** (SQLCipher). Archivos (PDF, firmas, identificaciones) en almacenamiento privado de la app, cifrados.
- **RFA-13** Una capa de repositorios encapsula SQLite; los servicios de envío y sincronización leen de ahí, no de la UI.

### 5.3 Clientes (CRM local)
- **RFA-20** **Pre-registro rápido**: nombre + teléfono o correo. Estado `INCOMPLETO`. No requiere identificación.
- **RFA-21** **Registro completo**: persona física o moral, nombre/razón social, RFC, domicilio, teléfono, correo y **identificación oficial** (INE, pasaporte o cédula; foto de anverso y reverso, número, vigencia). Persona moral: representante legal + poder o acta.
- **RFA-22** No se permite `COMPLETO` con identificación vencida.
- **RFA-23** Búsqueda y consulta de clientes descargados del CRM.
- **RFA-24** Detección de posibles duplicados por teléfono/correo/RFC.

### 5.4 Sincronización de clientes y catálogo
- **RFA-30** Botón **“Sincronizar”** (y automático al iniciar sesión con conexión) que:
  1. **Sube** los clientes nuevos o modificados en el dispositivo (pre-registros y registros completos, con su identificación).
  2. **Descarga** los clientes del CRM cambiados desde la última sincronización (incluye lo completado por la oficina en el Portal).
  3. **Descarga** catálogo de productos y plantillas de contrato vigentes (necesarios para generar contratos sin conexión).
- **RFA-31** Conflictos en clientes: prevalece el cambio más reciente por campo; el servidor registra el conflicto en auditoría.
- **RFA-32** Muestra la fecha/hora de la última sincronización correcta.

### 5.5 Eventos de compra
- **RFA-40** Crear evento `VENTA`: seleccionar cliente + producto(s) → se prellenan precio, modalidad de pago (RN-01/02), alcance y plantilla.
- **RFA-41** Ajustes: precio negociado, anticipo, fecha compromiso de entrega, funcionalidades extra/excluidas, notas.
- **RFA-42** Vista previa del PDF.
- **RFA-43** Crear eventos `ENTREGA`, `MODIFICACION` y `CANCELACION` ligados a una venta existente.
- **RFA-44** Un evento no se puede firmar si el cliente no está `COMPLETO`; se permite completarlo en ese momento.

### 5.6 Firma y evidencia
- **RFA-50** **Modo presentación**: el comprador solo ve el documento y el lienzo de firma; no accede al resto de la app.
- **RFA-51** Lectura completa obligatoria y **casillas de aceptación**: alcance, limitación de responsabilidad, forma de pago, aviso de privacidad y uso exclusivo de la firma.
- **RFA-52** Lienzo de firma para **S Pen**: solo acepta trazos del lápiz (ignora palma/dedos), presión variable, pantalla completa horizontal, *Borrar* / *Aceptar*. Interruptor para firmar con dedo si no hay S Pen (queda registrado).
- **RFA-53** Se guarda imagen **y trazo vectorial** (coordenadas, presión, tiempo) de la firma.
- **RFA-54** **Firma de un solo uso**: ligada al hash del documento del evento; no se guarda como elemento reutilizable del cliente; no existe “usar firma anterior”; si el evento se descarta, la firma se elimina.
- **RFA-55** Confirmación del vendedor con PIN/biometría antes de estampar su firma.
- **RFA-56** Evidencias: fecha/hora del dispositivo, geolocalización, dispositivo, versión de app, tipo de entrada (`S_PEN`/`DEDO`), identificación vinculada; selfie opcional.
- **RFA-57** Se genera el PDF final con **hoja de constancia** (evidencias + folio + hash) y se calcula su **SHA-256**. El evento pasa a `FIRMADO` y es **inmutable**.
- **RFA-58** Compartir copia al comprador (hoja de compartir de Android: WhatsApp, correo) en el momento.

### 5.7 Envío de eventos a la API
- **RFA-60** Al pasar a `FIRMADO`, el evento se **envía automáticamente** a la API (cliente, datos del evento, PDF, firmas, trazos, evidencias).
- **RFA-61** Si no hay conexión o el envío falla, queda en `ERROR_ENVIO` y se **reintenta automáticamente** al detectar conexión (con espera progresiva).
- **RFA-62** Pantalla **“Eventos pendientes de envío”** con botón de **envío manual** por evento y “enviar todos”, mostrando el motivo del último error.
- **RFA-63** El envío es **idempotente**: reenviar el mismo evento no crea duplicados (se identifica por su UUID).
- **RFA-64** El evento pasa a `ENVIADO` solo cuando la API confirma que recibió todo y que **el hash del PDF coincide**.
- **RFA-65** Los eventos `ENVIADO` se conservan localmente N días (configurable) y luego se pueden depurar del dispositivo; los no enviados **nunca** se depuran.
- **RFA-66** Indicador visible en la pantalla principal: en línea / sin conexión / N eventos pendientes.

## 6. Proyecto 2 — API (.NET 10, VSA)

### 6.1 Principios
- **RFS-01** Es el **único** componente que se conecta a PostgreSQL. App y Portal solo hablan con la API por HTTPS.
- **RFS-02** Versionada (`/api/v1/...`), documentada con OpenAPI.
- **RFS-03** Autenticación JWT con refresh token; autorización por rol (`VENDEDOR`, `OFICINA`, `ADMIN`, `SUPERADMIN`).
- **RFS-04** Auditoría de todas las operaciones de escritura y de acceso a firmas/identificaciones.

### 6.2 Endpoints principales

| Slice | Método | Uso |
|---|---|---|
| `Auth/Login`, `Auth/Refresh` | POST | App y Portal |
| `Eventos/Recibir` | `POST /api/v1/eventos` (multipart) | App: crea el evento completo en PostgreSQL |
| `Eventos/Estado` | `GET /api/v1/eventos/{id}/estado` | App: confirmar recepción/hash |
| `Clientes/Sync` | `POST /api/v1/clientes/sync` · `GET /api/v1/clientes/sync?desde=` | App: subir/bajar clientes |
| `Catalogo/Sync` | `GET /api/v1/catalogo/sync?desde=` | App: productos y plantillas |
| `Clientes/*`, `Contactos/*`, `Eventos/*`, `Seguimiento/*`, `Pagos/*`, `Suscripciones/*`, `Productos/*`, `Plantillas/*`, `Usuarios/*` | CRUD | Portal |
| `Infraestructura/BaseDatos` | GET / POST probar / PUT | Portal (solo superadmin), ver §6.4 |
| `Salud` | `GET /health` | Monitoreo |

### 6.3 Recepción de eventos (`POST /api/v1/eventos`)
- **RFS-10** Recibe en una sola petición multipart: JSON del evento (incluye cliente) + PDF + imágenes/trazos de firma + selfie.
- **RFS-11** **Idempotente por UUID**: si el evento ya existe y el hash coincide, responde éxito sin duplicar; si existe con **otro hash**, responde conflicto y alerta a administración (posible alteración).
- **RFS-12** Recalcula el SHA-256 del PDF y lo compara con el declarado por la App.
- **RFS-13** Transacción única: cliente (alta o actualización) + evento + evidencias + archivos. Si algo falla, no se guarda nada.
- **RFS-14** Registra **fecha/hora de recepción del servidor** como evidencia adicional.
- **RFS-15** Si el evento es `VENTA` de producto online, crea la **suscripción** en estado pendiente de inicio.
- **RFS-16** Tamaño máximo configurable por petición (p. ej. 25 MB).

### 6.4 Conexión a la base de datos configurable
Objetivo: poder pasar del PostgreSQL en contenedor a un **PostgreSQL administrado** sin recompilar ni redesplegar, configurándolo desde el Portal.

- **RFS-20** La cadena de conexión **no** se guarda en la propia BD (no podría leerse si la BD cambia). Se guarda en un **archivo de configuración cifrado** en un volumen persistente del contenedor de la API (con respaldo en variables de entorno/secretos como valor inicial).
- **RFS-21** Parámetros configurables: host, puerto, base, usuario, contraseña, modo SSL (`Disable`/`Require`/`VerifyFull`), certificado CA, tamaño del pool, timeout.
- **RFS-22** Flujo seguro de cambio:
  1. **Probar conexión** al destino (sin aplicar cambios).
  2. **Verificar esquema**: versión de migraciones del destino; si está vacío, ofrece aplicar migraciones.
  3. Activar **modo mantenimiento** (la API responde 503 a escrituras; la App encola eventos, no pierde nada).
  4. Migración de datos (ver RFS-24).
  5. **Cambiar** la conexión activa en caliente (sin reiniciar el contenedor).
  6. Verificación posterior (conteos de registros) y salida de mantenimiento.
  7. La configuración anterior se conserva para **revertir**.
- **RFS-23** Solo `SUPERADMIN`, con reautenticación y 2FA; queda en auditoría (sin registrar contraseñas).
- **RFS-24** La copia de datos entre servidores se hace con `pg_dump`/`pg_restore` mediante un proceso guiado (script en `infra/`, o tarea lanzada por la API con seguimiento de progreso en el Portal). No se copia “registro por registro” desde la API.
- **RFS-25** Para que la migración sea posible, el esquema debe ser **compatible con PostgreSQL administrado** (ver RNF-20).

### 6.5 Almacenamiento de archivos
- **RFS-30** Abstracción `IAlmacenamientoArchivos` con implementación inicial en **volumen del VPS** y lista para cambiar a almacenamiento de objetos (S3 compatible / Azure Blob) por configuración, igual que la BD.
- **RFS-31** Archivos cifrados en reposo; se sirven solo mediante enlaces temporales firmados o a través de la API con autorización.

## 7. Proyecto 3 — Portal de administración (web)

### 7.1 CRM y contactos
- **RFP-01** Listado de clientes con búsqueda, filtros (estado de registro, etapa, origen, vendedor) y ficha del cliente.
- **RFP-02** Bandeja **“Clientes por completar”**: pre-registros y registros incompletos, con campos faltantes y origen (dispositivo, vendedor, fecha).
- **RFP-03** Edición y completado de datos; **fusión de duplicados**.
- **RFP-04** **Contactos** por cliente: varias personas (nombre, puesto, teléfono, correo, rol: decisor, técnico, pagos).
- **RFP-05** **Bitácora de interacciones**: llamadas, visitas, WhatsApp, correos, con fecha, responsable y próximo seguimiento.
- **RFP-06** Etapas comerciales: prospecto, negociación, cerrado, perdido.

### 7.2 Eventos de compra y seguimiento de entrega
- **RFP-10** Listado de eventos recibidos (tipo, cliente, vendedor, fecha, monto, estado) y detalle con PDF, constancia y evidencias.
- **RFP-11** **Seguimiento de entrega** de cada venta: estado (RECIBIDO → EN_PROCESO → LISTO_PARA_ENTREGA → ENTREGADO → CERRADO), responsable, fecha compromiso, tareas/checklist, notas y alertas de atraso.
- **RFP-12** El estado `ENTREGADO` se asigna automáticamente al recibir un evento `ENTREGA` firmado desde la App (o manualmente con justificación).
- **RFP-13** Verificador de integridad: subir un PDF o capturar folio y comparar hash.
- **RFP-14** Los eventos firmados son de **solo lectura**; cualquier cambio requiere un nuevo evento firmado en la App.

### 7.3 Pagos, suscripciones, catálogo y administración
- **RFP-20** Registro de pagos (pago único, anticipo, mensualidades) con comprobante.
- **RFP-21** Suscripciones: inicio, periodicidad, día de corte, estado (`ACTIVA`, `VENCIDA`, `SUSPENDIDA`, `CANCELADA`), alertas de vencimiento.
- **RFP-22** Catálogo de productos (tipo OFFLINE/ONLINE, plataformas, funcionalidades, precio, plantilla).
- **RFP-23** Plantillas de contrato y actas con **editor de contenido** y **versionado** (ver §7.4).
- **RFP-24** Usuarios y roles; perfil de la empresa vendedora (razón social, RFC, domicilio, logo).
- **RFP-25** Tablero: clientes por completar, eventos recibidos hoy, entregas atrasadas, suscripciones por vencer, pagos atrasados, **dispositivos con eventos sin enviar** (según última conexión reportada).
- **RFP-26** **Configuración de infraestructura** (solo superadmin): conexión a BD (§6.4), almacenamiento de archivos, respaldos (última ejecución, descarga).

### 7.4 Editor de contratos y plantillas

El administrador edita desde el Portal el **texto** de cada plantilla: contrato A, contrato B, acta de entrega, convenio modificatorio y convenio de terminación. El sistema sustituye los datos del comprador y del vendedor y agrega **siempre** al final la sección de nombres y firmas.

- **RFP-30** **Editor de texto enriquecido** (flutter_quill) limitado a lo que el PDF sabe dibujar: títulos (niveles 1–3), párrafo, negrita, cursiva, subrayado, listas numeradas y con viñetas, alineación y salto de página.
- **RFP-31** **Variables** insertadas desde un panel lateral (no se escriben a mano), mostradas en el editor como etiquetas de color, p. ej. `{{comprador.nombre}}`:

  | Grupo | Variables |
  |---|---|
  | Comprador | `comprador.nombre`, `comprador.tipoPersona`, `comprador.rfc`, `comprador.domicilio`, `comprador.representante`, `comprador.identificacion` (tipo y número), `comprador.telefono`, `comprador.email` |
  | Vendedor | `vendedor.nombre` (razón social), `vendedor.representante` (usuario que firma), `vendedor.rfc`, `vendedor.domicilio` |
  | Operación | `evento.folio`, `evento.fecha`, `evento.lugar`, `producto.nombre`, `producto.plataformas`, `precio.total`, `precio.anticipo`, `precio.periodicidad`, `precio.diaCorte`, `entrega.fechaCompromiso` |
  | Bloques | `anexo.alcance` (tabla de funcionalidades incluidas/excluidas), `anexo.pagos` (calendario de pagos), `contratoOrigen.folio`, `contratoOrigen.hash` (solo actas y convenios) |

- **RFP-32** **Bloques condicionales** para no duplicar plantillas: `{{#si producto.online}} … {{/si}}` y `{{#si comprador.personaMoral}} … {{/si}}`, insertados desde el panel.
- **RFP-33** **Sección de nombres y firmas fija**: no forma parte del texto editable y no se puede borrar. El generador de PDF la agrega **siempre al final del contenido**:
  1. Contenido de la plantilla (cláusulas y anexos).
  2. **Sección de firmas**: leyenda de cierre (“Leído que fue el presente documento y enteradas las partes de su contenido y alcance, lo firman en `{{evento.lugar}}` el `{{evento.fecha}}`”) y dos columnas:
     - **EL VENDEDOR**: imagen de firma, línea, `vendedor.nombre`, “Representada por `vendedor.representante`”.
     - **EL COMPRADOR**: imagen de firma, línea, `comprador.nombre` y, si es persona moral, “Representada por `comprador.representante`”.
     - El bloque nunca se parte entre páginas: si no cabe, pasa completo a la siguiente.
  3. **Hoja de constancia de firma electrónica** (evidencias, folio, hash).
- **RFP-34** Opción por plantilla **“Rúbrica en cada página”**: estampa en pequeño, en el margen de cada hoja, la firma del comprador **del mismo evento**. Sigue siendo la firma de un solo uso (RN-05).
- **RFP-35** **Vista previa en PDF** en el Portal con datos de ejemplo (persona física/moral, offline/online) usando **el mismo generador de PDF que la App** (paquete `contratos_pdf`, §10): lo que se ve en el Portal es exactamente lo que se firmará en campo.
- **RFP-36** **Validación al guardar**: variables desconocidas, bloques `si` sin cerrar o formato no soportado impiden publicar y se señala el error.
- **RFP-37** **Versionado**: cada “Publicar” crea una versión nueva, inmutable, con autor, fecha y nota de cambio. Las Apps reciben la versión publicada en su siguiente sincronización. Se puede duplicar una versión anterior como borrador. Un documento firmado conserva siempre la versión con que se firmó.
- **RFP-38** Estados: `BORRADOR` (solo Portal) → `PUBLICADA` (la reciben las Apps) → `RETIRADA` (no se ofrece para eventos nuevos).
- **RFP-39** Almacenamiento: JSON **Quill Delta** con las variables como texto `{{...}}`.

### 7.5 ¿Odoo?
No en v1. Tendría su propia base de datos y habría que sincronizar tres lugares; no resuelve el trabajo offline ni la firma con S Pen; los módulos útiles (suscripciones, firma, CFDI) son de pago por usuario; y agrega Python como tercer stack. Si después se necesita contabilidad o facturación, se integra como sistema **secundario** (la API le envía datos ya cerrados, en un solo sentido), o se usa un PAC con API para CFDI.

## 8. Requerimientos no funcionales

| ID | Requerimiento |
|---|---|
| RNF-01 | App: Android (Samsung con S Pen) como plataforma principal; iOS opcional. |
| RNF-02 | App offline-first: registro de clientes, eventos, PDF y firma funcionan sin red. |
| RNF-03 | HTTPS/TLS en todo; contraseñas con hash (ASP.NET Identity / Argon2); JWT con expiración; BD local cifrada. |
| RNF-04 | Protección de datos personales (LFPDPPP): aviso de privacidad aceptado por el comprador; identificaciones y firmas cifradas y con acceso auditado. |
| RNF-05 | Conservación de documentos y bitácoras mínimo **10 años**. |
| RNF-06 | Respaldos diarios de PostgreSQL (`pg_dump`) y de archivos, con copia **fuera del VPS**; prueba de restauración mensual. |
| RNF-07 | Generación de PDF < 5 s en tableta de gama media. |
| RNF-08 | Portal responsivo para escritorio y tableta; idioma español. |
| RNF-09 | Dispositivos de prueba: teléfono Samsung con S Pen y Galaxy Tab con S Pen. |
| **RNF-20** | **Compatibilidad con PostgreSQL administrado**: sin extensiones exóticas ni funciones que requieran superusuario; migraciones con EF Core (bundle) ejecutables contra cualquier destino; soporte de SSL obligatorio; sin dependencias del sistema de archivos del contenedor de BD. |
| RNF-21 | Configuración externalizada: nada de cadenas de conexión ni secretos dentro de la imagen del contenedor. |
| RNF-22 | Logs estructurados (Serilog) y endpoint `/health` que incluya estado de la BD y del almacenamiento. |

## 9. Infraestructura (v1, OVHcloud)

```
                        Internet (HTTPS)
                              │
┌──────────────────────────── VPS ────────────────────────────┐
│  Docker Compose                                             │
│  ┌──────────────┐                                           │
│  │ Proxy inverso│ Caddy o Nginx (TLS automático)            │
│  └──┬────────┬──┘                                           │
│     │        │                                              │
│     ▼        ▼                                              │
│ ┌────────┐ ┌───────────────────┐    ┌───────────────────┐   │
│ │ Portal │ │ API .NET 10       │───►│ PostgreSQL        │   │
│ │ (web   │ │ vol: config cifr. │    │ vol: datos        │   │
│ │ estát.)│ │ vol: archivos     │    │ (no expuesto a    │   │
│ └────────┘ └───────────────────┘    │  internet)        │   │
│                                     └───────────────────┘   │
│  ┌───────────────┐                                          │
│  │ Respaldos     │ pg_dump + archivos → OVH Object Storage   │
│  └───────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
        ▲
        │ HTTPS (envío de eventos, sync de clientes/catálogo)
   App de campo (Samsung + S Pen, SQLite)
```

**Futuro:** PostgreSQL pasa a **OVH Managed PostgreSQL** (se cambia desde el Portal, §6.4) y los archivos a **OVH Object Storage** (§6.5). El contenedor de PostgreSQL se retira.

Las imágenes de contenedor las publica el CI de cada repositorio en GitHub Container Registry; en el VPS solo se ejecuta `docker compose pull && docker compose up -d`.

## 10. Repositorios

| Repositorio | Contenido | Artefacto |
|---|---|---|
| `App-contratos` (existente) | Esta documentación, [contrato-api.md](contrato-api.md), [planes/](planes/) | — |
| `contratos-api` | API .NET 10 (VSA) + `deploy/` (docker-compose, Caddy, respaldos, migración de BD) | Imagen `contratos-api` |
| `contratos-app` | App de campo Flutter (Android/iOS) | APK / AAB |
| `contratos-portal` | Portal Flutter Web | Imagen `contratos-portal` (Nginx con el build web) |
| `contratos-dart` | Paquetes Dart compartidos: `contratos_modelos` (DTOs + cliente HTTP) y `contratos_pdf` (motor de plantillas + generador de PDF) | Dependencia git fijada a un tag |

**¿Por qué `contratos-dart`?** App y Portal deben producir **exactamente el mismo PDF** con la misma plantilla (el Portal para la vista previa, la App para firmar). Con un solo generador compartido no hay diferencias entre lo que se aprueba en oficina y lo que se firma en campo.

Cada repositorio de código lleva un `CLAUDE.md` que apunta a esta documentación y a su plan de trabajo.

### Estructura de `contratos-api`

```
contratos-api/
  src/Contratos.Api/
    Features/
      Auth/  Usuarios/  Clientes/  Identificaciones/  Contactos/  Interacciones/
      Catalogo/  Plantillas/  Eventos/  Seguimiento/  Pagos/  Suscripciones/
      Sync/  Tablero/  Infraestructura/
    Common/        Auth, errores (ProblemDetails), auditoría, almacenamiento, conexión dinámica de BD
    Data/          ContratosDbContext (EF Core + Npgsql), migraciones
  tests/Contratos.Api.Tests/   (xUnit + Testcontainers)
  deploy/
    docker-compose.yml  Caddyfile  .env.example
    backups/       pg_dump programado → OVH Object Storage
    db-migracion/  pg_dump/pg_restore guiado
  Dockerfile
  CLAUDE.md
```

## 11. Modelo de datos (PostgreSQL)

- `usuario` (id, email, hash_password, rol, activo, 2fa)
- `empresa_vendedora` (id, razon_social, rfc, domicilio, logo_archivo_id)
- `firma_vendedor` (id, usuario_id, imagen_archivo_id, trazo_archivo_id, vigente)
- `cliente` (id, tipo_persona, nombre, rfc, curp, domicilio, telefono, email, estado_registro [`INCOMPLETO`/`COMPLETO`], etapa_crm, origen [`APP`/`PORTAL`], dispositivo_origen, vendedor_id)
- `identificacion` (id, cliente_id, titular_nombre, tipo [`INE`/`PASAPORTE`/`CEDULA`], numero, vigencia, anverso_archivo_id, reverso_archivo_id, poder_archivo_id)
- `contacto` (id, cliente_id, nombre, puesto, telefono, email, rol)
- `interaccion` (id, cliente_id, contacto_id, tipo, fecha, usuario_id, nota, proximo_seguimiento)
- `producto` (id, nombre, tipo [`OFFLINE`/`ONLINE`], plataformas, precio, periodicidad, plantilla_id)
- `producto_funcionalidad` (id, producto_id, descripcion)
- `plantilla` (id, nombre, tipo_documento [`CONTRATO_OFFLINE`/`CONTRATO_ONLINE`/`ACTA_ENTREGA`/`CONVENIO_MODIFICATORIO`/`CONVENIO_TERMINACION`], estado [`BORRADOR`/`PUBLICADA`/`RETIRADA`], borrador_delta [jsonb])
- `plantilla_version` (id, plantilla_id, numero, contenido_delta [jsonb], rubrica_por_pagina, nota_cambio, autor_id, publicada_en) — inmutable una vez publicada
- `evento` (id [UUID de la App], folio, tipo [`VENTA`/`ENTREGA`/`MODIFICACION`/`CANCELACION`], evento_origen_id, cliente_id, vendedor_id, plantilla_version_id, monto, modalidad_pago, fecha_firma, fecha_recepcion_servidor, pdf_archivo_id, pdf_hash, dispositivo)
- `evento_producto` (id, evento_id, producto_id, precio, alcance_json)
- `firma_evidencia` (id, evento_id, documento_hash, firmante [`VENDEDOR`/`COMPRADOR`], nombre, imagen_archivo_id, trazo_archivo_id, tipo_entrada [`S_PEN`/`DEDO`], fecha_dispositivo, lat, lng, dispositivo, identificacion_id, selfie_archivo_id) — **una por firma y por documento; nunca se reutiliza**
- `seguimiento` (id, evento_venta_id, estado, responsable_id, fecha_compromiso, fecha_entrega)
- `seguimiento_tarea` (id, seguimiento_id, descripcion, completada, fecha)
- `suscripcion` (id, evento_venta_id, inicio, periodicidad, dia_corte, monto, estado)
- `pago` (id, evento_venta_id, suscripcion_id, fecha, monto, metodo, comprobante_archivo_id)
- `archivo` (id, tipo, ruta, hash, tamano, cifrado)
- `auditoria` (id, usuario_id, accion, entidad, entidad_id, fecha, detalle)
- Columnas comunes: `created_at`, `updated_at`, `deleted_at`, `version`.

**SQLite (App)** contiene solo lo necesario en campo: `cliente`, `identificacion`, `producto`, `producto_funcionalidad`, `plantilla`, `evento`, `evento_producto`, `firma_evidencia`, `archivo_local`, `firma_vendedor`, más control: `sync_estado` (última sincronización) y en `evento`: `estado_envio`, `intentos`, `ultimo_error`, `fecha_ultimo_intento`.

## 12. Fuera de alcance (v1)

- Firma electrónica avanzada con e.firma del SAT.
- Facturación CFDI.
- Pasarela de cobro recurrente automático.
- Portal de autoservicio para el cliente final.
- Integración con Odoo.

## 13. Pendientes

- Revisión de plantillas por un **abogado**. Aunque los clientes son negocios, la Ley Federal de Protección al Consumidor da ciertas protecciones a micro y pequeños negocios; las cláusulas deben ser claras y equilibradas.
- Contratar VPS en OVHcloud, dominio y DNS (p. ej. `api.tudominio.mx`, `portal.tudominio.mx`).
- Crear los repositorios `contratos-api`, `contratos-app`, `contratos-portal`, `contratos-dart`.
