# contratos-portal

Portal de administración de **App-Contratos**: donde la oficina da seguimiento a clientes, ventas, entregas y pagos, y donde se **edita el texto de los contratos**.

> Documentación completa de la solución: repositorio [`App-contratos`](../App-Contratos/README.md).

---

## Parte 1 · La solución en general

App-Contratos permite **vender aplicaciones de software a negocios locales** y cerrar cada venta con un **contrato firmado en sitio** con el S Pen.

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
| `contratos-app` | App del vendedor en campo |
| **`contratos-portal`** | **Este repositorio** |

---

## Parte 2 · Este componente

### ¿Qué hace?
| Sección | Para qué |
|---|---|
| **Tablero** | Lo urgente de un vistazo: clientes por completar, entregas atrasadas, pagos vencidos, vendedores con eventos sin enviar |
| **Clientes** | CRM: ficha, identificación, completar pre-registros hechos en campo, fusionar duplicados |
| **Contactos e interacciones** | Varias personas por cliente e historial de llamadas, visitas y mensajes con próximos seguimientos |
| **Eventos** | Cada venta, entrega, modificación o cancelación firmada, con su PDF y evidencias |
| **Seguimiento de entrega** | Tablero por estado: recibido → en proceso → listo → entregado → cerrado |
| **Pagos y suscripciones** | Registrar pagos, suspender o reactivar suscripciones |
| **Plantillas** | **Editor del texto de los contratos** |
| **Productos, usuarios, empresa** | Catálogo y administración |
| **Infraestructura** | Cambiar la base de datos a la que apunta la API y ver respaldos (solo superadministrador) |

### El editor de contratos
- Se escribe el contrato como en un procesador de textos sencillo (títulos, negritas, listas).
- Los datos variables se insertan desde un panel: `{{comprador.nombre}}`, `{{vendedor.nombre}}`, `{{precio.total}}`… Se ven como etiquetas de color.
- Los párrafos que solo aplican a algunos casos se envuelven en condiciones (*solo si el producto es online*, *solo si es persona moral*).
- **La sección de nombres y firmas no se edita**: el sistema la agrega siempre al final.
- La **vista previa** usa el mismo generador de PDF que la App (`contratos_pdf`), así que lo que ves aquí es exactamente lo que se firmará en campo.
- Cada **Publicar** crea una versión nueva; los contratos ya firmados nunca cambian.

### Estructura
```
lib/
  core/        cliente de API, sesión, menú, tema, widgets comunes
  features/    tablero/ clientes/ contactos/ eventos/ seguimiento/ pagos/
               suscripciones/ productos/ plantillas/ usuarios/ empresa/
               integridad/ infraestructura/
test/
docs/features/ documentación de cada feature construido
Dockerfile     build web servido con Nginx
```

### Tecnologías
Flutter Web · riverpod · go_router · flutter_quill · printing · `contratos_modelos` y `contratos_pdf` (de `contratos-dart`) · Nginx (Docker).

### Ejecutar
```bash
flutter pub get
flutter run -d chrome --dart-define=API=mock                                    # sin API
flutter run -d chrome --dart-define=API_URL=http://localhost:5080/api/v1        # API local
```

### Pruebas
```bash
flutter analyze
flutter test
```

### Construir imagen
```bash
docker build -t contratos-portal --build-arg API_URL=https://api.<dominio>/api/v1 .
```
Se despliega junto con la API mediante `contratos-api/deploy/docker-compose.yml`.

### Documentos de referencia
- Requerimientos del Portal: `App-contratos/docs/requerimientos.md` §7
- Plan de trabajo: `App-contratos/docs/planes/plan-portal.md`
- Estado actual: [ESTADO.md](ESTADO.md)
- Features construidos: [docs/features/](docs/features/)
