# Entorno local de desarrollo

> Versión 0.2 · 2026-09-24 · Aplica a todos los repositorios. Cada sesión debe construir su parte para que funcione exactamente así.

## 1. Cómo se conecta todo en tu PC

```
┌──────────────────────────────── PC Windows ─────────────────────────────────┐
│                                                                             │
│  Podman (máquina WSL)                                                       │
│   └─ contratos-postgres  (PostgreSQL 17)  localhost:5432                    │
│                                   ▲                                         │
│                                   │                                         │
│  dotnet run  ── API ──────────────┘  http://localhost:5080                  │
│                  ▲    ▲                  /scalar  (documentación)           │
│                  │    │                  /api/v1/health                     │
│                  │    │                                                     │
│  flutter run -d chrome ── Portal ──┘  http://localhost:5090                 │
│                  │                                                          │
│                  │  adb reverse tcp:5080 tcp:5080                           │
└──────────────────┼──────────────────────────────────────────────────────────┘
                   │ USB
          Samsung + S Pen ── App (flavor dev) → http://localhost:5080/api/v1
```

## 2. Puertos y valores fijos

| Componente | Dirección | Notas |
|---|---|---|
| PostgreSQL | `localhost:5432` · BD `contratos` · usuario `contratos` · contraseña `contratos_dev` | Solo desarrollo. Contenedor `contratos-postgres`, volumen `contratos-pg-dev`. Definido en `contratos-api/deploy/docker-compose.dev.yml` (**ya existe**) |
| API | `http://localhost:5080` | Escucha en `0.0.0.0:5080` en `Development` |
| Documentación API | `http://localhost:5080/scalar` | |
| Portal | `http://localhost:5090` | `flutter run -d chrome --web-port 5090` |
| App | Samsung por USB | `adb reverse` hace que `localhost:5080` del teléfono llegue a la PC |
| Archivos de la API | `contratos-api/.local/archivos` | En `.gitignore` |
| Config de BD cifrada | `contratos-api/.local/config` | En `.gitignore` |

### Usuarios semilla (solo `Development`)
| Correo | Contraseña | Rol |
|---|---|---|
| `superadmin@local.test` | `Dev12345!` | SUPERADMIN (2FA desactivado en desarrollo) |
| `admin@local.test` | `Dev12345!` | ADMIN |
| `oficina@local.test` | `Dev12345!` | OFICINA |
| `vendedor@local.test` | `Dev12345!` | VENDEDOR |

## 3. Requisitos por repositorio (quién construye qué)

| Repo | Debe incluir |
|---|---|
| `contratos-api` | Usar el `deploy/docker-compose.dev.yml` existente (PostgreSQL); perfil `Development` con: URL `http://0.0.0.0:5080`, cadena de conexión `Host=localhost;Port=5432;Database=contratos;Username=contratos;Password=contratos_dev`, **CORS** permitiendo `http://localhost:5090`, migraciones al inicio, semilla con los usuarios de §2, rutas de `.local/`. Script `scripts/dev.ps1` que levanta PostgreSQL y corre la API |
| `contratos-app` | Flavor `dev` con `API_URL=http://localhost:5080/api/v1`; **tráfico HTTP sin cifrar permitido solo en `dev` y solo para `localhost`** (`network_security_config` del flavor); script `scripts/dev.ps1` que ejecuta `adb reverse tcp:5080 tcp:5080` y `flutter run --flavor dev` |
| `contratos-portal` | `--dart-define=API_URL=http://localhost:5080/api/v1` por defecto en desarrollo; puerto fijo 5090; script `scripts/dev.ps1` |
| `contratos-dart` | Nada que levantar. Ver §5 para probar cambios sin publicar tag |

## 4. Levantar todo

```powershell
# 1. Base de datos + API
cd C:\Apps\contratos-api
.\scripts\dev.ps1                 # podman machine start (si hace falta) + podman compose -f deploy/docker-compose.dev.yml up -d + dotnet run

# 2. Portal
cd C:\Apps\contratos-portal
.\scripts\dev.ps1                 # flutter run -d chrome --web-port 5090 --dart-define=API_URL=...

# 3. App en el Samsung (conectado por USB, depuración USB activada)
cd C:\Apps\contratos-app
.\scripts\dev.ps1                 # adb reverse tcp:5080 tcp:5080  +  flutter run --flavor dev
```

Sin API (solo interfaz): App con `--flavor mock`, Portal con `--dart-define=API=mock`.

## 5. Probar cambios de `contratos-dart` sin publicar tag

App y Portal referencian `contratos-dart` por **tag**. Para probar en local un cambio aún no publicado, crear en App o Portal un archivo **`pubspec_overrides.yaml`** (está en `.gitignore`, nunca se sube):

```yaml
dependency_overrides:
  contratos_modelos:
    path: ../contratos-dart/packages/contratos_modelos
  contratos_pdf:
    path: ../contratos-dart/packages/contratos_pdf
```
Al terminar, borrarlo y actualizar el tag en `pubspec.yaml` como cambio propio.

## 6. Preparación de la PC (una sola vez)

1. **Podman** (ya instalado, v5.8) con la máquina `podman-machine-default` (WSL). No arranca sola con Windows:
   ```powershell
   podman machine start
   ```
   `podman compose` usa `docker-compose.exe` como proveedor; todos los comandos `docker compose …` de la documentación funcionan como `podman compose …`.
2. **Pruebas de la API con Testcontainers**: Podman expone la API compatible con Docker en la tubería por defecto de Windows, así que no hace falta `DOCKER_HOST`. Si el contenedor auxiliar *Ryuk* falla al arrancar, definir `TESTCONTAINERS_RYUK_DISABLED=true` (variable de usuario). Usar imágenes con nombre completo (`docker.io/library/postgres:17-alpine`) porque Podman no resuelve nombres cortos sin preguntar.
3. **`adb` en el PATH**: agregar `C:\dev\Android\Sdk\platform-tools` a la variable `Path` del usuario.
4. **melos**: `dart pub global activate melos` (y agregar `%LOCALAPPDATA%\Pub\Cache\bin` al `Path` si lo pide).
5. **Samsung**: Ajustes → Acerca del teléfono → Información de software → tocar 7 veces “Número de compilación” → Opciones de desarrollador → Depuración USB. Conectar y aceptar la huella de la PC.

Comprobación:
```powershell
podman machine start
podman compose -f C:\Apps\contratos-api\deploy\docker-compose.dev.yml up -d
$env:PGPASSWORD='contratos_dev'; psql -h localhost -U contratos -d contratos -c "select version();"
adb devices
flutter devices        # debe aparecer el Samsung
```
