# Estructura base de los contratos

> Versión 0.4 · 2026-09-24
>
> **Aviso:** este documento es una guía de estructura para construir las plantillas del sistema, no asesoría legal. Antes de usar los contratos con clientes reales deben ser revisados y ajustados por un abogado en la jurisdicción donde se venderá (se asume México).

## 1. Dos plantillas base

| Plantilla | Se usa para | Modalidad de cobro |
|---|---|---|
| **A. Contrato de desarrollo y licencia de uso de software** | Apps **offline** | Pago único |
| **B. Contrato de prestación de servicios de software por suscripción (SaaS)** | Apps **online** | Mensual o anual, obligatoria |

Ambas comparten la mayoría de cláusulas; cambian las marcadas con **[A]** o **[B]**. En el sistema, las diferencias se resuelven con bloques condicionales (`{{#si producto.online}}`) y los nombres del comprador y del vendedor se insertan con variables (`{{comprador.nombre}}`, `{{vendedor.nombre}}`). El texto se edita desde el Portal (ver [requerimientos.md §7.4](requerimientos.md)).

### Documentos derivados (uno por evento, cada uno con firma nueva del comprador)

| Evento | Documento | Contenido mínimo |
|---|---|---|
| `ENTREGA` | **Acta de entrega-recepción** | Referencia al contrato (folio + hash), lo entregado contra el Anexo A, pendientes si los hay, fecha de inicio del plazo de aceptación/garantía y, si es online, fecha de inicio de la suscripción |
| `MODIFICACION` | **Convenio modificatorio** | Referencia al contrato, cláusulas o anexos que cambian, nuevo precio/plazo; lo no modificado sigue vigente |
| `CANCELACION` | **Convenio de terminación** | Referencia al contrato, fecha de terminación, saldos pendientes o finiquito, entrega/exportación de datos, subsistencia de confidencialidad y propiedad intelectual |

## 2. Cláusulas y qué protegen

### Declaraciones
Identidad de las partes (nombre/razón social, RFC, domicilio, representante legal y documento con que acredita facultades), y que ambas tienen capacidad para contratar.

### 1. Objeto
Qué se entrega: nombre del producto, plataformas (web, Android, iOS) y remisión al **Anexo A – Alcance**.

### 2. Alcance y exclusiones — *delimita el servicio*
- El servicio comprende **únicamente** las funcionalidades listadas en el Anexo A.
- Todo lo no listado (nuevas funciones, integraciones con terceros, migración de datos, capacitación adicional, cambios de diseño) se cotiza por separado mediante convenio.
- El cliente declara haber revisado el Anexo A y que satisface sus necesidades.

### 3. Entrega y aceptación — *da certeza de que se cumplió*
- Fecha o plazo de entrega.
- El cliente cuenta con **N días naturales** (p. ej. 10) para reportar por escrito defectos contra el Anexo A.
- Si no hay reporte en ese plazo o el cliente comienza a usar la aplicación en operación real, se considera **aceptada**.
- Los defectos reportados a tiempo se corrigen sin costo; no se consideran defectos las solicitudes fuera del alcance.

### 4. Precio y forma de pago
- **[A]** Pago único de $X + IVA. Puede pactarse anticipo y saldo contra entrega. La licencia de uso surte efectos al liquidar el total.
- **[B]** Cuota **mensual/anual** de $X + IVA, pagadera por adelantado el día de corte. **Renovación automática** por periodos iguales salvo aviso de cancelación con N días de anticipación. Ajuste de precio anual con aviso previo.
- **[B]** Falta de pago: tras N días de atraso, el proveedor puede **suspender el servicio** sin responsabilidad; tras N días adicionales, dar por terminado el contrato.
- **[B]** Los periodos pagados no son reembolsables salvo incumplimiento del proveedor.

### 5. Propiedad intelectual y licencia
- El proveedor conserva la titularidad del código fuente, diseño y componentes reutilizables (salvo que se pacte cesión expresa y con su precio).
- **[A]** Se otorga licencia de uso **no exclusiva, intransferible, perpetua**, para el número de instalaciones/usuarios indicado.
- **[B]** Se otorga derecho de acceso y uso **mientras la suscripción esté vigente y pagada**.
- Prohibido revender, sublicenciar, descompilar o copiar.
- Los **datos que capture el cliente son del cliente**.

### 6. Responsabilidad sobre la información — *te protege de la calidad de los datos*
- El cliente es el **único responsable** del contenido, exactitud, veracidad, integridad y legalidad de la información que capture, cargue o genere con la aplicación, y de las decisiones que tome con base en ella.
- El proveedor no valida ni revisa dicha información.
- El cliente garantiza contar con el consentimiento/derecho para tratar datos personales de terceros que capture.

### 7. Uso permitido y uso indebido — *te protege del uso diferente*
- La aplicación se destina exclusivamente al propósito descrito en el Anexo A.
- Cualquier uso distinto, ilícito, modificación no autorizada, integración por terceros o uso fuera de los requisitos técnicos indicados es **responsabilidad exclusiva del cliente** y libera al proveedor de garantía y soporte sobre lo afectado.
- **[B]** El proveedor puede suspender cuentas usadas para fines ilícitos.

### 8. Garantía limitada y soporte
- **[A]** Garantía de N días posteriores a la aceptación, limitada a corregir defectos contra el Anexo A. Actualizaciones y soporte posteriores: póliza opcional.
- **[B]** Soporte incluido mientras la suscripción esté vigente, en el horario y canales indicados. Objetivo de disponibilidad (p. ej. 99%) **sin** que constituya garantía absoluta; exclusiones: mantenimiento programado, fallas de internet del cliente, proveedores de nube, fuerza mayor.
- No se garantiza compatibilidad con versiones futuras de sistemas operativos o navegadores no existentes al momento de la entrega.

