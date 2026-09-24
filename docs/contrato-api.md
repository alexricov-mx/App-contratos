# Contrato de la API v1

> Versión 0.1 · 2026-09-24 · Fuente de verdad compartida por `contratos-api`, `contratos-app`, `contratos-portal` y `contratos-dart`.
>
> Regla: si una implementación necesita algo que no está aquí, **primero se actualiza este documento** y después el código. El OpenAPI que publica la API debe coincidir con este documento.

## 1. Convenciones

| Tema | Regla |
|---|---|
| Base URL | `https://api.<dominio>/api/v1` |
| Formato | JSON UTF-8, propiedades en **camelCase** |
| IDs | **UUID v4** en texto. Los genera **quien crea** el registro (la App para clientes y eventos creados en campo) |
| Fechas | ISO 8601 en **UTC** con `Z` (`2026-09-24T18:30:00Z`). Solo fecha: `2026-09-24` |
| Dinero | `number` con 2 decimales, moneda en campo aparte (`"MXN"`) |
| Enums | Texto en MAYÚSCULAS (`"OFFLINE"`, `"VENTA"`) |
| Auth | `Authorization: Bearer <accessToken>` en todo excepto `/auth/login` y `/auth/refresh` |
| Errores | `application/problem+json` (RFC 9457) con `type`, `title`, `status`, `detail`, `errors` (validación por campo) y `code` (código de negocio) |
| Paginación | `?pagina=1&tamano=50` → `{ "items": [], "pagina": 1, "tamano": 50, "total": 123 }` |
| Concurrencia | Entidades editables llevan `version` (entero). En `PUT` se envía; si no coincide → `409` con `code: "VERSION_CONFLICTO"` |
| Mantenimiento | `503` con `code: "MANTENIMIENTO"` y cabecera `Retry-After`. La App lo trata como error temporal |
| Idempotencia | `POST /eventos` es idempotente por `id`. Además se acepta cabecera `Idempotency-Key` |

### Códigos de negocio (`code`)
`CREDENCIALES_INVALIDAS`, `TOKEN_EXPIRADO`, `SIN_PERMISO`, `VALIDACION`, `VERSION_CONFLICTO`, `CLIENTE_INCOMPLETO`, `EVENTO_HASH_CONFLICTO`, `EVENTO_HASH_INVALIDO`, `ARCHIVO_FALTANTE`, `PLANTILLA_INVALIDA`, `MANTENIMIENTO`, `BD_CONEXION_FALLIDA`, `BD_ESQUEMA_INCOMPATIBLE`.

### Roles
`VENDEDOR` (App), `OFICINA`, `ADMIN`, `SUPERADMIN` (Portal). En cada endpoint se indica el rol mínimo; `ADMIN` incluye `OFICINA`; `SUPERADMIN` incluye todo.

---

## 2. Autenticación

### `POST /auth/login`
```json
// Request
{ "email": "ana@empresa.mx", "password": "********", "dispositivo": { "id": "uuid", "nombre": "Galaxy Tab S9", "tipo": "APP" } }
// 200
{ "accessToken": "jwt", "expiraEn": 900, "refreshToken": "opaco",
  "usuario": { "id": "uuid", "nombre": "Ana López", "email": "ana@empresa.mx", "rol": "VENDEDOR" } }
```
`tipo`: `APP` | `PORTAL`. Access token 15 min; refresh token 30 días (App) / 12 h (Portal), rotativo.

### `POST /auth/refresh`
`{ "refreshToken": "opaco" }` → mismo cuerpo que login.

### `POST /auth/logout`
Revoca el refresh token actual. `204`.

### `POST /auth/2fa/verificar` (SUPERADMIN)
`{ "codigo": "123456" }` → `{ "tokenElevado": "jwt", "expiraEn": 300 }`. Requerido en cabecera `X-Token-Elevado` para `/infraestructura/*`.

---

## 3. DTOs compartidos

