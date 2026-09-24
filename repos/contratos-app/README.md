# contratos-app

App de campo de **App-Contratos**: la herramienta del vendedor para cerrar ventas en el negocio del cliente, con contrato en PDF y **firma con S Pen**, funcione o no el internet.

> Documentación completa de la solución: repositorio [`App-contratos`](../App-Contratos/README.md).

---

## Parte 1 · La solución en general

App-Contratos permite **vender aplicaciones de software a negocios locales** y cerrar cada venta con un **contrato firmado en sitio**.

```
App de campo (Flutter + SQLite) ──► API (.NET 10) ──► PostgreSQL
                                        ▲
Portal de administración (Flutter Web) ─┘
```

| Repositorio | Papel |
|---|---|
| `App-contratos` | Requerimientos, contrato de la API, planes |
| `contratos-dart` | Modelos, cliente HTTP y generador de PDF compartidos |
| `contratos-api` | API y despliegue |
| **`contratos-app`** | **Este repositorio** |
| `contratos-portal` | Portal de la oficina |

---

## Parte 2 · Este componente

### ¿Qué hace?
1. **Clientes**: pre-registro rápido (nombre + teléfono) y registro completo con **foto de la identificación oficial**.
2. **Venta**: elige cliente y producto, ajusta precio y alcance, y ve la vista previa del contrato.
3. **Firma**: entrega la tableta al comprador en **modo presentación**; lee el contrato, acepta las casillas y **firma con el S Pen**. El vendedor confirma su firma con PIN o huella.
4. **PDF**: genera el documento con ambas firmas y una hoja de constancia (fecha, lugar, dispositivo, huella digital del archivo).
5. **Envío**: manda el evento a la API. Si no hay señal, lo guarda y lo **reintenta solo**; también se puede **reenviar a mano**.
6. **Sincronización**: baja clientes, productos y plantillas; sube los clientes capturados en campo.

### Idea central: offline-first
```
 Pantalla ──► Repositorios ──► SQLite (cifrado)       ← siempre primero aquí
                                   │
                    ┌──────────────┴───────────────┐
          SincronizacionService           EnvioEventosService
         (clientes y catálogo)          (eventos firmados, reintentos)
                    └──────────────┬───────────────┘
                                   ▼
                                  API
```
La pantalla **nunca espera al internet** para guardar. Los servicios de fondo se encargan de hablar con la API cuando se puede.

### El lienzo de firma y el S Pen
- En modo S Pen el lienzo **solo acepta el lápiz**: si apoyas la mano en la pantalla no pinta nada.
- Registra la **presión** del lápiz (trazo más grueso al apretar) y guarda el **trazo completo** (puntos, presión y tiempo) como evidencia adicional.
- La firma del comprador es **de un solo uso**: queda ligada al documento de ese evento y no se puede reutilizar.

### Estructura
```
lib/
  core/        base de datos (drift), repositorios, servicios, seguridad, conectividad
  features/    auth/ clientes/ sincronizacion/ eventos/ firma/ pendientes/ perfil/
test/
docs/features/ documentación de cada feature construido
```

### Tecnologías
Flutter · drift + SQLCipher · riverpod · go_router · `contratos_modelos` y `contratos_pdf` (de `contratos-dart`) · local_auth · camera · geolocator · connectivity_plus · workmanager.

### Requisitos para desarrollar
- Flutter estable
- Teléfono o tableta **Samsung con S Pen** con depuración USB activada (el emulador no simula el S Pen)

### Ejecutar
```bash
flutter pub get
flutter run --flavor mock -d <dispositivo>   # sin API, con datos de ejemplo
flutter run --flavor dev  -d <dispositivo>   # contra la API local
```

### Pruebas
```bash
flutter analyze
flutter test
```

### Generar instalable
```bash
flutter build apk --flavor prod --release
```

### Documentos de referencia
- Requerimientos de la App: `App-contratos/docs/requerimientos.md` §5
- Plan de trabajo: `App-contratos/docs/planes/plan-app.md`
- Estado actual: [ESTADO.md](ESTADO.md)
- Features construidos: [docs/features/](docs/features/)