### 9. Limitación de responsabilidad
- El proveedor no responde por daños indirectos, lucro cesante, pérdida de oportunidades o de información derivada de uso indebido, falta de respaldos del cliente **[A]**, o causas ajenas.
- La responsabilidad total del proveedor se limita a lo efectivamente pagado por el cliente (**[A]** el precio; **[B]** los últimos 12 meses).
- ⚠️ En México no es válido excluir responsabilidad por **dolo** (Código Civil Federal). Además, aunque tus clientes son negocios, la Ley Federal de Protección al Consumidor da ciertas protecciones a **micro y pequeños negocios**, y ante Profeco una cláusula desproporcionada se puede anular. Conviene que el abogado ajuste el tope de responsabilidad a algo razonable.

### 10. Datos personales y confidencialidad
- Cada parte guarda confidencialidad de la información de la otra.
- **[B]** Respecto a los datos que el cliente aloja en la plataforma, el cliente actúa como **responsable** y el proveedor como **encargado** del tratamiento; el proveedor solo los trata para prestar el servicio.
- Aviso de privacidad del proveedor respecto a los datos del cliente como comprador.

### 11. Respaldos y terminación
- **[A]** El cliente es responsable de respaldar la información de su instalación.
- **[B]** El proveedor realiza respaldos con la periodicidad indicada. Al terminar el contrato, el cliente tiene N días para solicitar la **exportación de sus datos**; después pueden eliminarse.
- Causas de terminación anticipada y aviso previo.

### 12. Caso fortuito o fuerza mayor

### 13. Modificaciones
Solo por escrito mediante convenio modificatorio firmado por ambas partes (el sistema lo genera como documento vinculado).

### 14. Consentimiento y firma electrónica
- Las partes aceptan celebrar el contrato por **medios electrónicos** y reconocen la validez de las firmas autógrafas digitalizadas y de la constancia de firma anexa (fecha, hora, ubicación, dispositivo, identificación presentada y huella digital SHA-256).
- **Uso exclusivo de la firma:** la firma del cliente se otorga **únicamente para este documento**, identificado por su folio y huella digital. El proveedor se obliga a no utilizarla, copiarla ni reproducirla en ningún otro documento; cualquier acto posterior (modificaciones, entregas, cancelaciones) requerirá una nueva firma del cliente.
- Fundamento de referencia en México: Código de Comercio (mensajes de datos y firma electrónica, arts. 89 y siguientes) y Código Civil Federal (consentimiento por medios electrónicos).

### 15. Jurisdicción y ley aplicable
Leyes y tribunales de la ciudad que se elija; renuncia al fuero por domicilio presente o futuro.

### Anexos
- **Anexo A – Alcance:** funcionalidades incluidas y exclusiones (variable `{{anexo.alcance}}`, generada desde el catálogo).
- **Anexo B – Precio y calendario de pagos** (variable `{{anexo.pagos}}`).

### Sección de nombres y firmas (fija, la agrega el sistema)
No forma parte del texto editable de la plantilla. El generador de PDF la pone **siempre al final del contenido**, después de los anexos, para que la firma abarque todo el documento:

> Leído que fue el presente documento y enteradas las partes de su contenido y alcance legal, lo firman de conformidad en {{evento.lugar}}, el {{evento.fecha}}.
>
> | EL VENDEDOR | EL COMPRADOR |
> |---|---|
> | *(firma)* | *(firma)* |
> | ______________________ | ______________________ |
> | {{vendedor.nombre}} | {{comprador.nombre}} |
> | Representada por {{vendedor.representante}} | *(si es persona moral)* Representada por {{comprador.representante}} |

Opcionalmente, la plantilla puede activar la **rúbrica en cada página** (firma pequeña del comprador del mismo evento en el margen).

Después va la **hoja de constancia de firma electrónica** generada por el sistema.

## 3. Qué da fuerza probatoria al documento (y cómo lo cubre el sistema)

| Elemento | Cómo lo cubre la app | Tareas |
|---|---|---|
| Identificación del firmante | Identificación oficial obligatoria para registro completo, selfie opcional, OTP | P1.6, P2.5 · OTP después de v1 |
| Autoría de la firma | Firma con S Pen + trazo vectorial (presión y ritmo) | P0.2–P0.4 |
| Firma no reutilizable | Firma ligada al hash de un solo documento + cláusula de uso exclusivo | P2.7 |
| Consentimiento explícito | Lectura completa obligatoria + casillas de aceptación | P2.4 |
| Integridad (que no se alteró) | Hash SHA-256 calculado en la App y **verificado de nuevo por la API** al recibir el evento; verificador en el Portal | D2.9, A3.4, W2.6 |
| Fecha cierta | Hora del dispositivo + hora de recepción en el servidor | P2.6, A3.4 |
| Conservación | PDF + bitácora en PostgreSQL/almacenamiento del servidor, respaldos fuera del VPS, 10 años | A1.9, A5.3 |
| Máxima fuerza (sello de tiempo de tercero) | Constancia NOM-151 de un Prestador de Servicios de Certificación | Después de v1 |
| Firma del vendedor consciente | PIN/biometría en cada contrato, no estampado automático | P2.5 |

**Recomendación práctica:** la firma autógrafa digitalizada es una firma electrónica **simple**; es válida y útil como prueba, pero en caso de disputa la otra parte puede cuestionarla. La combinación de evidencias (identificación + OTP + hash + NOM-151) es lo que la vuelve difícil de refutar.
