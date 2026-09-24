# <ID> · <Nombre del feature> — Requerimiento

> Repo: `<repo>` · Rama: `feature/<ID>-<nombre-corto>` · Fecha: AAAA-MM-DD
> Fuente: `App-Contratos/docs/planes/plan-<repo>.md` § <ID>

## 1. Objetivo
<Una o dos frases: qué problema resuelve este feature y para quién.>

## 2. Referencias
- Requerimientos: <RF-… de requerimientos.md>
- Casos de uso: <CU-…>
- Contrato de API: <secciones de contrato-api.md que usa o implementa>

## 3. Alcance (entra)
| # | Elemento | Criterio de aceptación |
|---|---|---|
| 1 | | |

## 4. Fuera de alcance (no entra)
- <Funcionalidades que parecen relacionadas pero van en otro feature, con su ID.>

## 5. Límites: lo que NO se debe modificar
| Componente / archivo / módulo | Motivo |
|---|---|
| | <pertenece a otro feature / es contrato compartido / ya está integrado> |

## 6. Dependencias
| Necesita | Estado |
|---|---|
| <feature, tag de contratos-dart, endpoint> | Lista / Pendiente / Se usa mock |

## 7. Definición de terminado
- [ ] Todos los criterios de §3 cumplidos.
- [ ] Pruebas nuevas y suite completa en verde (04-pruebas.md).
- [ ] Nada de §5 modificado (o justificado en 02-analisis.md).
- [ ] 05-resumen.md escrito; README actualizado si aplica.
