# 10 — Estado y pendientes por etapa (2026-11-16)

Qué falta implementar, separado por etapa y por repositorio. Contrastado **contra el código**, no
contra la documentación: varios puntos que [07](./07_inspeccion.md) daba por pendientes ya están
resueltos, y aparecieron dos que no estaban anotados.

## 1. Dónde estamos

La **Etapa 1 (web) está funcionalmente cerrada** y desplegada en producción para tres instituciones.
Lo que queda de ella son detalles, no funcionalidad.

| Suite | Resultado hoy |
|:---|:---|
| Unitarias backend (Jest) | ✅ 452 / 452 |
| Unitarias front (Karma) | ✅ 628 / 628 |
| E2E de punta a punta (Playwright) | ✅ 41 / 41, en 21 escenarios |
| E2E del backend (supertest, `test/`) | ❌ **41 / 49 fallan** |

El peso de lo que falta **no está en la Etapa 1: está en la Etapa 2**, que es el núcleo del proyecto
(voz + IA en móvil) y de la que hoy solo existen los cimientos.

## 2. Etapa 1 — Web: lo que queda

Nada de esto impide operar. Son cabos sueltos.

### Backend

| # | Pendiente | Prioridad | Nota |
|:-:|:---|:-:|:---|
| B-1 | `POST /checkToken` y `/expiredToken` reciben el token en la *query string*, donde queda escrito en los registros de acceso del proxy | P2 | Era el S12 de [07](./07_inspeccion.md). **Dato nuevo: el front no los usa.** Son endpoints huérfanos, así que lo correcto es borrarlos, no arreglarlos |
| B-2 | `GET/POST/PATCH/DELETE /data-fire` con CRUD abierto a cualquier usuario autenticado | P2 | Era el S13. El módulo está en pausa esperando requisitos; mientras tanto conviene cerrar la ruta por rol o desmontar el controlador |
| B-3 | Los cargos tienen nivel y peso, pero no superior (`parentId`) | P3 | El organigrama se dibuja por niveles. Sin jerarquía explícita no se puede validar "un único superior" como pide el manual |
| B-4 | Falta un proveedor S3 para `STORAGE_SERVICE` | P2 | El contrato (`IStorageService`) y las claves ya están; solo falta la implementación con `@aws-sdk/client-s3`. Hoy todo va a disco en `/app/data` |
| B-5 | 11 vulnerabilidades moderadas propias de NestJS 10 | P3 | Se cierran migrando a Nest 11 |

### Front

Sin pendientes funcionales de Etapa 1. Las dos pantallas que [07](./07_inspeccion.md) listaba como
faltantes (editar emergencia, recuperación de contraseña) están hechas, igual que el centro de
notificaciones, la administración de cargos y la configuración de institución.

## 3. Producción y despliegue

| # | Pendiente | Prioridad | Nota |
|:-:|:---|:-:|:---|
| P-1 | **Clave de Google Maps restringida a otros dominios**: el mapa no carga en `demo`, `test` ni `fru` | **P0** | Es lo único que hoy se ve roto para un usuario. Se arregla en Google Cloud agregando los tres subdominios como referentes; no hace falta recompilar |
| P-2 | **Copias de seguridad sin programar** | **P0** | `scripts/backup.sh` y `restore.sh` están probados, pero no hay cron en el VPS ni copia fuera del servidor. Tres instituciones con datos reales y ningún respaldo automático |
| P-3 | Clave de API de Resend sin cargar | P1 | Todo lo demás del correo está configurado y el DNS de `mail.emerg-ia.com` verificado. Sin la clave, el sistema queda en modo simulado y nadie puede recuperar su contraseña solo |
| P-4 | Sin monitoreo de espacio en disco | P1 | Las notas de voz y los croquis crecen sin aviso |
| P-5 | Contraseña del usuario `ubuntu` en texto plano y acceso SSH por contraseña activo | P1 | Cambiarla y dejar solo llave |
| P-6 | Sin SMTP no hay activación por correo: `REQUIRE_EMAIL_ACTIVATION=false` | P2 | Se puede activar en cuanto el correo funcione (guía en `back-sci/infra/correo-resend.md`) |

## 4. Reportes — la etapa intermedia

Decidida como paso previo al módulo de incendios. **No hay nada implementado**: ni módulo en el
backend ni pantalla en el front.

Pero lo urgente no es construir los reportes, sino **empezar a guardar los datos que necesitan**,
porque la historia no se puede reconstruir después. Esto es lo que falta hoy:

| Dato | Estado | Qué habilita |
|:---|:-:|:---|
| Marca de tiempo de cada cambio de estado (reportada → activa → finalizada / cancelada) | ❌ Solo se guarda el estado actual | Duración, simultaneidad, tiempos de respuesta |
| Hora de primer arribo a la escena | ❌ | Tiempo de respuesta, el indicador más pedido |
| Hora de liberación del personal | ✅ Derivable de `organization_change` (`released` + `occurred_at`) | Horas de participación, fatiga |
| Catálogo de tipos de emergencia | 🟡 Es texto libre con sugerencias en el front | Estadística por tipo fiable; hoy "Incendio" e "incendio forestal" no agrupan |
| Zona administrativa (municipio o distrito) | ❌ | Reportes territoriales y boletín público |
| Fecha de cada devolución de recursos | ❌ Solo la cantidad devuelta | Tiempo de uso de los equipos |
| Historial de disponibilidad del personal | ❌ | Disponibilidad histórica |
| Instantes como `timestamptz` | ✅ `occurred_at` en emergencia, asignación, acción y triage | Consultas por rango con zona horaria correcta |

