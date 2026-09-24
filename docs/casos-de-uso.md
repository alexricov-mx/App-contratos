# Casos de uso — App-Contratos

> Versión 0.4 · 2026-09-24 · Referencias: [requerimientos.md](requerimientos.md), [contrato-api.md](contrato-api.md)

## Índice

### App de campo
| ID | Caso de uso | Offline |
|---|---|---|
| CU-A01 | Iniciar sesión | Solo tras primer login |
| CU-A02 | Registrar firma del vendedor | Sí (se sube al sincronizar) |
| CU-A03 | Sincronizar clientes y catálogo | No |
| CU-A04 | Pre-registrar cliente | Sí |
| CU-A05 | Registrar cliente completo (con identificación) | Sí |
| CU-A06 | Iniciar evento de venta | Sí |
| CU-A07 | Firmar documento del evento | Sí |
| CU-A08 | Enviar evento a la API (automático) | Encola si no hay red |
| CU-A09 | Reenviar eventos pendientes (manual) | — |
| CU-A10 | Registrar evento de entrega / modificación / cancelación | Sí |

### API
| ID | Caso de uso |
|---|---|
| CU-S01 | Recibir evento |
| CU-S02 | Sincronizar clientes |
| CU-S03 | Cambiar conexión de base de datos |

### Portal de administración
| ID | Caso de uso | Rol |
|---|---|---|
| CU-P01 | Consultar tablero | Oficina |
| CU-P02 | Completar clientes pendientes / fusionar duplicados | Oficina |
| CU-P03 | Gestionar contactos e interacciones | Oficina |
| CU-P04 | Consultar eventos de compra | Oficina |
| CU-P05 | Dar seguimiento a la entrega | Oficina |
| CU-P06 | Registrar pagos y gestionar suscripciones | Oficina |
| CU-P07 | Gestionar catálogo de productos | Admin |
| CU-P08 | Gestionar usuarios | Admin |
| CU-P09 | Verificar integridad de un documento | Oficina |
| CU-P10 | Configurar conexión a la base de datos | Superadmin |
| CU-P11 | Editar contenido de un contrato (plantilla) | Admin |

---

# App de campo

## CU-A01 · Iniciar sesión
1. El vendedor ingresa correo y contraseña.
2. La App valida contra la API y guarda la sesión cifrada.
3. Ejecuta CU-A03 (sincronización inicial).
4. Ofrece activar PIN/biometría.

**Alternos**
- **1a. Sin conexión con sesión previa:** entra con PIN/biometría en modo offline.
- **1b. Sin conexión y sin sesión previa:** el primer acceso requiere internet.

## CU-A02 · Registrar firma del vendedor
1. El vendedor dibuja su firma con el S Pen; puede repetirla.
2. Define su PIN de firma (o usa biometría).
3. La firma se guarda cifrada en SQLite y se sube en la siguiente sincronización.

## CU-A03 · Sincronizar clientes y catálogo
**Disparadores:** login, botón “Sincronizar”, al abrir la App con conexión.

1. La App **sube** clientes nuevos/modificados (con identificaciones) → CU-S02.
2. La App **descarga** clientes cambiados en el CRM desde la última sincronización.
3. La App **descarga** productos y plantillas vigentes.
4. Guarda la marca de última sincronización y la muestra.

**Alternos**
- **1a. Cliente en conflicto:** la API resuelve por campo más reciente y devuelve el registro final.
- **Error de red a mitad:** lo no confirmado se reintenta en la próxima sincronización.

## CU-A04 · Pre-registrar cliente
1. “Nuevo rápido”: nombre + teléfono o correo.
2. Se guarda en SQLite con UUID, estado `INCOMPLETO`.
3. Se sube en la siguiente sincronización o junto con el primer evento de ese cliente.

**Alterno — 1a. Posible duplicado:** se advierte y se ofrece abrir el existente.

