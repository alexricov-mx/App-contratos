# contratos-api

API de **App-Contratos**: la única puerta de entrada a la base de datos. La usan la App de campo y el Portal de administración.

> Documentación completa de la solución: repositorio [`App-contratos`](../App-Contratos/README.md).

---

## Parte 1 · La solución en general

App-Contratos permite **vender aplicaciones de software a negocios locales** y cerrar cada venta con un **contrato firmado en sitio** con el S Pen, aunque no haya internet.

```
App de campo (Flutter + SQLite) ──► API (.NET 10) ──► PostgreSQL
                                        ▲
Portal de administración (Flutter Web) ─┘
```

| Repositorio | Papel |
|---|---|
| `App-contratos` | Requerimientos, contrato de la API, planes |
| `contratos-dart` | Modelos, cliente HTTP y generador de PDF compartidos |
| **`contratos-api`** | **Este repositorio** |
| `contratos-app` | App del vendedor en campo |
| `contratos-portal` | Portal de la oficina |

---

## Parte 2 · Este componente

### ¿Qué hace?
- **Autentica** a vendedores y personal de oficina (JWT).
- **Recibe los eventos** firmados en campo (venta, entrega, modificación, cancelación), verifica que el PDF no se haya alterado (SHA-256) y los guarda **sin duplicar** aunque lleguen dos veces.
- **Sincroniza** clientes y catálogo con la App.
- Da servicio al **Portal**: CRM, contactos, eventos, seguimiento de entrega, pagos, suscripciones, plantillas de contrato y usuarios.
- Permite **cambiar a qué base de datos se conecta** desde el Portal, sin reinstalar, para migrar después a un PostgreSQL administrado.

### ¿Cómo está organizado? — Vertical Slice Architecture
En lugar de separar el código en capas (controladores, servicios, repositorios), cada **caso de uso** tiene su propia carpeta con todo lo que necesita:

```
src/Contratos.Api/
  Features/
    Clientes/
      Registrar/          ← un caso de uso = una carpeta
        Endpoint.cs       ← ruta HTTP
        Request.cs        ← datos de entrada
        Validator.cs      ← reglas de validación
        Handler.cs        ← lógica y acceso a datos
      Listar/
      ...
    Eventos/  Sync/  Plantillas/  Infraestructura/ ...
  Common/                 ← solo lo realmente compartido (auth, errores, auditoría, archivos, conexión de BD)
  Data/                   ← DbContext y migraciones
tests/
deploy/                   ← docker-compose, Caddy, respaldos
docs/features/            ← documentación de cada feature construido
```

**¿Por qué?** Para cambiar un caso de uso solo hay que abrir una carpeta, y un cambio en “Registrar cliente” no puede romper “Recibir evento” por accidente.

### Tecnologías
.NET 10 · Minimal APIs · EF Core + Npgsql · FluentValidation · Serilog · OpenAPI/Scalar · xUnit + Testcontainers · Docker.

### Requisitos para desarrollar
- .NET 10 SDK
- Docker Desktop (para PostgreSQL local y las pruebas de integración)

### Ejecutar en local
```bash
docker compose -f deploy/docker-compose.dev.yml up -d postgres
dotnet run --project src/Contratos.Api
# Documentación interactiva: http://localhost:5080/scalar
```

### Pruebas
```bash
dotnet test
```
Las pruebas de integración levantan un PostgreSQL real en contenedor (Testcontainers).

### Desplegar en el VPS (OVH)
Ver [deploy/README.md](deploy/README.md). En resumen:
```bash
cd deploy
cp .env.example .env    # llenar valores
docker compose pull
docker compose up -d
```

### Configuración de la base de datos
La conexión se guarda **cifrada** en `/app/config/conexion-bd.json` (volumen del contenedor). Si no existe, se usa la variable `ConnectionStrings__Contratos`. Se puede cambiar desde el Portal → Infraestructura → Base de datos (solo superadministrador).

### Documentos de referencia
- Contrato de la API: `App-contratos/docs/contrato-api.md`
- Plan de trabajo: `App-contratos/docs/planes/plan-api.md`
- Estado actual: [ESTADO.md](ESTADO.md)
- Features construidos: [docs/features/](docs/features/)
