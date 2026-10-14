# 08 — Reportes y estadísticas operativas (propuesta)

Propuesta para validar con el responsable del proyecto. Los reportes son **independientes del módulo de incendios** (decisión 19).

Cada indicador lleva una marca de disponibilidad de datos:
- ✅ se puede calcular con lo que ya se registra;
- 🟡 se puede, pero con precisión limitada;
- ❌ requiere registrar un dato nuevo (ver §5).

## 1. Emergencias — panorama operativo

Para la jefatura y para la planificación de turnos y bases.

| Indicador | Para qué sirve | Datos |
|:---|:---|:-:|
| Emergencias **reportadas, activas, finalizadas y canceladas** por periodo (día, semana, mes, año) | Volumen de trabajo y tendencia | ✅ |
| Emergencias por **tipo** (incendio estructural, forestal, rescate, accidente de tránsito, materiales peligrosos…) | Qué tipo de capacidad y entrenamiento priorizar | 🟡 hoy `type` es texto libre: conviene un catálogo de tipos |
| **Tasa de cancelación** y motivos (falsa alarma, duplicada, resuelta antes del arribo) | Calidad del despacho; detectar falsas alarmas recurrentes | 🟡 el motivo es texto libre |
| Emergencias por **hora del día y día de la semana** (mapa de calor) | Dimensionar guardias según la demanda real | ✅ |
| Emergencias por **zona** (municipio, distrito o macrodistrito) y mapa de puntos | Ubicar recursos y bases; detectar zonas críticas | 🟡 hay coordenadas, falta la zona administrativa (se deriva o se registra) |
| **Emergencias simultáneas** (pico de activas a la vez) | Saber si la capacidad alcanza en los picos | ❌ requiere las horas de inicio y cierre (§5) |
| **Duración** de las emergencias (de activación a cierre) por tipo | Planificar relevos; detectar incidentes que se alargan | ❌ (§5) |
| **Tiempo de respuesta**: reporte → activación → primer arribo | El indicador clave de desempeño de un servicio de emergencias | ❌ (§5) |

## 2. Víctimas y atención prehospitalaria (SCI-207)

Para la jefatura médica y la coordinación con salud.

| Indicador | Para qué sirve | Datos |
|:---|:---|:-:|
| Víctimas por **clasificación de triage** (rojo, amarillo, verde, negro) por periodo y tipo de emergencia | Severidad de los incidentes atendidos | ✅ |
| Víctimas por **sexo y rango de edad** | Perfil de la población afectada (prevención) | ✅ |
| **Traslados por centro de salud de destino** y por **medio** | Carga sobre cada hospital; coordinación con la red de salud | ❌ requiere el destino y el medio (decisión 27) |
| **Evolución del triage** (cuántos empeoran o mejoran entre evaluaciones) | Calidad de la atención en escena | ✅ |
| **Fallecidos** por tipo de emergencia | Indicador de gravedad; prevención | ✅ |
| Emergencias con víctimas **sin 207 finalizado** | Control de la documentación | ✅ |

## 3. Personal y participación

Para recursos humanos, guardias y reconocimiento.

| Indicador | Para qué sirve | Datos |
|:---|:---|:-:|
| **Emergencias por persona** en un periodo | Participación y reconocimiento | ✅ |
| **Horas de participación** por persona (suma de su tiempo asignado) | Carga de trabajo, horas extra, compensaciones | ❌ requiere la hora de liberación (§5) |
| **Cargos SCI ocupados** por persona (cuántas veces fue CI, LUMED, etc.) | Experiencia real en mando; plan de capacitación | ✅ |
| **Horas continuas** en una misma emergencia | Alerta de fatiga: seguridad del personal | ❌ (§5) |
| **Personal en servicio** vs. fuera de servicio por fecha | Disponibilidad real frente a la demanda | 🟡 hoy solo se conoce el estado actual, no el histórico |
| **Traspasos de mando** por emergencia y su duración media en el mando | Rotación de comandantes en incidentes largos | 🟡 está en la bitácora como texto |