## CU-A05 · Registrar cliente completo
1. Abre “Nuevo cliente” o un cliente `INCOMPLETO`.
2. Elige persona física o moral; captura datos (validación de RFC, correo, teléfono).
3. Fotografía la **identificación oficial** (anverso y reverso), tipo, número y vigencia. Persona moral: representante legal y poder/acta.
4. Si todo está completo y la identificación vigente → `COMPLETO`.
5. Se guarda en SQLite (imágenes cifradas).

**Alternos**
- **3a. Identificación vencida:** se advierte; queda `INCOMPLETO`.
- **3b. No trae identificación:** queda `INCOMPLETO`; no podrá firmar.

## CU-A06 · Iniciar evento de venta
1. “Nueva venta”: selecciona cliente y producto(s).
2. Se prellenan precio, modalidad de pago, alcance y plantilla (descargados en CU-A03).
3. Ajusta precio, anticipo, fecha compromiso, funcionalidades extra/excluidas.
4. Vista previa del PDF. Estado `EN_CURSO`.

**Alternos**
- **1a. Cliente incompleto:** se abre CU-A05 antes de continuar.
- **2a. Catálogo desactualizado** (sin sincronizar hace > N días): se advierte, pero se permite continuar.
- **4a. El comprador no acepta:** el vendedor descarta el evento (`DESCARTADO`).

## CU-A07 · Firmar documento del evento
**Precondición:** evento `EN_CURSO`, cliente `COMPLETO`.

1. Modo presentación; se entrega el dispositivo y el **S Pen** al comprador.
2. El comprador recorre el documento completo.
3. Marca las casillas: alcance, limitación de responsabilidad, forma de pago, aviso de privacidad, **uso exclusivo de la firma para este documento**.
4. Confirma su nombre (prellenado desde la identificación).
5. Firma con el **S Pen** (el lienzo ignora palma y dedos); puede borrar y repetir.
6. (Opcional) Selfie sosteniendo su identificación.
7. El vendedor confirma su firma con PIN/biometría.
8. La App estampa firmas, agrega la hoja de constancia, calcula el SHA-256, liga la firma a ese hash y guarda todo en SQLite. Evento `FIRMADO` (inmutable).
9. Ofrece compartir la copia con el comprador (WhatsApp/correo).
10. Dispara CU-A08.

**Regla:** cada evento exige una firma nueva; no hay “usar firma anterior”.

**Alternos**
- **5a. Sin S Pen:** se habilita firma con dedo; queda registrado `DEDO`.
- **7a. PIN incorrecto 3 veces:** bloqueo; el evento queda `EN_CURSO`.
- **Cancelación antes del paso 8:** la firma capturada se **elimina**.

## CU-A08 · Enviar evento a la API (automático)
1. Al quedar `FIRMADO`, la App cambia a `ENVIANDO` y llama `POST /api/v1/eventos` con el evento, el cliente, el PDF, firmas, trazos y evidencias → CU-S01.
2. La API confirma recepción y hash.
3. La App marca `ENVIADO` y guarda la fecha de recepción del servidor.

**Alternos**
- **1a. Sin conexión:** queda `ERROR_ENVIO` (“sin conexión”); se reintenta automáticamente al detectar red.
- **2a. Error del servidor o tiempo agotado:** `ERROR_ENVIO` con el motivo; reintentos con espera progresiva.
- **2b. Conflicto de hash:** `ERROR_ENVIO` con alerta; **no** se reintenta automáticamente, requiere revisión de oficina.

## CU-A09 · Reenviar eventos pendientes (manual)
1. El vendedor abre “Eventos pendientes de envío” (el indicador de la pantalla principal muestra cuántos hay).
2. Ve cada evento con su estado, intentos y último error.
3. Pulsa “Enviar” en uno o “Enviar todos”.
4. Se ejecuta CU-A08 para cada uno.

## CU-A10 · Evento de entrega / modificación / cancelación
1. Desde una venta existente elige “Registrar entrega”, “Modificar” o “Cancelar”.
2. Se genera el documento correspondiente (acta de entrega-recepción, convenio modificatorio o de terminación) ligado a la venta.
3. Se firma con CU-A07 (**firma nueva**) y se envía con CU-A08.
4. En el servidor, un evento `ENTREGA` actualiza el seguimiento a `ENTREGADO`; si es online, activa la suscripción.

