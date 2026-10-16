# 06 — Decisiones y preguntas abiertas

## Decisiones tomadas (2026-10-14)

| # | Tema | Decisión |
|:-:|:---|:---|
| 1 | País y norma | **Bolivia**. Referencia oficial: *Manual de Campo SCI*, USAID/OFDA/LAC, 3.ª ed. 2013, resumido en [`SCI_fase_basica/`](./SCI_fase_basica/00_INDICE.md) |
| 2 | Formularios oficiales | Campos y reglas tomados de `SCI_fase_basica/04` (201) y `05` (207) |
| 3 | Croquis del 201 | Debe aceptar una **imagen** y, si es posible, un **mapa sobre el que se pueda dibujar** (perímetro, instalaciones, recursos, norte) |
| 4 | 201 y periodos | **Un F201 por periodo operacional** |
| 5 | Traslado en el 207 | Falta el **lugar / destino del traslado**: se agrega |
| 6 | Traspaso de mando | Exige **F201 y F207 finalizados**, con un flujo de **"finalizar todo" en un paso** antes del traspaso y advertencias claras: que nadie esté usando los formularios, y que tras cerrarlos nadie podrá cargar datos |
| 7 | Formulario 211 | **Fuera de esta etapa**: no se considera |
| 8 | Speech-to-text | Proveedor aún no definido, inicialmente en la nube. El diseño debe permitir **cambiar de proveedor sin afectar el resto** |
| 9 | Momento de la transcripción | **Al sincronizar**, en el servidor (no en el teléfono) |
| 10 | Confirmación de la IA | La IA **propone** y un usuario confirma. Resumen de acción: confirma **el mismo autor**. Dato de formulario: confirma el responsable, el **CI** o el **Líder de la Unidad Médica (LUMED)** según el formulario; si ese cargo no está asignado, el **CI** |
| 11 | Qué extrae la IA | Datos relevantes para el **201** o el **207**. Si no hay ninguno, la transcripción queda como **resumen de acción** en la bitácora |
| 12 | Idioma | Español de Bolivia. La jerga regional no se aborda por ahora |
| 13 | Móvil | **Flutter**, para Android e iOS |
| 14 | Alcance del móvil | **Lo mismo que la web**, más registro offline con sincronización posterior y la entrada por audio |
| 15 | Dispositivos | **Teléfono personal** por usuario |
| 16 | Instituciones | **Una sola institución**, sin multitenant |
| 17 | Unidades | Son recursos individuales, pero eso pertenece al 211: fuera de esta etapa |
| 18 | Privacidad de víctimas | Los usuarios `basic` solo ven víctimas de emergencias en las que participaron |
| 19 | Reportes | **Sí habrá** reportes y estadísticas operativas, **separados** del módulo de incendios |
| 20 | Infraestructura | **VPS** con HTTPS y respaldos. Audios en el servidor, pero el almacenamiento debe quedar **listo para migrar a S3** |
| 21 | Documentación | Unificar las copias duplicadas de `docs/sistema` en una sola fuente |
| 22 | Flujo de git | Commits **directo en `main`**, con Conventional Commits. Front: `github.com/DAVV422/front-sci-emergyai` |
| 23 | Versionado de `docs/` | **Repositorio propio**: `github.com/DAVV422/docs-sci`. Las copias de `docs/sistema` en los repos se reemplazan por un enlace a él |
| 24 | Periodos operacionales | **No se modelan en esta etapa.** Un **traspaso de mando cierra el periodo**: se finalizan los formularios y el nuevo CI inicia **su propio F201**. Sin entidad Periodo ni Reunión de Evaluación |
| 25 | Lugar de registro del 207 | Por defecto **ACV** o **UM**, pero **editable**: si no existe ACV ni UM, se escribe otro lugar |
| 26 | Quién registra en el 207 | Los **encargados** (Encargado del ACV y Líder de la Unidad Médica), el **Comandante del Incidente** y los roles globales **`manager`/`admin`** (para correcciones desde la central; el autor queda en el log) |
| 27 | Destino del traslado | Se crea **sobre la marcha** desde el 207 y se guarda en una **tabla de centros de salud**, que luego sirve para **sugerencias y autocompletado** |
| 28 | Reportes | Base: emergencias (atendidas, activas, reportadas, por tipo…) y participación del personal (emergencias, horas, cargos…). Propuesta completa en [08](./08_reportes_propuesta.md), pendiente de validar |
| 29 | Comando Unificado | **No por ahora** (2026-10-16): se mantiene un único CI activo por emergencia |
| 30 | Confirmación del traspaso | **Solo quien entrega** (2026-10-16): el CI saliente marca la lista de transferencia del manual; quien recibe no confirma en el sistema, pero recibe una notificación |

## Preguntas abiertas

1. **Reportes:** validar o recortar la propuesta de [08](./08_reportes_propuesta.md) y elegir los primeros a construir.
