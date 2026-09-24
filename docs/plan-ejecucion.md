# Plan de ejecución — App-Contratos

> Versión 0.4 · 2026-09-24 · Referencias: [requerimientos.md](requerimientos.md), [casos-de-uso.md](casos-de-uso.md), [contrato-api.md](contrato-api.md)

## 1. Enfoque

El código lo construye **Claude** a partir de esta documentación, con un **repositorio por proyecto** y una **sesión de Claude por repositorio** trabajando en paralelo. Para que las sesiones no se desalineen:

1. **[contrato-api.md](contrato-api.md) es la frontera** entre repositorios. API, App y Portal se construyen contra él, no uno contra el otro. Si algo cambia, primero se actualiza el contrato.
2. Cada repositorio tiene su **plan de trabajo** en [planes/](planes/) con tareas numeradas, criterios de aceptación y comandos de verificación.
3. Cada repositorio lleva un `CLAUDE.md` con: enlace a esta documentación, stack, convenciones, cómo compilar/probar y “no inventar endpoints fuera de contrato-api.md”.
4. Se trabaja por **tareas pequeñas verificables**: cada tarea termina con compilación + pruebas en verde y un commit.
5. Cada feature genera sus documentos de **requerimiento, análisis, plan, pruebas y resumen** y actualiza `ESTADO.md`, según [proceso/flujo-de-trabajo.md](proceso/flujo-de-trabajo.md).

## 2. Repositorios y planes

| Repositorio | Plan | Depende de |
|---|---|---|
| `contratos-dart` | [planes/plan-dart.md](planes/plan-dart.md) | contrato-api.md |
| `contratos-api` | [planes/plan-api.md](planes/plan-api.md) | contrato-api.md |
| `contratos-app` | [planes/plan-app.md](planes/plan-app.md) | `contratos-dart` |
| `contratos-portal` | [planes/plan-portal.md](planes/plan-portal.md) | `contratos-dart` |

## 3. Calendario (2 días)

```
            Día 1 — mañana        Día 1 — tarde           Día 2 — mañana          Día 2 — tarde
Dart     │ D1 modelos+cliente │ D2 motor plantillas+PDF│ (ajustes)             │
API      │ A1 base+BD+auth    │ A2 CRM+catálogo+plantil│ A3 sync+eventos        │ A4 seguim+pagos+infra BD │ A5 deploy OVH
App      │ P0 base+lienzo SPen│ P1 SQLite+clientes+sync│ P2 venta+PDF+firma     │ P3 envío eventos         │
Portal   │                    │ W0 base+login+CRM      │ W1 editor plantillas   │ W2 eventos+seguim+pagos+infra│
Integr.  │                    │                        │                        │ Prueba de punta a punta  │
```

| Hito | Cuándo | Qué se puede probar |
|---|---|---|
| H1 | Fin día 1 mañana | API responde `/health` en contenedor local; lienzo S Pen funcionando en el Samsung |
| H2 | Fin día 1 | Clientes y catálogo sincronizan App ↔ API local; Portal lista clientes; `contratos_pdf` genera un PDF con bloque de firmas |
| H3 | Día 2 mediodía | Venta firmada en modo avión; editor de plantillas publica una versión que llega a la App |
| H4 | Fin día 2 | **Punta a punta en OVH**: pre-registro → registro con INE → venta → firma con S Pen → envío → visible en Portal → acta de entrega → seguimiento `ENTREGADO` |

### Mock de la API
Mientras `contratos-api` no esté lista, App y Portal usan un **mock** generado desde el OpenAPI (o las respuestas de ejemplo de contrato-api.md) incluido en `contratos_modelos` (`ApiMock`). Así el día 1 no se bloquea nadie.

## 4. Fuera del control de Claude (hacerlo en paralelo)

Estos puntos **no** caben en los 2 días de construcción y conviene arrancarlos ya:

| Tarea | Responsable | Bloquea |
|---|---|---|
| Contratar VPS OVHcloud, dominio y DNS (`api.`, `portal.`) | Tú | Despliegue (A5) |
| Crear repositorios en GitHub y dar acceso | Tú | Todo |
| Tener a mano el teléfono y la tableta Samsung con **depuración USB** habilitada | Tú | Pruebas de S Pen |
| **Revisión de plantillas por un abogado** | Abogado | Uso con clientes reales (no bloquea construcción: se usan las plantillas base de [contrato-base.md](contrato-base.md)) |
| Aviso de privacidad (LFPDPPP) | Abogado | Uso con clientes reales |
| Pruebas con usuarios reales en campo | Tú | Salida a producción |
| Publicación en Google Play (si aplica; si no, APK instalado directamente) | Tú | Distribución |

## 5. Definición de terminado (v1)

- [ ] Los 4 repositorios compilan y sus pruebas pasan en CI.
- [ ] API y Portal desplegados en el VPS OVH con HTTPS.
- [ ] Recorrido H4 completo en el Samsung real con S Pen.
- [ ] Un evento reenviado dos veces no se duplica; un evento con el PDF alterado se rechaza.
- [ ] Una plantilla editada y publicada en el Portal se ve igual en la vista previa y en el PDF firmado en la App.
- [ ] Respaldo diario de PostgreSQL ejecutándose y restaurado al menos una vez.
- [ ] Simulacro de cambio de conexión de BD desde el Portal (a una segunda BD en el mismo VPS) y reversión.

## 6. Después de v1

| Feature | Valor |
|---|---|
| OTP al comprador (correo/SMS) | Refuerza la identidad del firmante |
| Constancia **NOM-151** vía Prestador de Servicios de Certificación | Máxima fuerza probatoria |
| Migración real a **OVH Managed PostgreSQL** y **OVH Object Storage** | Operación sin mantener la BD en el VPS |
| Pasarela de pago recurrente (Stripe / Mercado Pago / Conekta) | Cobro automático de suscripciones |
| Facturación CFDI (PAC con API) | Facturar ventas y suscripciones |
| Portal del cliente final | Autoservicio |