Y queda una decisión pendiente del responsable del proyecto: **validar o recortar la propuesta de
[08](./08_reportes_propuesta.md)** y elegir los primeros reportes. Es la única pregunta abierta que
queda en [06](./06_preguntas_abiertas.md).

## 5. Etapa 2 — Móvil con voz e IA

Es **el núcleo del proyecto** y está casi todo por hacer. Lo que existe son los cimientos del lado
del servidor, que no es poco: significa que la app puede empezar a construirse sin tocar el backend.

### Lo que ya está listo para el móvil

| Pieza | Estado |
|:---|:---|
| Subida de audio con idempotencia (`POST /action/upload-audio/:emergencyId`) | ✅ Con límite de 25 MB, lista blanca de formatos y firma del archivo verificada |
| Sincronización por lotes (`POST /sync/batch`) con idempotencia por `clientGeneratedId` | ✅ Con las mismas reglas de autorización que la web |
| Manejo de conflictos (emergencia cerrada mientras se estaba offline) | ✅ |
| Notificaciones push FCM | ✅ Integrado, sin probar con dispositivo real |
| Almacenamiento abstraído por claves | ✅ Falta solo el proveedor S3 (B-4) |

### Lo que falta

**App móvil (repositorio nuevo, no existe):**

| # | Pendiente | Nota |
|:-:|:---|:---|
| M-1 | App Flutter para Android e iOS con **la misma funcionalidad que la web** | Es el bloque de trabajo más grande de todo lo que queda |
| M-2 | Base de datos local, cola de sincronización y envío por lotes | El endpoint ya existe |
| M-3 | Grabación de notas de voz y subida | El endpoint ya existe |
| M-4 | Modo offline de punta a punta | Decisión de diseño ya documentada en [sistema/05](./sistema/05_guia_offline_y_sincronizacion.md) |

**Pipeline de IA (backend, nada implementado):**

| # | Pendiente | Nota |
|:-:|:---|:---|
| IA-1 | Interfaz `SpeechToTextProvider` intercambiable | Hoy no existe ninguna abstracción. Las columnas `transcription`, `nlpExtractedData` y `processed` están en `AudioEntity` desde el esquema inicial, y siguen vacías |
| IA-2 | Cola de procesamiento asíncrono de audios | Debe correr al sincronizar, no en la petición |
| IA-3 | Extracción de datos hacia 201 y 207, y resumen de acción cuando no hay datos de formulario | |
| IA-4 | Borradores pendientes de confirmación y flujo de confirmación humana | Decidido: el autor confirma su resumen; el CI o el LUMED confirman los datos de formulario (el CI si el cargo no está asignado). **La IA propone, una persona confirma** |
| IA-5 | Pantallas de confirmación en web y móvil | Lo que la IA propone tiene que poder revisarse desde las dos |

## 6. Etapas posteriores

| Etapa | Estado |
|:---|:---|
| **Formulario 211** (registro y control de recursos) | No empezado. El terreno está preparado: el inventario ya maneja vehículos con carga estándar y equipo por tipo de emergencia, que es lo que el 211 necesita tratar como recursos individuales |
| **Histórico y riesgo de incendios** | **En pausa, bloqueado.** Espera los requisitos funcionales del responsable. Solo existe `DataFireEntity`, que no se debe modificar hasta entonces (ver B-2) |
| Comando Unificado (varios CI) | Descartado por ahora (decisión 29) |
| Periodos operacionales formales | Fuera de alcance: el traspaso de mando marca el cambio de periodo (decisión 24) |

## 7. Deuda técnica

| # | Asunto | Prioridad | Nota |
|:-:|:---|:-:|:---|
| D-1 | **La suite E2E del backend (supertest) está podrida: 41 de 49 fallan** | P2 | En octubre fallaban 24; ha empeorado al ritmo de los cambios. Arma los módulos a mano con mocks que no siguieron a los servicios. **Hoy no protege nada y da una señal falsa de rotura.** Lo sensato es borrarla: los 41 escenarios de Playwright y las 452 unitarias cubren lo que ella pretendía |
| D-2 | La documentación quedó desfasada | P2 | [03](./03_funcionalidades.md) todavía marca como pendientes la recuperación de contraseña y el centro de notificaciones, que están hechos. [02](./02_alcance_y_etapas.md) dice "fuera de alcance: multitenant", y desde entonces se implementó institución configurable y despliegue de varias copias |
| D-3 | El CLI de TypeORM es sensible a la caja de la letra de unidad en Windows | P3 | `migration:generate` falla desde Git Bash y funciona desde PowerShell. La aplicación ya es inmune (las entidades se registran desde los módulos) |

## 8. Qué haría primero

1. **Los dos P0 de producción** (P-1 mapa y P-2 respaldos). Hay datos reales de tres instituciones
   sin respaldo automático, y el mapa roto es lo único que un usuario ve fallar hoy.
2. **Cargar la clave de Resend** (P-3). Está todo lo demás listo y desbloquea que la gente recupere
   su contraseña sin pedírselo al administrador.
3. **Las marcas de tiempo de §4**, aunque los reportes no se construyan todavía. Es poco trabajo y
   cada semana que pasa es historia que se pierde para siempre.
4. **Borrar la suite de supertest** (D-1) y los endpoints huérfanos (B-1). Limpieza de media hora que
   quita ruido y una señal falsa de rotura.
5. **Decidir los primeros reportes** (la pregunta abierta de [06](./06_preguntas_abiertas.md)), que
   es una decisión de producto, no de código.
6. **Arrancar la Etapa 2.** Es el núcleo del proyecto y el bloque más grande; conviene empezar por el
   pipeline de IA en el backend, que se puede construir y probar con audios subidos desde la web
   antes de que exista la app.