### Cliente
```json
{
  "id": "uuid",
  "tipoPersona": "FISICA",               // FISICA | MORAL
  "nombre": "Abarrotes Don Pepe",
  "rfc": "PEPJ800101ABC",                // opcional en pre-registro
  "curp": null,
  "domicilio": { "calle": "", "numExt": "", "numInt": "", "colonia": "", "cp": "", "municipio": "", "estado": "" },
  "telefono": "5512345678",
  "email": "pepe@correo.mx",
  "representante": null,                 // obligatorio si MORAL
  "estadoRegistro": "INCOMPLETO",        // INCOMPLETO | COMPLETO (lo calcula la API; la App lo calcula igual localmente)
  "etapaCrm": "PROSPECTO",               // PROSPECTO | NEGOCIACION | CERRADO | PERDIDO
  "origen": "APP",                       // APP | PORTAL
  "vendedorId": "uuid",
  "identificacion": {                    // null en pre-registro
    "id": "uuid",
    "titularNombre": "José Pérez",
    "tipo": "INE",                       // INE | PASAPORTE | CEDULA
    "numero": "1234567890123",
    "vigencia": "2031-12-31",
    "anversoArchivoId": "uuid",
    "reversoArchivoId": "uuid",
    "poderArchivoId": null
  },
  "camposFaltantes": ["rfc", "identificacion"],   // solo lectura
  "version": 3,
  "creadoEn": "…Z", "actualizadoEn": "…Z", "eliminadoEn": null
}
```
**Regla `COMPLETO`:** nombre, tipoPersona, rfc, domicilio (calle, cp, municipio, estado), teléfono o email, identificación con anverso+reverso y `vigencia >= hoy`; si `MORAL`, además `representante` y `poderArchivoId`.

### Producto
```json
{ "id": "uuid", "nombre": "Punto de venta offline", "tipo": "OFFLINE",   // OFFLINE | ONLINE
  "plataformas": ["ANDROID", "WEB"],                                     // ANDROID | IOS | WEB | WINDOWS
  "precio": 15000.00, "moneda": "MXN",
  "periodicidad": null,                                                  // null si OFFLINE; MENSUAL | ANUAL si ONLINE
  "funcionalidades": [ { "id": "uuid", "descripcion": "Registro de ventas", "orden": 1 } ],
  "plantillaId": "uuid", "activo": true, "version": 1, "actualizadoEn": "…Z" }
```
Regla: `OFFLINE` ⇒ `periodicidad = null`; `ONLINE` ⇒ `periodicidad` obligatoria.

### PlantillaVersion (lo que recibe la App)
```json
{ "id": "uuid", "plantillaId": "uuid", "nombre": "Contrato SaaS",
  "tipoDocumento": "CONTRATO_ONLINE",   // CONTRATO_OFFLINE | CONTRATO_ONLINE | ACTA_ENTREGA | CONVENIO_MODIFICATORIO | CONVENIO_TERMINACION
  "numero": 4, "contenidoDelta": { "ops": [ … ] }, "rubricaPorPagina": true,
  "publicadaEn": "…Z" }
```

### Archivo
Los archivos binarios viajan en `multipart/form-data`; en JSON solo se referencian por `id`.
`GET /archivos/{id}` → binario (autorizado y auditado). `GET /archivos/{id}/url` → `{ "url": "…", "expiraEn": 300 }`.

---

## 4. Sincronización (App)

### `POST /sync/clientes` (VENDEDOR)
`multipart/form-data`:
- parte `datos` (JSON): `{ "clientes": [Cliente, …] }` (con `version` que tenía el dispositivo)
- partes binarias con nombre = `archivoId` para cada imagen de identificación/poder nueva

```json
// 200
{ "resultados": [
    { "id": "uuid", "estado": "APLICADO",   "cliente": { …Cliente final… } },
    { "id": "uuid", "estado": "FUSIONADO",  "cliente": { … }, "camposEnConflicto": ["telefono"] },
    { "id": "uuid", "estado": "RECHAZADO",  "error": { "code": "VALIDACION", "detail": "…" } } ] }
```
Resolución: por campo, gana el `actualizadoEn` más reciente; el servidor registra el conflicto en auditoría.

