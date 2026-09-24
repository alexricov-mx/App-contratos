# Plan de trabajo — `contratos-dart`

> Paquetes Dart compartidos por App y Portal. Leer antes: [contrato-api.md](../contrato-api.md), [requerimientos.md §7.4](../requerimientos.md), [contrato-base.md](../contrato-base.md).

## Stack
Dart 3 / Flutter estable · monorepo con `melos` · `freezed` + `json_serializable` · `dio` · `pdf` · `crypto` · pruebas con `test` y golden tests de PDF.

## Estructura
```
contratos-dart/
  packages/
    contratos_modelos/   DTOs, enums, cliente HTTP, ApiMock
    contratos_pdf/       motor de plantillas (Delta + variables) y generador de PDF
  melos.yaml
  CLAUDE.md
```
Se consume desde App y Portal como dependencia git con `path` y `ref: vX.Y.Z`.

---

## D1 · `contratos_modelos` (día 1, mañana)

| # | Tarea | Aceptación |
|---|---|---|
| D1.1 | Crear monorepo, `melos.yaml`, `CLAUDE.md`, CI (analyze + test) | `melos run test` en verde |
| D1.2 | DTOs de contrato-api.md §3–§8 con `freezed`/`json_serializable`: Cliente, Domicilio, Identificacion, Producto, Funcionalidad, PlantillaVersion, Evento (+ Firma, Dispositivo, ProductoEvento), EventoRecibido, TrazoFirma, Contacto, Interaccion, Seguimiento, Tarea, Pago, Suscripcion, Usuario, Tablero, ConexionBd, Operacion, ProblemDetails, Pagina<T> | Round-trip JSON de cada ejemplo de contrato-api.md en pruebas |
| D1.3 | Enums en MAYÚSCULAS con valor desconocido tolerado (`UNKNOWN`) | Prueba con valor no previsto no revienta |
| D1.4 | Regla `estadoRegistro` (función pura `calcularEstadoRegistro(Cliente, hoy)` + `camposFaltantes`) idéntica a contrato-api.md §3 | Pruebas: física completa, moral sin poder, INE vencida |
| D1.5 | Cliente HTTP `ContratosApi` (dio): interceptor de token + refresh, mapeo de ProblemDetails a `ApiException(code, status)`, multipart para `/sync/clientes`, `/eventos`, pagos | Pruebas con servidor falso (`http_mock_adapter`) |
| D1.6 | `ApiMock`: implementación en memoria de `ContratosApi` con datos semilla (3 clientes, 2 productos, 2 plantillas) | App y Portal pueden arrancar sin API |
| D1.6b | Documentar en README el uso de `pubspec_overrides.yaml` para probar cambios sin tag ([entorno-local.md §5](../entorno-local.md)) | — |
| D1.7 | Tag `v0.1.0` | — |

## D2 · `contratos_pdf` (día 1, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| D2.1 | Catálogo de variables y bloques (constante única, igual a `GET /plantillas/variables`) | — |
| D2.2 | `ValidadorPlantilla.validar(delta)` → lista de errores (variable desconocida, `si` sin cerrar, atributo no permitido) según contrato-api.md §7 | Pruebas por cada tipo de error con posición |
| D2.3 | `ResolvedorVariables`: sustituye `{{…}}` con `DatosDocumento` (comprador, vendedor, evento, productos, precios, contrato origen) y evalúa bloques `{{#si}}` | Pruebas persona física/moral, offline/online |
| D2.4 | Bloques especiales: `anexo.alcance` (tabla incluidas/excluidas), `anexo.pagos` (tabla de calendario) | Golden test |
| D2.5 | `GeneradorPdf.generar(plantilla, datos, firmas?) → Uint8List`: Delta → widgets `pdf` (títulos 1–3, negrita, cursiva, subrayado, listas, alineación, salto de página), encabezado con logo, pie con folio y “Página X de Y” | Golden tests |
| D2.6 | **Sección de nombres y firmas** (RFP-33) agregada siempre al final: leyenda de cierre con lugar y fecha, dos columnas VENDEDOR / COMPRADOR con imagen, línea, nombre y “Representada por…” cuando aplica; bloque indivisible (`pdf` `Inseparable`/`KeepTogether`) | Golden tests: con firmas, sin firmas (vista previa muestra líneas vacías), bloque que salta de página |
| D2.7 | **Rúbrica por página** (RFP-34) opcional | Golden test |
| D2.8 | **Hoja de constancia** (evidencias, folio, identificación, tipo de entrada, geolocalización, hash del contenido) | Golden test |
| D2.9 | Cálculo del hash: SHA-256 del PDF final (`hashPdf(bytes)`) | Prueba determinista |
| D2.10 | Fuentes embebidas (p. ej. Noto Sans) con acentos y ñ | Golden test con “Señor Peña” |
| D2.11 | Plantillas semilla en Delta: contrato offline, contrato online, acta de entrega, convenio modificatorio, convenio de terminación, redactadas desde [contrato-base.md](../contrato-base.md) | Pasan el validador |
| D2.12 | Tag `v0.2.0` | — |

**Importante:** el PDF debe ser **determinista** para los mismos datos (fecha de creación fija desde `DatosDocumento`, sin metadatos aleatorios) para que el hash sea reproducible en pruebas.
