# Flujo de trabajo con varias sesiones de Claude

> Versión 0.1 · 2026-09-24

## 1. Organización en disco y en VSCode

```
C:\Apps\
  App-Contratos\              ← repo App-contratos (esta documentación)
    contratos.code-workspace  ← abre todo junto
  contratos-dart\             ← repo
  contratos-api\              ← repo
  contratos-app\              ← repo
  contratos-portal\           ← repo
```

| Ventana de VSCode | Qué abre | Sesión de Claude | Rol |
|---|---|---|---|
| **Principal** | `contratos.code-workspace` (todas las carpetas) | Sesión **coordinadora** | Mantiene la documentación central, revisa avances, resuelve dudas entre repos, aplica cambios al contrato de la API |
| Dart | `C:\Apps\contratos-dart` | Sesión Dart | Construye `contratos-dart` |
| API | `C:\Apps\contratos-api` | Sesión API | Construye `contratos-api` |
| App | `C:\Apps\contratos-app` | Sesión App | Construye `contratos-app` |
| Portal | `C:\Apps\contratos-portal` | Sesión Portal | Construye `contratos-portal` |

> En la sesión coordinadora, si Claude no puede leer las carpetas hermanas, agrégalas con `/add-dir C:\Apps\contratos-api` (y las demás).

## 2. Reglas entre sesiones

1. **La documentación central es de solo lectura para las sesiones de repositorio.** `requerimientos.md`, `casos-de-uso.md`, `contrato-api.md`, `contrato-base.md` y `planes/` solo los modifica la sesión coordinadora.
2. Si una sesión de repositorio necesita un cambio en el contrato de la API (un campo, un endpoint, un código de error), lo escribe en la sección **“Cambios propuestos a la documentación central”** de su documento de análisis y **se detiene en esa parte** hasta que la coordinadora lo apruebe y actualice `contrato-api.md`.
3. Cada sesión **solo modifica su propio repositorio**.
4. Cada feature se trabaja en su **rama** `feature/<ID>-<nombre-corto>` y se integra a `main` al cerrarse.
5. `ESTADO.md` en la raíz de cada repo se actualiza **al empezar y al terminar cada sección** del plan. Es lo que lee la coordinadora.
6. Nada se da por terminado sin **pruebas en verde** y el documento de pruebas con resultados reales (salida de los comandos).

## 3. Ciclo de vida de un feature

Cada feature (bloque del plan de su repo: D1, D2, A1…A5, P0…P3, W0…W2) genera una carpeta:

```
<repo>/docs/features/<ID>-<nombre-corto>/
  01-requerimiento.md   Qué entra, qué NO entra, qué no se debe tocar
  02-analisis.md        Situación actual del proyecto, dependencias, riesgos, decisiones
  03-plan.md            Secciones a construir, en orden, con su prueba cada una
  04-pruebas.md         Casos de prueba y resultados reales
  05-resumen.md         Qué se construyó, explicado de forma didáctica
```

Plantillas en [plantillas/](plantillas/).

```
 ┌──────────────┐   ┌────────────┐   ┌──────────┐   ┌──────────────────────┐   ┌──────────┐   ┌────────────┐
 │01 Requerim.  │──►│02 Análisis │──►│03 Plan   │──►│ Construir sección    │──►│04 Pruebas│──►│05 Resumen  │
 │ (alcance)    │   │ (situación)│   │ (orden)  │   │ + su prueba, commit  │   │ (todo OK)│   │ (didáctico)│
 └──────────────┘   └────────────┘   └──────────┘   │ (repetir por sección)│   └──────────┘   └────────────┘
                          │                         └──────────────────────┘
                          └─► si hay cambios a la documentación central → pedir a la coordinadora
```

### Paso a paso para la sesión de repositorio
1. Leer `CLAUDE.md`, `ESTADO.md` y el bloque del feature en su plan (`App-Contratos/docs/planes/plan-<repo>.md`).
2. Crear rama y carpeta del feature.
3. **01-requerimiento.md**: copiar el alcance del plan, precisar criterios de aceptación y listar explícitamente lo que **no** se toca (otros features, archivos compartidos, contratos).
4. **02-analisis.md**: revisar el estado real del código (qué existe, qué compila, qué pruebas hay, qué dependencias están listas), riesgos y decisiones técnicas.
5. **03-plan.md**: secciones numeradas en el orden en que se construirán; cada una con archivos que toca, prueba que la valida y comando para verificarla.
6. Construir **sección por sección**: código + prueba → comando de verificación en verde → commit (`<ID>.<n>: descripción`) → actualizar `ESTADO.md`.
7. **04-pruebas.md**: ejecutar la suite completa (no solo las pruebas nuevas, para comprobar que no se rompió nada de otros features) y pegar resultados.
8. **05-resumen.md**: explicar de forma didáctica qué se construyó.
9. Actualizar el README del repo si el feature cambia cómo se instala, ejecuta o usa.
10. Integrar la rama a `main` y marcar el feature como ✅ en `ESTADO.md`.

## 4. Protección entre features

- El requerimiento de cada feature lista **archivos y módulos fuera de alcance**. Si durante la construcción hace falta tocarlos, se documenta en el análisis y se justifica.
- Antes de cerrar un feature se ejecuta **toda** la suite de pruebas del repo (regresión).
- Los cambios a modelos compartidos (`contratos-dart`) se publican con **tag nuevo**; App y Portal actualizan la referencia en un feature propio, no “de paso”.
- Las migraciones de base de datos solo **agregan**; no se modifican migraciones ya integradas a `main`.

## 5. Monitoreo desde la sesión coordinadora

La coordinadora, a petición tuya (p. ej. “revisa el avance”):
1. Lee `ESTADO.md` de los cuatro repositorios.
2. Revisa `git log` reciente y las carpetas `docs/features/` en curso.
3. Atiende los **cambios propuestos a la documentación central** pendientes.
4. Detecta bloqueos entre repos (p. ej. la App esperando un tag de `contratos-dart`).
5. Te entrega un reporte corto: qué avanzó, qué está bloqueado, qué decisión necesita de ti.

## 6. Catálogo de features

| ID | Repo | Nombre | Depende de |
|---|---|---|---|
| D1 | dart | Modelos y cliente HTTP | contrato-api.md |
| D2 | dart | Motor de plantillas y PDF | D1 |
| A1 | api | Base, BD y autenticación | — |
| A2 | api | CRM, catálogo y plantillas | A1 |
| A3 | api | Sincronización y eventos | A2 |
| A4 | api | Seguimiento, pagos e infraestructura | A3 |
| A5 | api | Despliegue en OVH | A4, W0 (imagen del portal) |
| P0 | app | Base y lienzo S Pen | — |
| P1 | app | SQLite, clientes y sincronización | D1, P0 |
| P2 | app | Venta, PDF y firma | D2, P1 |
| P3 | app | Envío de eventos | P2 |
| W0 | portal | Base, login y CRM | D1 |
| W1 | portal | Editor de contratos y plantillas | D2, W0 |
| W2 | portal | Eventos, seguimiento, pagos e infraestructura | W1 |

Detalle de cada feature: [planes/](../planes/).