### `GET /sync/clientes?desde=2026-09-24T00:00:00Z&pagina=1&tamano=200` (VENDEDOR)
Clientes creados/modificados/eliminados después de `desde` (incluye `eliminadoEn`). Respuesta paginada + `"marca": "…Z"` que el dispositivo guarda como siguiente `desde`.

### `GET /sync/catalogo?desde=…` (VENDEDOR)
```json
{ "productos": [Producto], "plantillas": [PlantillaVersion],   // solo versiones PUBLICADAS vigentes
  "empresa": { "razonSocial": "", "rfc": "", "domicilio": {…}, "logoArchivoId": "uuid" },
  "firmaVendedor": { "imagenArchivoId": "uuid", "trazoArchivoId": "uuid", "actualizadaEn": "…Z" },
  "marca": "…Z" }
```

### `PUT /sync/firma-vendedor` (VENDEDOR)
`multipart`: `imagen` (PNG) + `trazo` (JSON de trazo, ver §5.2). `200` → `{ "imagenArchivoId", "trazoArchivoId" }`.

---

## 5. Eventos

### 5.1 `POST /eventos` (VENDEDOR) — crea el evento completo

`multipart/form-data`, límite 25 MB. Partes:

| Parte | Tipo | Obligatoria |
|---|---|---|
| `evento` | JSON (abajo) | Sí |
| `pdf` | `application/pdf` | Sí |
| `firmaCompradorImagen` | `image/png` | Sí |
| `firmaCompradorTrazo` | `application/json` | Sí |
| `firmaVendedorImagen` | `image/png` | Sí |
| `selfie` | `image/jpeg` | No |
| `<archivoId>` | imágenes de identificación del cliente si aún no se subieron | Condicional |

```json
{
  "id": "uuid",                               // generado en la App; clave de idempotencia
  "folio": "VTA-ANA-20260924-0007",           // {TIPO}-{iniciales vendedor}-{yyyyMMdd}-{consecutivo del dispositivo}
  "tipo": "VENTA",                            // VENTA | ENTREGA | MODIFICACION | CANCELACION
  "eventoOrigenId": null,                     // obligatorio si tipo != VENTA
  "cliente": { …Cliente… },                   // se hace alta/actualización en la misma transacción
  "plantillaVersionId": "uuid",
  "productos": [ { "productoId": "uuid", "precio": 15000.00,
                   "alcance": [ { "descripcion": "Registro de ventas", "incluido": true } ] } ],
  "montoTotal": 15000.00, "moneda": "MXN",
  "modalidadPago": "PAGO_UNICO",              // PAGO_UNICO | SUSCRIPCION
  "anticipo": 5000.00,
  "periodicidad": null, "diaCorte": null,
  "fechaCompromisoEntrega": "2026-10-15",
  "notas": "",
  "lugar": "Toluca, Estado de México",
  "pdfSha256": "hex64",
  "firmas": [
    { "firmante": "COMPRADOR", "nombre": "José Pérez", "tipoEntrada": "S_PEN",
      "fechaDispositivo": "…Z", "lat": 19.28, "lng": -99.65, "precisionMetros": 12,
      "identificacionId": "uuid", "aceptaciones": ["ALCANCE","RESPONSABILIDAD","PAGO","PRIVACIDAD","USO_FIRMA"] },
    { "firmante": "VENDEDOR", "nombre": "Ana López", "tipoEntrada": "S_PEN",
      "fechaDispositivo": "…Z", "confirmacion": "PIN" }       // PIN | BIOMETRIA
  ],
  "dispositivo": { "id": "uuid", "modelo": "SM-X710", "so": "Android 15", "versionApp": "1.0.0+12" }
}
```

Respuestas:

| Código | Caso | Cuerpo |
|---|---|---|
| `201` | Creado | `EventoRecibido` |
| `200` | Ya existía con el mismo hash (reenvío) | `EventoRecibido` |
| `409` `EVENTO_HASH_CONFLICTO` | Ya existía con **otro** hash | ProblemDetails — la App **no** reintenta sola |
| `422` `EVENTO_HASH_INVALIDO` | El PDF recibido no coincide con `pdfSha256` | ProblemDetails — la App reintenta (posible corrupción en tránsito) |
| `422` `CLIENTE_INCOMPLETO` / `VALIDACION` / `ARCHIVO_FALTANTE` | Datos inválidos | ProblemDetails — no reintentar sin corregir |
| `503` `MANTENIMIENTO` | BD en cambio | Reintentar después de `Retry-After` |

