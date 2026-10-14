# 06 – Formulario SCI-211: Registro y Control de Recursos

## 1. Ficha

| Atributo | Valor |
|---|---|
| Código | SCI-211 |
| Nombre | Registro y Control de Recursos |
| Lo llena | **Registrador(es)** (hasta 3 por hoja); en el periodo inicial, quien el CI designe |
| Lugares de registro | **Áreas de Espera, Base, Helibase y Puesto de Comando** |
| Regla clave | **El registro se hace solo una vez.** El registrador debe mantener el SCI-211 **actualizado en todo momento**, porque es clave para la toma de decisiones |
| Se usa en | Informe del Incidente, **Reunión de Evaluación**, preparación de la Reunión de Planificación, dimensionamiento de la desmovilización (LUDES) |
| Distribución | Copias a la **Unidad de Recursos** y a la **Sección de Administración y Finanzas**. La información se pasa lo más rápido posible al **CI, JSO, JSP o Unidad de Recursos**. Originales a la **Unidad de Documentación** |
| Lista maestra | La **Unidad de Recursos** mantiene un listado maestro de todo el personal y equipo reportado al incidente |

## 2. Propósito

Llevar un **registro y control general de los recursos** en el incidente: desde que se solicitan, su arribo, quién los suministra, su estado y asignación, hasta su desmovilización.

## 3. Estructura del formulario

### Encabezado

| Campo | Tipo sugerido |
|---|---|
| Nombre del incidente | Referencia al incidente |
| Fecha y hora de preparación | Fecha-hora (24 h) |
| Lugar de registro | Referencia a instalación (E, B, H, PC) |

### Detalle (una fila por recurso), agrupado en bloques A–E

| Bloque | N° | Campo | Instrucción del manual | Tipo sugerido |
|---|---|---|---|---|
| **A. Solicitud de recurso** | 1 | Por quién | Nombre de quien solicita el recurso (**CI o JSO**) | Persona / posición |
| | 2 | Fecha y hora | De la solicitud (día, mes, año; 24 h) | Fecha-hora |
| | 3 | Clase | Cada usuario ingresa los nombres por clase de sus recursos | Catálogo (ej. Ambulancia, Motobomba, Cuadrilla…) |
| | 4 | Tipo | Tipificación del recurso dentro de la clase (según capacidad) | Catálogo dependiente de la clase |
| **B. Arribo real** | 5 | Fecha y hora | De arribo (24 h) | Fecha-hora |
| **C. Suministrado por** | 6 | Institución | Institución de procedencia | Catálogo de instituciones |
| | 7 | Matrícula | Número de identificación local del recurso | Texto (placa, código, matrícula de aeronave) |
| | 8 | Número de personas | Cantidad de personas que componen el recurso | Entero |
| **D. Estado de los recursos** | 9 | Disponible / No disponible / Asignado a | Indicar si está **Disponible, No disponible o Fuera de servicio**; y a qué está asignado | Enum de estado + asignación (posición/área) |
| **E. Desmovilizado** | 10 | Por quién | **CI, JSO** u otro (ej. Supervisor de División) | Persona / posición |
| | 11 | Fecha y hora | Quién autoriza, fecha y hora **efectiva** de la desmovilización | Fecha-hora |
| — | 12 | Observaciones | **Cronológicamente** todos los movimientos del recurso, estados, periodos y anotaciones. Ej.: "Ambulancia disponible, asignada al sector norte en el periodo inicial" | Bitácora de movimientos |

### Pie
- Nombre del Registrador 1
- Nombre del Registrador 2
- Nombre del Registrador 3

> **Inconsistencia del manual a resolver en el diseño:** las instrucciones del campo 9 dicen "Disponible, No disponible o Fuera de servicio", mientras que las columnas impresas del formulario son "Disponible · No disponible · Asignado a". El Anexo H (cambio de estado) usa **Asignado, Disponible, Fuera de servicio**. Se recomienda modelar el estado como un enum único y la asignación como un campo aparte (ver §5).

## 4. Estados de los recursos y su ciclo de vida

Según el Anexo H y el texto claro:

| Estado | Significado |
|---|---|
| **Asignado** | Trabajando en una tarea táctica / asignación del incidente |
| **Disponible** | En el incidente, listo para asignarse (normalmente en el Área de Espera), con disponibilidad inmediata |
| **Fuera de servicio** | No puede asignarse (razones mecánicas, de personal —"descubierta"—, descanso, etc.). Al repararse: "de vuelta en servicio, disponible" |

Estados adicionales útiles para el sistema (derivados de los bloques del formulario): **Solicitado** (bloque A sin arribo), **En ruta** (despachado, aún sin arribo), **Desmovilizado** (bloque E completo).