---

# API

## CU-S01 · Recibir evento
1. Autentica al vendedor.
2. Valida el esquema del JSON y los archivos (tipos y tamaños).
3. Recalcula el SHA-256 del PDF y lo compara con el declarado.
4. Busca el evento por UUID:
   - no existe → continúa;
   - existe con el mismo hash → responde éxito (idempotente) y termina;
   - existe con otro hash → responde **409**, registra alerta y termina.
5. En una transacción: alta/actualización del cliente e identificación, alta del evento, productos, evidencias y archivos; registra fecha de recepción del servidor.
6. Efectos: `VENTA` → crea seguimiento `RECIBIDO` (+ suscripción pendiente si es online). `ENTREGA` → seguimiento `ENTREGADO`. `CANCELACION` → venta `CANCELADO`.
7. Responde con folio, hash confirmado y fecha de recepción.

**Alterno — Modo mantenimiento (CU-S03):** responde 503; la App lo trata como error temporal y reintenta.

## CU-S02 · Sincronizar clientes
1. **Push:** recibe lote de clientes con `updated_at` y `version`; aplica por campo el valor más reciente; registra conflictos; devuelve los registros finales.
2. **Pull:** devuelve clientes (y catálogo, en su endpoint) modificados desde la marca `desde` del dispositivo, con paginación.

## CU-S03 · Cambiar conexión de base de datos
Se invoca desde CU-P10. Ver pasos en [requerimientos.md §6.4](requerimientos.md).

---

# Portal de administración

## CU-P01 · Consultar tablero
Muestra: clientes por completar, eventos recibidos hoy, entregas atrasadas, suscripciones por vencer, pagos atrasados y vendedores con eventos sin enviar.

## CU-P02 · Completar clientes pendientes / fusionar duplicados
1. Abre “Clientes por completar”: lista con campos faltantes, vendedor y dispositivo de origen.
2. Completa datos (incluida la identificación si el cliente la envía a la oficina).
3. Al guardar, los cambios llegan a las Apps en su siguiente sincronización (CU-A03).

**Alterno — Fusión:** elige el registro principal; eventos, contactos e interacciones se reasignan; el duplicado queda con baja lógica.

## CU-P03 · Gestionar contactos e interacciones
1. En la ficha del cliente agrega contactos (nombre, puesto, teléfono, correo, rol).
2. Registra interacciones (llamada, visita, WhatsApp, correo) con nota y fecha de próximo seguimiento.
3. Los seguimientos vencidos aparecen en el tablero.

## CU-P04 · Consultar eventos de compra
1. Filtra por fecha, tipo, cliente, vendedor o estado.
2. Abre el detalle: datos, productos, PDF, constancia, evidencias (identificación y firmas con acceso auditado) y eventos ligados.

## CU-P05 · Dar seguimiento a la entrega
1. Desde una venta, asigna responsable y fecha compromiso.
2. Avanza el estado: `EN_PROCESO` → `LISTO_PARA_ENTREGA`; gestiona tareas/checklist y notas.
3. Al recibir el evento `ENTREGA` firmado desde la App, el estado pasa a `ENTREGADO` automáticamente.
4. Cierra la venta (`CERRADO`) cuando se liquidó y se entregó.

**Alterno — 3a. Entrega sin acta firmada:** se permite marcar `ENTREGADO` manualmente con justificación obligatoria (queda en auditoría).

## CU-P06 · Registrar pagos y gestionar suscripciones
1. Registra pago (fecha, monto, método, comprobante) ligado a una venta o suscripción.
2. Consulta suscripciones y cambia su estado (suspender por falta de pago, reactivar, cancelar), con auditoría.

