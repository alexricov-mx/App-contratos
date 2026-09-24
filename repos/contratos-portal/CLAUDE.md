# CLAUDE.md — contratos-portal

Idioma de trabajo: **español** (código en español para el dominio: `Cliente`, `Evento`, `Plantilla`; términos técnicos estándar en inglés cuando sea lo habitual).

## Contexto
Este repositorio es parte de **App-Contratos**. La documentación central está en la carpeta hermana `../App-Contratos` (repo `App-contratos`). Leer antes de empezar cualquier feature:

1. `../App-Contratos/README.md` — visión general
2. `../App-Contratos/docs/proceso/flujo-de-trabajo.md` — **cómo se trabaja** (obligatorio)
3. `../App-Contratos/docs/planes/plan-portal.md` — **tu plan de trabajo**
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
- Flutter Web, riverpod, go_router con URLs legibles.
- `contratos_modelos` y `contratos_pdf` desde `contratos-dart` **por tag** (nunca `main`).
- `--dart-define=API=mock` usa `ApiMock`; `API_URL` apunta a la API real.
- Editor de plantillas con `flutter_quill` limitado a los formatos de `contrato-api.md` §7; variables como etiquetas no editables que se serializan `{{…}}`; la sección de firmas se muestra como bloque gris de solo lectura y **no** se guarda en el Delta.
- La vista previa usa **exclusivamente** `contratos_pdf` (mismo generador que la App).
- Token en memoria; nada sensible en `localStorage`.

## Comandos
```
flutter pub get
flutter analyze
flutter test
flutter run -d chrome --dart-define=API=mock
flutter build web --release --dart-define=API_URL=https://api.<dominio>/api/v1
docker build -t contratos-portal .
```