```json
// EventoRecibido
{ "id": "uuid", "folio": "…", "pdfSha256": "hex64", "recibidoEn": "…Z", "estadoSeguimiento": "RECIBIDO" }
```

Efectos en servidor: `VENTA` → crea `seguimiento` (`RECIBIDO`) y, si `SUSCRIPCION`, `suscripcion` (`PENDIENTE_INICIO`). `ENTREGA` → seguimiento `ENTREGADO` y suscripción `ACTIVA` (inicio = fecha de firma). `CANCELACION` → seguimiento `CANCELADO`, suscripción `CANCELADA`.

### 5.2 Formato del trazo de firma
```json
{ "ancho": 1600, "alto": 700, "trazos": [ [ { "x": 12.5, "y": 30.1, "p": 0.62, "t": 0 }, … ], … ] }
```
`p` = presión 0–1; `t` = ms desde el primer punto.

### 5.3 `GET /eventos/{id}/estado` (VENDEDOR)
`200` → `EventoRecibido` · `404` si no existe (la App lo reenvía).

---

## 6. Portal

Todos paginados y con filtros por query string. Rol mínimo `OFICINA` salvo que se indique.

| Recurso | Endpoints |
|---|---|
| Tablero | `GET /tablero` → `{ clientesIncompletos, eventosHoy, entregasAtrasadas, suscripcionesPorVencer, pagosAtrasados, seguimientosVencidos, dispositivosConPendientes: [{ usuario, dispositivo, ultimaConexion }] }` |
| Clientes | `GET /clientes?q=&estadoRegistro=&etapaCrm=&vendedorId=` · `GET /clientes/{id}` · `POST /clientes` · `PUT /clientes/{id}` · `DELETE /clientes/{id}` (lógico) · `POST /clientes/{id}/fusionar` `{ "duplicadoId": "uuid" }` · `POST /clientes/{id}/identificacion` (multipart) |
| Contactos | `GET/POST /clientes/{id}/contactos` · `PUT/DELETE /contactos/{id}` — `{ id, nombre, puesto, telefono, email, rol: DECISOR|TECNICO|PAGOS|OTRO }` |
| Interacciones | `GET/POST /clientes/{id}/interacciones` · `PUT /interacciones/{id}` — `{ id, contactoId, tipo: LLAMADA|VISITA|WHATSAPP|CORREO, fecha, nota, proximoSeguimiento, usuarioId }` |
| Eventos | `GET /eventos?tipo=&clienteId=&vendedorId=&desde=&hasta=` · `GET /eventos/{id}` (incluye firmas, evidencias, eventos ligados, URLs de archivos) |
| Seguimiento | `GET /seguimientos?estado=&atrasados=true` · `GET /seguimientos/{id}` · `PUT /seguimientos/{id}` `{ estado, responsableId, fechaCompromiso, version }` · `POST /seguimientos/{id}/tareas` · `PUT /tareas/{id}` · `POST /seguimientos/{id}/entregado-manual` `{ justificacion }` (ADMIN) |
| Pagos | `GET /pagos?eventoId=` · `POST /pagos` (multipart: `datos` + `comprobante`) `{ eventoVentaId, suscripcionId, fecha, monto, metodo: EFECTIVO|TRANSFERENCIA|TARJETA|OTRO, referencia }` |
| Suscripciones | `GET /suscripciones?estado=&vencenEnDias=` · `POST /suscripciones/{id}/suspender` · `/reactivar` · `/cancelar` `{ motivo }` |
| Productos (ADMIN) | `GET/POST /productos` · `PUT/DELETE /productos/{id}` |
| Plantillas (ADMIN) | ver §7 |
| Usuarios (ADMIN) | `GET/POST /usuarios` · `PUT /usuarios/{id}` · `POST /usuarios/{id}/restablecer-password` · `POST /usuarios/{id}/revocar-dispositivos` |
| Empresa (ADMIN) | `GET/PUT /empresa` · `PUT /empresa/logo` (multipart) |
| Integridad | `POST /integridad/verificar` (multipart `pdf`) o `GET /integridad/folio/{folio}` → `{ coincide, evento: { id, folio, recibidoEn, pdfSha256 } }` |
| Auditoría (ADMIN) | `GET /auditoria?entidad=&entidadId=&usuarioId=&desde=&hasta=` |

