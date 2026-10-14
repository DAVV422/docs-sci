# 02 — Alcance y etapas

## Etapa 1 — Gestión de la emergencia en la web (en curso)

**Objetivo:** que la central pueda gestionar una emergencia completa siguiendo el SCI, con los Formularios **201** y **207** como ejes.

Incluye:
- **Usuarios, roles y autenticación.** Activación de cuentas, perfil y estado de guardia.
- **Emergencias.** Alta georreferenciada con tres puntos (incidente, Puesto de Comando y Área de Espera), máquina de estados (pendiente → activa → finalizada / cancelada) y bloqueo de edición al cerrar.
- **Evaluación inicial** de riesgos.
- **Organigrama SCI.** 35 cargos en 5 niveles, asignación de personal, asignación y traspaso formal de mando del CI.
- **Formulario 201 — Resumen del incidente.** Con foto fija (snapshot) del organigrama, finalización irreversible y hoja imprimible.
- **Formulario 207 — Registro de víctimas.** Planillas, triage START/SALT, re-evaluaciones, historial, finalización y hoja imprimible.
- **Inventario y recursos.** Catálogo de equipos, despacho a la emergencia y devolución con control de stock.
- **Bitácora.** Entradas manuales y automáticas, y reproducción de notas de voz.
- **Tiempo real** entre consolas web (WebSocket).
- **Pruebas automatizadas:** unitarias y E2E (Playwright).

Requisitos agregados el 2026-10-14 (pendientes de implementar):
- **Un F201 por mando**: el traspaso cierra el periodo y el nuevo CI abre su 201. No se modelan periodos operacionales en esta etapa. Croquis: imagen adjunta y, si es posible, mapa dibujable.
- **Destino del traslado** en el 207, con tabla de centros de salud y autocompletado. Lugar de registro ACV/UM editable. Registran el EACV, el LUMED y el CI.
- **Traspaso de mando** que exige F201 y F207 finalizados, con un flujo de "finalizar todo" y advertencias.
- **Privacidad de víctimas:** los `basic` solo ven las emergencias en las que participaron.
- **Almacenamiento de archivos abstraído**, listo para pasar del disco a S3.

Fuera de la Etapa 1:
- **Grabación** de audio: es exclusiva del móvil; la web solo reproduce.
- Procesamiento de IA de los audios.
- Módulo de predicción de incendios.

## Etapa 2 — App móvil de terreno

**Objetivo:** captura en terreno por voz, offline-first.

Incluye:
- App móvil **Flutter** para Android e iOS, con **la misma funcionalidad que la web**, más modo offline y entrada por voz. Un teléfono personal por usuario.
- Grabación de notas de voz y subida al backend (`POST /api/action/upload-audio/:emergencyId`, ya disponible).
- **Pipeline de IA**, que corre en el servidor **al sincronizar**:
  - speech-to-text con un proveedor en la nube **intercambiable**;
  - extracción de datos para el **201** o el **207**; si no hay datos de formulario, la transcripción queda como **resumen de acción**;
  - **la IA propone y una persona confirma**: el autor confirma su resumen de acción, y el CI o el LUMED confirman los datos de formulario (el CI si el cargo no está asignado).

  **No implementado todavía:** el backend solo tiene preparadas las columnas `transcription`, `nlpExtractedData` y `processed` en `AudioEntity`.
- Base de datos local, cola de sincronización y envío por lotes a `POST /api/sync/batch` (endpoint ya disponible, con idempotencia por `clientGeneratedId`).
- Notificaciones push FCM (backend ya integrado).
- Pendientes de backend detectados para esta etapa: la sincronización no propaga el rol del usuario (las operaciones offline omiten la validación táctica) y `form201Service.update` se invoca sin el usuario.

## Reportes y estadísticas operativas

Se harán **antes y por separado** del módulo de incendios. Los primeros reportes están por definir: ver [06](./06_preguntas_abiertas.md).

## Más adelante — Formulario 211

El registro y control de recursos (SCI-211) **no se aborda en esta etapa**. Las unidades (ambulancias, carros bomba) serán recursos individuales cuando se implemente.

## Etapa futura — Histórico y riesgo de incendios forestales y de interfaz

- Registro histórico de incendios con variables propias: superficie, vegetación, clima, causa y otras.
- Cálculo de probabilidad o riesgo por zona y rango de fechas, y visualización en mapa.
- **En pausa:** el responsable del proyecto entregará los requisitos funcionales. Hoy solo existe una entidad mínima, `DataFireEntity`, que no se debe modificar hasta entonces.

## Fuera de alcance

- Multitenant: la plataforma es para **una sola institución**.
- Formulario 211 en esta etapa, y formularios del PAI (202, 204, 205, 206, 214, 215).
- **[A confirmar]** Integración con sistemas externos de despacho (911/CAD).