## CU-P07 · Gestionar catálogo
1. Productos: tipo OFFLINE/ONLINE (el sistema fuerza la modalidad de cobro), funcionalidades, precio, plantilla.
2. Los cambios llegan a las Apps en la siguiente sincronización.

## CU-P11 · Editar contenido de un contrato (plantilla)
**Actor:** Administrador · Referencia: [requerimientos.md §7.4](requerimientos.md).

1. Abre “Plantillas” y elige una (p. ej. *Contrato SaaS*) o crea una nueva indicando el tipo de documento.
2. El editor muestra el borrador; al final aparece, en gris y sin poder editarse, el aviso de la **sección de nombres y firmas** que el sistema agrega automáticamente.
3. Edita el texto con el formato permitido (títulos, negritas, listas, etc.).
4. Donde debe ir un dato, inserta una **variable** desde el panel lateral: `{{comprador.nombre}}`, `{{vendedor.nombre}}`, `{{comprador.representante}}`, `{{precio.total}}`, etc. Se ve como una etiqueta de color.
5. Para texto que solo aplica a algunos casos, envuelve el párrafo en un bloque condicional (p. ej. *solo si el producto es online*).
6. Revisa la **vista previa en PDF** con distintos datos de ejemplo (persona física/moral, offline/online). La vista previa termina con la sección de firmas (líneas vacías con los nombres de vendedor y comprador) y la hoja de constancia.
7. El borrador se guarda automáticamente.
8. Pulsa **Publicar** y escribe una nota de cambio. El sistema valida la plantilla y crea una **versión nueva** inmutable.
9. Las Apps reciben la versión publicada en su próxima sincronización; los eventos nuevos la usan.

**Alternos**
- **8a. Plantilla inválida** (variable desconocida, bloque sin cerrar): no se publica; se marcan los errores en el texto.
- **8b. Otro administrador guardó antes:** se avisa del conflicto de versión y se ofrece recargar.
- **Volver a una versión anterior:** en el historial, “Duplicar como borrador” y publicar de nuevo.
- **Retirar plantilla:** deja de ofrecerse para eventos nuevos; los documentos ya firmados no cambian.

**Regla:** un documento firmado conserva siempre la versión de plantilla con que se firmó; editar la plantilla nunca modifica documentos existentes.

## CU-P08 · Gestionar usuarios
Alta, baja, rol (`VENDEDOR`, `OFICINA`, `ADMIN`, `SUPERADMIN`), restablecer contraseña, cerrar sesiones de un dispositivo perdido.

## CU-P09 · Verificar integridad de un documento
1. Sube un PDF o captura un folio.
2. El sistema compara el hash y muestra “Íntegro” con su constancia, o “No coincide / no registrado”.

## CU-P10 · Configurar conexión a la base de datos
**Actor:** Superadmin (reautenticación + 2FA).

1. Abre “Infraestructura → Base de datos”; ve la conexión actual (sin contraseña), versión de esquema y estado.
2. Captura la nueva conexión: host, puerto, base, usuario, contraseña, modo SSL, certificado CA.
3. Pulsa **Probar conexión** → la API intenta conectarse y reporta resultado y versión de PostgreSQL.
4. La API revisa el esquema del destino: vacío (ofrece aplicar migraciones), misma versión, o incompatible (bloquea).
5. El superadmin elige:
   - **Solo cambiar** (el destino ya tiene los datos, p. ej. restaurados manualmente), o
   - **Migrar y cambiar** (la API activa mantenimiento y ejecuta el proceso `pg_dump`/`pg_restore` con progreso visible).
6. La API verifica conteos de tablas principales, cambia la conexión activa en caliente y sale de mantenimiento.
7. Se guarda la configuración anterior para **revertir** y todo queda en auditoría.

**Alternos**
- **3a. Falla la prueba:** muestra el error (red, credenciales, SSL); no cambia nada.
- **6a. Falla la verificación:** se mantiene la conexión anterior, se sale de mantenimiento y se reporta.
- **Revertir:** botón que restaura la conexión anterior siguiendo los mismos pasos 3–6.
