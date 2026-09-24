# Plan de trabajo — `contratos-app`

> Leer antes: [requerimientos.md §3, §5](../requerimientos.md), [casos-de-uso.md](../casos-de-uso.md) (CU-A*), [contrato-api.md](../contrato-api.md), [plan-dart.md](plan-dart.md).

## Stack
Flutter estable · Android (Samsung + S Pen) principal, iOS opcional · `contratos_modelos` + `contratos_pdf` (git) · drift + `sqlcipher_flutter_libs` · riverpod · go_router · `flutter_secure_storage` · `local_auth` · `camera` / `image_picker` · `geolocator` · `connectivity_plus` · `workmanager` · `share_plus` · `printing` (vista previa).

## Arquitectura
```
lib/
  core/        db (drift), repositorios, servicios (sync, envío), seguridad, conectividad
  features/
    auth/  clientes/  sincronizacion/  eventos/  firma/  pendientes/  perfil/
  main.dart
```
Regla central: **la UI solo escribe en repositorios SQLite**. Los servicios `SincronizacionService` y `EnvioEventosService` leen de SQLite y hablan con `ContratosApi`. Con `--dart-define=API=mock` se usa `ApiMock`.

---

## P0 · Base y lienzo S Pen (día 1, mañana)

| # | Tarea | Aceptación |
|---|---|---|
| P0.1 | Proyecto, `CLAUDE.md`, flavors `mock`/`dev`/`prod`, CI (analyze + test + APK) | APK de prueba instalable |
| P0.1b | **Entorno local** según [entorno-local.md](../entorno-local.md): flavor `dev` con `API_URL=http://localhost:5080/api/v1`, HTTP sin cifrar permitido solo en `dev` y solo para `localhost`, `scripts/dev.ps1` con `adb reverse`, `pubspec_overrides.yaml` en `.gitignore` | `.\scripts\dev.ps1` instala y abre la App en el Samsung |
| P0.2 | **Widget `LienzoFirma`**: `Listener` sobre pointer events; en modo S Pen solo acepta `PointerDeviceKind.stylus` (e `invertedStylus` = borrar), ignora `touch` (rechazo de palma); interruptor para permitir dedo | Probado en teléfono y tableta Samsung |
| P0.3 | Grosor variable por presión (`event.pressure`), suavizado (curvas cuadráticas), pantalla completa horizontal, *Borrar* / *Aceptar* | Firma se ve natural |
| P0.4 | Exportar **PNG** (fondo transparente, 1600×700) y **trazo JSON** (contrato-api.md §5.2) | Prueba unitaria del JSON |
| P0.5 | Pantalla de demo del lienzo para validar en dispositivo | **H1** |

## P1 · SQLite, clientes y sincronización (día 1, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| P1.1 | Esquema drift cifrado (requerimientos §11, parte SQLite) con columnas de control (`pendienteSubir`, `estadoEnvio`, `intentos`, `ultimoError`, `fechaUltimoIntento`); llave de cifrado en `flutter_secure_storage` | Migración de esquema probada |
| P1.2 | Archivos locales (identificaciones, firmas, PDFs) cifrados en directorio privado | — |
| P1.3 | Login (online) + sesión persistente + desbloqueo PIN/biometría offline | CU-A01 |
| P1.4 | Firma del vendedor (CU-A02) usando `LienzoFirma` + PIN de firma | — |
| P1.5 | Lista y búsqueda de clientes; **pre-registro rápido** (CU-A04) con aviso de duplicados | — |
| P1.6 | **Registro completo** (CU-A05): formulario física/moral, validación RFC, captura de identificación con cámara (anverso/reverso), vigencia, poder; `calcularEstadoRegistro` de `contratos_modelos` | Pruebas de widget |
| P1.7 | `SincronizacionService` (CU-A03): push clientes pendientes (multipart), pull clientes y catálogo con `marca`, descarga de logo/firma vendedor; muestra última sincronización | Prueba con `ApiMock` y con API local → **H2** |

## P2 · Venta, PDF y firma (día 2, mañana)

| # | Tarea | Aceptación |
|---|---|---|
| P2.1 | Asistente de venta (CU-A06): cliente → productos → ajustes (precio, anticipo, fecha compromiso, alcance extra/excluido, lugar) → vista previa (`contratos_pdf` sin firmas, `printing`) | — |
| P2.2 | Bloqueo si cliente no está `COMPLETO` → abre registro completo | — |
| P2.3 | Folio local `{TIPO}-{iniciales}-{yyyyMMdd}-{consecutivo}` | Prueba |
| P2.4 | **Modo presentación** (CU-A07): `SystemChrome` inmersivo, bloqueo de navegación atrás, lectura completa obligatoria (scroll al final), casillas de aceptación (5), nombre prellenado | — |
| P2.5 | Firma del comprador con `LienzoFirma` (S Pen); selfie opcional; confirmación del vendedor con PIN/biometría | — |
| P2.6 | Evidencias: geolocalización (con timeout; si falla, se registra “no disponible”), dispositivo, versión | — |
| P2.7 | Generar PDF final con `contratos_pdf` (firmas + rúbrica + constancia), calcular SHA-256, guardar evento `FIRMADO` inmutable; al cancelar antes de firmar se **eliminan** imagen y trazo | Prueba: no queda firma huérfana en SQLite ni en disco |
| P2.8 | Compartir PDF (`share_plus`) | Venta completa en **modo avión** → **H3** |

## P3 · Envío de eventos (día 2, tarde)

| # | Tarea | Aceptación |
|---|---|---|
| P3.1 | `EnvioEventosService` (CU-A08): al firmar → `ENVIANDO` → `POST /eventos` multipart; maneja respuestas según contrato-api.md §5.1 (201/200 → `ENVIADO`; 409 → `ERROR_ENVIO` sin reintento automático; 422 hash → reintento; 422 validación → sin reintento; 503/red → reintento) | Pruebas con `ApiMock` que simula cada código |
| P3.2 | Reintento automático: al recuperar conectividad (`connectivity_plus`) y periódico en segundo plano (`workmanager`, cada 15 min), con espera progresiva | — |
| P3.3 | Pantalla **Eventos pendientes** (CU-A09): estado, intentos, último error, “Enviar” y “Enviar todos” | — |
| P3.4 | Indicador en inicio: en línea / sin conexión / N pendientes | — |
| P3.5 | Eventos `ENTREGA`, `MODIFICACION`, `CANCELACION` desde una venta (CU-A10), con firma nueva | — |
| P3.6 | Depuración de eventos `ENVIADO` con más de N días (configurable); nunca los no enviados | Prueba |
| P3.7 | Build `prod` firmado apuntando a `https://api.<dominio>` | Recorrido **H4** en el Samsung |

## Comandos de verificación
```
flutter analyze
flutter test
flutter run --flavor mock -d <samsung>
flutter build apk --flavor prod --release
```