```
SOLICITADO ─► EN_RUTA ─► (arribo + registro único) ─► DISPONIBLE ◄──► ASIGNADO
                                                         ▲   │           │
                                                         │   ▼           ▼
                                                    FUERA_DE_SERVICIO ◄──┘
                                                         │
                         DISPONIBLE / ASIGNADO / FDS ────┴──► DESMOVILIZADO (final)
```

### Reporte de cambios de estado (Anexo H)
- Se informa: **cambio en el estado** de los recursos (asignados, disponibles, fuera de servicio) y **movimiento de recursos entre divisiones**.
- Flujo: Supervisor de División / Coordinador de Rama / JSO → **Centro de Comunicaciones** → **Unidad de Recursos**.
- **Nota:** la autoridad que aprueba el cambio de estado es **responsable de reportarlo**.

### Liberación de recursos (Anexo J)
Cuando los recursos de una FT o EI quedan disponibles:
- El Supervisor de División → Coordinador de Rama → JSO deciden:
  - **Reasignación (activa)** a otra tarea, o
  - **Reasignación** al **Área de Espera** (Disponible), o
  - **Liberación** a la Base, Campamento o según instrucciones escritas o verbales.
- En todos los casos: **avisar a la Unidad de Recursos a través de Comunicaciones**.
- La autoridad que aprueba el cambio de estado es responsable de reportarlo al CI.

## 5. Modelo de datos sugerido

```yaml
Recurso:                         # una fila del SCI-211; registro único por incidente
  id: uuid
  incidente_id: uuid
  lugar_registro_id: instalacion_id     # E, B, H o PC
  registrador_id: persona_id
  # A. Solicitud
  solicitado_por: persona_id            # CI o JSO
  fecha_hora_solicitud: datetime
  clase: catalogo_clase
  tipo: catalogo_tipo
  # B. Arribo
  fecha_hora_arribo: datetime?
  # C. Suministrado por
  institucion_id: uuid
  matricula: text
  numero_personas: int
  personal: [persona_id]?               # opcional: detalle del personal del recurso
  lider_id: persona_id?
  medio_comunicacion: text?             # el recurso simple "cuenta con comunicaciones"
  # D. Estado
  estado: SOLICITADO | EN_RUTA | DISPONIBLE | ASIGNADO | FUERA_DE_SERVICIO | DESMOVILIZADO
  asignado_a: posicion_id | agrupacion_id | instalacion_id ?   # división, grupo, FT, EI, E...
  # E. Desmovilización
  desmovilizado_por: persona_id?
  fecha_hora_desmovilizacion: datetime?
  # 12. Observaciones / bitácora
  movimientos: [ {fecha_hora, estado_anterior, estado_nuevo, asignacion_anterior, asignacion_nueva,
                  periodo_id, aprobado_por, reportado_via, nota} ]
  tarjeta_t_color: derivado de la clase (ver Anexo D)
```

## 6. Reglas de negocio derivadas

1. **Registro único:** no se permite dar de alta dos veces el mismo recurso en el mismo incidente (validar por institución + matrícula). Si llega a otro punto de registro, se registra un *movimiento*, no un alta.
2. **Solicitud:** solo el **CI o el JSO** (o quien tenga delegada la función) pueden registrar una solicitud (campo 1).
3. **Arribo:** al registrar fecha/hora de arribo el recurso pasa a **Disponible** por defecto (salvo que se marque otro estado).
4. **Asignación:** solo recursos **Disponibles** pueden asignarse. Al asignar se exige el destino (división/grupo/FT/EI/instalación).
5. **Cambio de estado:** toda transición registra quién la aprobó (responsable de reportarla), hora y periodo; genera entrada automática en Observaciones (campo 12).
6. **Desmovilización:** solo CI, JSO u otra autoridad de la línea de mando (p.ej. Supervisor de División) con autorización; requiere fecha-hora efectiva. Estado final, no reversible.
7. **Distribución automática:** todo cambio actualiza en tiempo real la **lista maestra** de la Unidad de Recursos y queda visible para Administración y Finanzas (horas de uso de equipo, costos).
8. **Equipo de Intervención:** todos sus recursos de la misma clase y tipo. **Fuerza de Tarea:** clases/tipos mezclados. Ambos con líder y comunicaciones.
9. **Antes del cierre operacional** los registradores entregan las listas a la Unidad de Recursos; el sistema puede bloquear el cierre del periodo si hay recursos sin estado actualizado.
10. **Indicadores para la Reunión de Evaluación:** totales por estado, por clase/tipo, por institución, por asignación; recursos solicitados pendientes de arribo con tiempo transcurrido; número total de personas en el incidente (insumo también para la Unidad de Alimentación).
11. El LUDES usa el SCI-211 para estimar el tamaño de la desmovilización (recursos activos por institución, clase y zona).
