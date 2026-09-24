# Plan de trabajo — `contratos-portal`

> Leer antes: [requerimientos.md §7](../requerimientos.md) (en especial §7.4 editor), [casos-de-uso.md](../casos-de-uso.md) (CU-P*), [contrato-api.md](../contrato-api.md), [plan-dart.md](plan-dart.md).

## Stack
Flutter Web (estable) · `contratos_modelos` + `contratos_pdf` (git) · riverpod · go_router (URLs legibles) · `flutter_quill` (editor) · `printing` (vista previa PDF) · `data_table_2` o similar para tablas · imagen Docker con Nginx sirviendo `build/web`.

## Arquitectura
```
lib/
  core/        api (ContratosApi / ApiMock), auth, layout (menú lateral), tema, widgets comunes
  features/
    tablero/  clientes/  contactos/  eventos/  seguimiento/  pagos/  suscripciones/
    productos/  plantillas/  usuarios/  empresa/  integridad/  infraestructura/
```
El token se guarda en memoria + refresh en `sessionStorage`; nada sensible en `localStorage`.

---

## W0 · Base, login y CRM (día 1, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| W0.1 | Proyecto, `CLAUDE.md`, `Dockerfile` (build web + Nginx con `try_files` para rutas), CI (analyze + test + imagen a GHCR) | Imagen corre local |
| W0.2 | Login, refresh, cierre de sesión, guardias de ruta por rol; menú lateral según rol | — |
| W0.3 | Tablero (CU-P01) con tarjetas y enlaces a listas filtradas | — |
| W0.4 | Clientes: lista con búsqueda/filtros/paginación; ficha con pestañas (Datos, Identificación, Contactos, Interacciones, Eventos) | — |
| W0.5 | Bandeja **Clientes por completar** (CU-P02) con `camposFaltantes`; edición; subida de identificación; **fusión de duplicados** | — |
| W0.6 | Contactos e interacciones (CU-P03) con próximo seguimiento | Funciona contra `ApiMock` y API local → **H2** |

## W1 · Editor de contratos y plantillas (día 2, mañana)

Referencia exacta: requerimientos RFP-30 a RFP-39 y contrato-api.md §7.

| # | Tarea | Aceptación |
|---|---|---|
| W1.1 | Lista de plantillas (tipo, estado, versión publicada, fecha) y creación | — |
| W1.2 | **Editor** `flutter_quill` con barra limitada a: títulos 1–3, negrita, cursiva, subrayado, listas, alineación, salto de página (embed propio) | No es posible aplicar formatos fuera de la lista (tampoco pegando HTML: se limpia al pegar) |
| W1.3 | **Panel de variables** agrupado (Comprador, Vendedor, Operación, Bloques) desde `GET /plantillas/variables`; insertar en el cursor; las variables se muestran como **etiquetas de color** no editables carácter a carácter (embed o atributo propio que se serializa como `{{…}}`) | Prueba: guardar/cargar conserva `{{comprador.nombre}}` |
| W1.4 | Inserción de **bloques condicionales** (`si producto.online`, `si comprador.personaMoral`, …) envolviendo la selección | — |
| W1.5 | **Sección de nombres y firmas** mostrada al final del editor como bloque **gris de solo lectura** (“Se agrega automáticamente: firmas de VENDEDOR y COMPRADOR”), no editable ni borrable | — |
| W1.6 | Opción “Rúbrica en cada página” | — |
| W1.7 | Autoguardado de borrador (`PUT /plantillas/{id}/borrador` con `version`; manejo de 409) | — |
| W1.8 | **Vista previa PDF** lado a lado con selector de datos de ejemplo (física/moral × offline/online) usando `contratos_pdf` — mismo generador que la App | El PDF incluye la sección de firmas al final (con líneas vacías) y la hoja de constancia de ejemplo |
| W1.9 | Validación en vivo con `ValidadorPlantilla` (errores marcados en el texto) y al publicar con `POST /plantillas/{id}/validar` | Variable inválida impide publicar |
| W1.10 | **Publicar** con nota de cambio; historial de versiones; ver una versión (solo lectura + vista previa); **duplicar** versión como borrador; **retirar** | Una versión publicada llega a la App tras sincronizar → **H3** |
| W1.11 | Catálogo de productos (CU-P07): tipo OFFLINE/ONLINE, funcionalidades ordenables, precio, periodicidad, plantilla | — |

## W2 · Eventos, seguimiento, pagos, administración e infraestructura (día 2, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| W2.1 | Eventos (CU-P04): lista con filtros; detalle con datos, productos, visor PDF, constancia, evidencias (identificación, firmas, selfie, mapa con coordenadas) y eventos ligados | — |
| W2.2 | Seguimiento de entrega (CU-P05): tablero kanban por estado + lista de atrasados; responsable, fecha compromiso, tareas; `entregado-manual` con justificación | — |
| W2.3 | Pagos (CU-P06): registrar con comprobante; saldo por venta | — |
| W2.4 | Suscripciones: lista con filtros (por vencer, vencidas), suspender/reactivar/cancelar con motivo | — |
| W2.5 | Usuarios, empresa (logo, datos fiscales), auditoría | — |
| W2.6 | Verificador de integridad (CU-P09): subir PDF o capturar folio | — |
| W2.7 | **Infraestructura → Base de datos** (CU-P10): reautenticación + 2FA; ver conexión actual; formulario ConexionBd; *Probar*; resultado de esquema; *Solo cambiar* / *Migrar y cambiar* con confirmación escrita; barra de progreso y log de la operación (polling `/infraestructura/operaciones/{id}`); *Revertir* | Simulacro contra segunda BD |
| W2.8 | Infraestructura → Respaldos: lista y “Respaldar ahora” | — |
| W2.9 | Build `prod` y publicación de imagen; se sirve en `https://portal.<dominio>` vía Caddy | **H4** |

## Comandos de verificación
```
flutter analyze
flutter test
flutter run -d chrome --dart-define=API=mock
flutter build web --release --dart-define=API_URL=https://api.<dominio>/api/v1
docker build -t contratos-portal .
```