---

## 7. Plantillas (ADMIN)

| Método | Ruta | Uso |
|---|---|---|
| `GET` | `/plantillas` | Lista con estado y última versión publicada |
| `POST` | `/plantillas` | `{ nombre, tipoDocumento }` → crea en `BORRADOR` |
| `GET` | `/plantillas/{id}` | Incluye `borradorDelta`, `rubricaPorPagina`, `version` |
| `PUT` | `/plantillas/{id}/borrador` | `{ borradorDelta, rubricaPorPagina, version }` — guarda sin publicar |
| `POST` | `/plantillas/{id}/validar` | Valida el borrador → `{ valido, errores: [{ posicion, code, detail }] }` |
| `POST` | `/plantillas/{id}/publicar` | `{ notaCambio }` → valida y crea `PlantillaVersion` inmutable |
| `GET` | `/plantillas/{id}/versiones` | Historial |
| `POST` | `/plantillas/{id}/versiones/{numero}/duplicar` | Copia esa versión al borrador |
| `POST` | `/plantillas/{id}/retirar` | Estado `RETIRADA` |
| `GET` | `/plantillas/variables` | Catálogo de variables y bloques permitidos (fuente para el panel del editor y para la validación) |

**Validación (misma lógica en `contratos_pdf` y en la API):**
- Variables solo del catálogo de `/plantillas/variables`.
- Bloques `{{#si X}}…{{/si}}` balanceados; `X` ∈ `producto.online`, `producto.offline`, `comprador.personaMoral`, `comprador.personaFisica`.
- Atributos Quill permitidos: `header` (1–3), `bold`, `italic`, `underline`, `list` (`ordered`/`bullet`), `align`, y el embed `salto-pagina`. Cualquier otro → `PLANTILLA_INVALIDA`.
- La plantilla **no** incluye la sección de firmas; la agrega el generador (requerimientos RFP-33).

---

## 8. Infraestructura (SUPERADMIN + `X-Token-Elevado`)

| Método | Ruta | Cuerpo / Respuesta |
|---|---|---|
| `GET` | `/infraestructura/bd` | `{ host, puerto, baseDatos, usuario, modoSsl, versionPostgres, versionEsquema, mantenimiento, anterior: { host, … } }` (nunca devuelve contraseña) |
| `POST` | `/infraestructura/bd/probar` | `ConexionBd` → `{ ok, versionPostgres, esquema: VACIO|COMPATIBLE|DESACTUALIZADO|INCOMPATIBLE, detalle }` |
| `POST` | `/infraestructura/bd/cambiar` | `{ conexion: ConexionBd, modo: SOLO_CAMBIAR|MIGRAR_Y_CAMBIAR }` → `202 { operacionId }` |
| `GET` | `/infraestructura/operaciones/{id}` | `{ estado: EN_CURSO|OK|FALLIDA, paso, progreso: 0-100, log: [] }` |
| `POST` | `/infraestructura/bd/revertir` | `202 { operacionId }` |
| `GET` | `/infraestructura/respaldos` | `[{ fecha, tamano, destino, ok }]` |
| `POST` | `/infraestructura/respaldos` | Lanza respaldo inmediato → `202 { operacionId }` |

```json
// ConexionBd
{ "host": "", "puerto": 5432, "baseDatos": "contratos", "usuario": "", "password": "",
  "modoSsl": "REQUIRE", "certificadoCa": null, "poolMax": 50, "timeoutSegundos": 15 }   // DISABLE | REQUIRE | VERIFY_FULL
```

## 9. Salud
`GET /health` (público) → `{ "estado": "OK", "bd": "OK", "almacenamiento": "OK", "mantenimiento": false, "version": "1.0.0" }`