## 4. Recursos e inventario

| Indicador | Para qué sirve | Datos |
|:---|:---|:-:|
| **Equipos más despachados** por tipo de emergencia | Qué comprar o reponer primero | ✅ |
| Unidades **en uso ahora** y **no devueltas** tras el cierre de la emergencia | Pérdidas y control patrimonial | ✅ |
| **Stock crítico** (disponible bajo un mínimo) | Reposición preventiva | ❌ requiere un "mínimo" por equipo |
| **Tiempo de uso** de cada equipo (despacho → devolución) | Desgaste y mantenimiento | 🟡 la devolución no guarda su propia fecha |

## 5. Calidad del proceso SCI

Para la mejora continua y las auditorías.

| Indicador | Datos |
|:---|:-:|
| % de emergencias con **F201 finalizado** | ✅ |
| % de emergencias con víctimas que tienen **F207 finalizado** | ✅ |
| **Entradas de bitácora** por emergencia (manuales vs. automáticas; con voz en la Etapa 2) | ✅ |
| Tiempo desde la activación hasta el **primer F201** | ❌ (§6) |
| Emergencias cerradas **con recursos sin devolver** | ✅ |

## 6. Reportes públicos

Para un boletín mensual o anual, transparencia y prensa. **Siempre agregados y sin datos personales.** El manual establece que la información que sale del incidente la autoriza el CI; para los reportes institucionales, la jefatura.

Recomendados:
- Emergencias **atendidas por tipo y por mes**, con comparación interanual.
- Emergencias por **municipio o zona**: mapa agregado, sin direcciones exactas.
- **Personas atendidas** y **trasladadas**, totales y por clasificación de triage.
- **Tiempo promedio de respuesta**, cuando exista el dato.
- **Personal y horas** movilizados en el año.
- **Equipos desplegados** (cantidad total).

Reglas de privacidad para los reportes públicos:
- Ningún nombre, documento, teléfono ni coordenada exacta de víctimas.
- No publicar cifras muy pequeñas que permitan identificar a alguien (por ejemplo "1 fallecido en tal comunidad en tal fecha"). Agregar por periodo o zona más amplios, o mostrar "menos de 5".
- Publicación revisada por la jefatura antes de difundirse.

## 7. Datos que hay que empezar a registrar

Los reportes de tiempos y horas son de los más valiosos para tomar decisiones, y hoy **no se pueden calcular** porque esos momentos no se guardan. Conviene registrarlos **cuanto antes**: los datos históricos no se pueden reconstruir después.

| Dato nuevo | Dónde | Habilita |
|:---|:---|:---|
| **Marca de tiempo de cada cambio de estado** (reportada, activada, finalizada, cancelada) | Emergencia | Duración, simultaneidad, tiempos de respuesta |
| **Hora de primer arribo** a la escena | Emergencia o primera asignación | Tiempo de respuesta |
| **Hora de liberación** del personal (`releasedAt`) | Asignación (`attend`) | Horas de participación, fatiga |
| **Catálogo de tipos de emergencia** | Emergencia | Estadística confiable por tipo |
| **Zona administrativa** (municipio o distrito) | Emergencia | Reportes territoriales y públicos |
| **Fecha de cada devolución** de recursos | Recurso | Tiempo de uso de equipos |
| **Historial de disponibilidad** del personal | Usuario | Disponibilidad histórica |
| Fechas como `timestamp` en lugar de texto | Emergencia, acciones | Consultas por rango y zona horaria correctas |

## 8. Orden sugerido

1. **Primero, registrar los datos de §7.** No cuesta mucho y evita perder meses de historia.
2. **Tablero de emergencias** (§1) y **víctimas por triage** (§2): con los datos actuales ya aportan valor.
3. **Participación del personal** (§3), cuando exista la hora de liberación.
4. **Exportación** de cada reporte a Excel/CSV y PDF.
5. **Boletín público** (§6).
