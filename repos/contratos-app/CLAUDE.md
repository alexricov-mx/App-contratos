# CLAUDE.md — contratos-app

Idioma de trabajo: **español** (código en español para el dominio: `Cliente`, `Evento`, `Plantilla`; términos técnicos estándar en inglés cuando sea lo habitual).

## Contexto
Este repositorio es parte de **App-Contratos**. La documentación central está en la carpeta hermana `../App-Contratos` (repo `App-contratos`). Leer antes de empezar cualquier feature:

1. `../App-Contratos/README.md` — visión general
2. `../App-Contratos/docs/proceso/flujo-de-trabajo.md` — **cómo se trabaja** (obligatorio)
3. `../App-Contratos/docs/planes/plan-app.md` — **tu plan de trabajo**
4. `../App-Contratos/docs/entorno-local.md` — cómo debe correr en local
5. `../App-Contratos/docs/contrato-api.md` — fuente de verdad de la comunicación entre componentes
6. `../App-Contratos/docs/requerimientos.md` y `casos-de-uso.md` — secciones indicadas en tu plan
7. `ESTADO.md` de este repo

## Reglas
- **No modificar nada en `../App-Contratos`** ni en otros repositorios. Si necesitas un cambio en la documentación central (sobre todo `contrato-api.md`), escríbelo en la sección 8 de tu `02-analisis.md`, regístralo en `ESTADO.md` como bloqueo y detén solo esa parte hasta que la sesión coordinadora lo apruebe.
- **No inventar** endpoints, campos, códigos de error ni variables de plantilla que no estén en `contrato-api.md` / `requerimientos.md`.
- Trabajar **un feature a la vez**, en la rama `feature/<ID>-<nombre-corto>`.
- Por cada feature crear `docs/features/<ID>-<nombre-corto>/` con `01-requerimiento.md`, `02-analisis.md`, `03-plan.md`, `04-pruebas.md`, `05-resumen.md` usando las plantillas de `../App-Contratos/docs/proceso/plantillas/`.
- Escribir 01, 02 y 03 **antes** de programar. Construir en el orden de `03-plan.md`.
- Cada sección del plan: código + prueba → verificación en verde → commit `<ID>.<n>: descripción` → actualizar `ESTADO.md`.
- Respetar la lista **“lo que NO se debe modificar”** del requerimiento; si es inevitable, justificarlo en el análisis.
- Antes de cerrar un feature: **suite completa** en verde (regresión), `04-pruebas.md` con resultados reales, `05-resumen.md` didáctico, README actualizado si cambió cómo se instala/usa/ejecuta.
- No dar nada por terminado sin haber ejecutado las pruebas. Si algo no se pudo probar (p. ej. requiere el dispositivo físico), decirlo explícitamente en `04-pruebas.md` y en `ESTADO.md`.

## Stack y convenciones
- Flutter, Android (Samsung + S Pen) como plataforma principal.
- `contratos_modelos` y `contratos_pdf` desde `contratos-dart` **por tag** (nunca `main`). Actualizar el tag es un cambio propio, documentado.
- **Offline-first**: la UI solo escribe en repositorios SQLite (drift + SQLCipher). Solo `SincronizacionService` y `EnvioEventosService` hablan con la API.
- Flavors: `mock` (usa `ApiMock`), `dev` (API local), `prod`.
- Lienzo de firma: en modo S Pen solo acepta `PointerDeviceKind.stylus`; guardar PNG + trazo JSON (contrato-api.md §5.2).
- Firma del comprador **de un solo uso**: nunca se guarda como elemento reutilizable del cliente; si el evento se cancela, se elimina.
- Las pruebas que requieren el dispositivo físico se documentan como **manuales** en `04-pruebas.md` con el modelo de dispositivo.

## Comandos
```
flutter pub get
flutter analyze
flutter test
flutter run --flavor mock -d <dispositivo>
flutter build apk --flavor prod --release
```

