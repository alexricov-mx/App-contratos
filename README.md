# App-Contratos

Sistema para **vender aplicaciones de software a negocios locales** y dejar cada venta respaldada con un **contrato firmado en sitio**, aunque no haya internet.

Este repositorio (`App-contratos`) contiene la documentación de toda la solución. El código vive en cuatro repositorios separados, descritos abajo.

---

## 1. El problema que resuelve

Cuando vendes una aplicación a un negocio, conviene dejar por escrito:

- **Qué** se entrega exactamente (y qué no).
- **Cuánto** se paga y cómo: **pago único** si la app funciona sin internet, **suscripción** mensual o anual si depende de un servidor.
- Que **no** eres responsable de la calidad de los datos que el cliente capture ni del uso distinto que le dé.

Y quieres que ese documento se firme **en el momento**, en el negocio del cliente, con tu tableta o teléfono y el **S Pen**, y que después sirva como **respaldo legal**.

## 2. La solución en un dibujo

```
   EN CAMPO                                   EN LA NUBE (VPS OVH)                     EN LA OFICINA
┌──────────────────┐                     ┌──────────────────────────┐              ┌──────────────────┐
│  App de campo    │   1. sincroniza     │                          │              │ Portal de        │
│  (Flutter)       │◄──── clientes y ────┤   API (.NET 10)          │◄────────────►│ administración   │
│                  │      catálogo       │   única puerta a la BD   │   CRM,       │ (Flutter Web)    │
│  SQLite local    │                     │                          │   eventos,   │                  │
│  firma con S Pen │   2. envía evento   │   ┌──────────────────┐   │   entregas,  │ editor de        │
│  genera PDF      ├──── firmado ───────►│   │ PostgreSQL       │   │   pagos      │ contratos        │
└──────────────────┘                     │   └──────────────────┘   │              └──────────────────┘
                                         └──────────────────────────┘
```

1. El **vendedor** usa la **App** en el negocio del cliente. Todo se guarda primero en el teléfono o tableta (SQLite), así que funciona sin internet.
2. Registra al cliente, arma la venta, la App genera el **contrato en PDF** y el comprador lo **firma con el S Pen**.
3. Al terminar, la App **envía el evento** (contrato + firmas + evidencias) a la **API**. Si no hay señal, lo reintenta sola o el vendedor lo reenvía a mano.
4. La **API** es la única que habla con la base de datos **PostgreSQL**.
5. La **oficina** usa el **Portal** para dar seguimiento: clientes, contactos, ventas, entregas, pagos, suscripciones y el **texto de los contratos**.

## 3. Conceptos clave

| Concepto | Qué significa aquí |
|---|---|
| **Evento** | Cada momento en que el comprador firma algo: una **venta**, una **entrega**, una **modificación** o una **cancelación**. Cada evento produce un PDF. |
| **Firma de un solo uso** | La firma del comprador vale solo para el documento de ese evento. Si mañana firma otra cosa, firma de nuevo. |
| **Pre-registro / registro completo** | Un cliente se puede capturar rápido con nombre y teléfono; para firmar un contrato se necesita su **identificación oficial**. |
| **Offline-first** | La App siempre guarda primero en el dispositivo; el internet solo se usa para enviar y recibir. |
| **Idempotencia** | Si la App envía el mismo evento dos veces (por ejemplo, porque se cortó la señal), el servidor no lo duplica. |
| **Hash SHA-256** | “Huella digital” del PDF. Si alguien cambia una sola letra, la huella cambia; así se prueba que el documento no se alteró. |
| **Plantilla** | El texto del contrato con espacios como `{{comprador.nombre}}` que el sistema llena. Se edita en el Portal; la sección de firmas se agrega siempre al final. |

## 4. Repositorios

| Repositorio | Qué es | Tecnología |
|---|---|---|
| **`App-contratos`** (este) | Requerimientos, casos de uso, contrato de la API, planes y proceso de trabajo | Markdown |
| **`contratos-dart`** | Piezas compartidas por App y Portal: modelos de datos, cliente de la API y **generador de PDF** | Dart |
| **`contratos-api`** | La API y la configuración para desplegar todo en el servidor | .NET 10, PostgreSQL, Docker |
| **`contratos-app`** | La App de campo para el vendedor | Flutter (Android, Samsung + S Pen) |
| **`contratos-portal`** | El Portal de administración para la oficina | Flutter Web |

```
contratos-dart  ──(dependencia)──►  contratos-app
       │
       └────────(dependencia)──►  contratos-portal

contratos-app ──HTTP──► contratos-api ◄──HTTP── contratos-portal
                              │
                          PostgreSQL
```

## 5. Documentación

| Documento | Para qué sirve |
|---|---|
| [requerimientos.md](docs/requerimientos.md) | Qué debe hacer el sistema, reglas de negocio, arquitectura, modelo de datos |
| [casos-de-uso.md](docs/casos-de-uso.md) | Paso a paso de cada acción de los usuarios y del sistema |
| [contrato-api.md](docs/contrato-api.md) | **Fuente de verdad** de cómo se comunican App, Portal y API |
| [contrato-base.md](docs/contrato-base.md) | Estructura legal de los contratos y qué les da fuerza probatoria |
| [plan-ejecucion.md](docs/plan-ejecucion.md) | Calendario, hitos y definición de terminado |
| [planes/](docs/planes/) | Plan de trabajo detallado por repositorio |
| [proceso/flujo-de-trabajo.md](docs/proceso/flujo-de-trabajo.md) | Cómo trabajan las sesiones de Claude en paralelo y qué documentos genera cada feature |
| [proceso/plantillas/](docs/proceso/plantillas/) | Plantillas de requerimiento, análisis, plan, pruebas, resumen y estado |
| [repos/](repos/) | Archivos iniciales (`README.md`, `CLAUDE.md`, `ESTADO.md`) para copiar a la raíz de cada repositorio al crearlo |

## 6. Cómo se construye

La solución la construye **Claude** en varias ventanas de VSCode, una por repositorio, coordinadas desde este repositorio:

1. Clona los cinco repositorios como carpetas hermanas en `C:\Apps\` y copia a cada uno sus archivos iniciales desde [repos/](repos/).
2. Abre [contratos.code-workspace](contratos.code-workspace) en una ventana: ahí trabaja la **sesión coordinadora**.
3. Abre cada repositorio en su propia ventana con su propia sesión de Claude.
4. Cada sesión sigue su plan en [docs/planes/](docs/planes/) feature por feature, generando en su repo `docs/features/<ID>/` los documentos de requerimiento, análisis, plan, pruebas y resumen.
5. La coordinadora revisa los `ESTADO.md` de cada repo y mantiene la documentación central al día.

Detalle en [flujo-de-trabajo.md](docs/proceso/flujo-de-trabajo.md).

## 7. Estado legal

Las plantillas de contrato son una **guía técnica**, no asesoría legal. Antes de usarlas con clientes reales deben ser revisadas por un abogado en México, junto con el aviso de privacidad (LFPDPPP).
