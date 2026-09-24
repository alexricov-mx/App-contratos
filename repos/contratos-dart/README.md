# contratos-dart

Paquetes Dart **compartidos** por la App de campo y el Portal de **App-Contratos**.

> Documentación completa de la solución: repositorio [`App-contratos`](../App-Contratos/README.md).

---

## Parte 1 · La solución en general

App-Contratos permite **vender aplicaciones de software a negocios locales** y cerrar cada venta con un **contrato firmado en sitio** con el S Pen.

```
App de campo (Flutter + SQLite) ──► API (.NET 10) ──► PostgreSQL
                                        ▲
Portal de administración (Flutter Web) ─┘

        contratos-dart ──► usado por la App y por el Portal
```

| Repositorio | Papel |
|---|---|
| `App-contratos` | Requerimientos, contrato de la API, planes |
| **`contratos-dart`** | **Este repositorio** |
| `contratos-api` | API y despliegue |
| `contratos-app` | App del vendedor en campo |
| `contratos-portal` | Portal de la oficina |

---

## Parte 2 · Este componente

### ¿Por qué existe?
La App y el Portal hablan con la misma API y **deben producir exactamente el mismo PDF**: el Portal lo muestra como vista previa al editar una plantilla y la App lo genera al firmar. Si cada uno tuviera su propio código, tarde o temprano no coincidirían. Aquí vive ese código **una sola vez**.

### Paquetes

| Paquete | Contenido |
|---|---|
| `contratos_modelos` | Clases de datos (Cliente, Evento, Plantilla…) idénticas a `contrato-api.md`, reglas compartidas (p. ej. cuándo un cliente está “completo”), cliente HTTP `ContratosApi` y `ApiMock` para trabajar sin servidor |
| `contratos_pdf` | Motor de plantillas (sustituye `{{variables}}` y evalúa bloques `{{#si}}`), validador de plantillas y **generador de PDF**: contenido, sección fija de nombres y firmas, rúbrica por página y hoja de constancia |

### Cómo se genera un contrato
```
Plantilla (Quill Delta)  +  Datos del evento (comprador, vendedor, productos, precio)
            │                              │
            └──────────► ResolvedorVariables ◄┘
                               │
                         GeneradorPdf
                               │
   contenido → sección de firmas (siempre al final) → hoja de constancia
                               │
                          PDF + SHA-256
```

### Uso desde App o Portal
```yaml
dependencies:
  contratos_modelos:
    git:
      url: https://github.com/alexricov-mx/contratos-dart.git
      path: packages/contratos_modelos
      ref: v0.2.0
  contratos_pdf:
    git:
      url: https://github.com/alexricov-mx/contratos-dart.git
      path: packages/contratos_pdf
      ref: v0.2.0
```
Siempre se referencia un **tag**, nunca `main`, para que un cambio aquí no rompa la App o el Portal sin aviso.

### Desarrollo
```bash
dart pub global activate melos
melos bootstrap
melos run test
```

### Pruebas “golden” de PDF
El PDF se compara contra imágenes de referencia en `test/goldens/`. Si un cambio es intencional:
```bash
melos run test:update-goldens
```
El PDF es **determinista**: con los mismos datos siempre produce el mismo archivo y el mismo hash.

### Publicar una versión
1. Pruebas en verde.
2. Actualizar `CHANGELOG.md`.
3. `git tag vX.Y.Z && git push --tags`.
4. Avisar a la coordinadora (se refleja en `ESTADO.md`) para que App y Portal actualicen su referencia.

### Documentos de referencia
- Contrato de la API: `App-contratos/docs/contrato-api.md`
- Editor y reglas de plantillas: `App-contratos/docs/requerimientos.md` §7.4
- Plan de trabajo: `App-contratos/docs/planes/plan-dart.md`
- Estado actual: [ESTADO.md](ESTADO.md)
