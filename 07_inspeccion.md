# 07 — Inspección del proyecto (2026-10-14)

Revisión de `back-sci` (rama `main`, commit `a04b1f6`) y `front-sci` (rama `main`, commit `b1e6a82`) contra tres referencias:
- el *Manual de Campo SCI* (USAID/OFDA/LAC, 2013), resumido en [`SCI_fase_basica/`](./SCI_fase_basica/00_INDICE.md);
- las [decisiones del 2026-10-14](./06_preguntas_abiertas.md);
- buenas prácticas para operar en producción.

Método:
- revisión de código de cada controlador y servicio;
- mapa de autorización de todas las rutas;
- ejecución de todas las suites de prueba;
- auditoría de dependencias;
- contraste campo por campo de los formularios.

## 1. Resumen

**Lo que funciona y está verificado:**
- El flujo web de la Etapa 1 (emergencia → organigrama → 201 → 207 → recursos → bitácora → tiempo real → impresión) funciona de punta a punta: **17/17 escenarios E2E** con Playwright.
- Tests unitarios en verde: **451** en el front y **272** en el back.

**Los cinco problemas más graves:**
1. **Autorización incompleta en el backend.** Cualquier usuario autenticado puede, por API:
   - cambiar el estado de cualquier emergencia (activarla, finalizarla o cancelarla) y editar sus datos;
   - borrar o editar recursos despachados;
   - editar datos de víctimas;
   - saltarse todas las reglas tácticas usando `/sync/batch`.

   La web oculta esos botones, pero el servidor no lo impide.
2. **Datos de víctimas sin restricción:** cualquier usuario autenticado ve las víctimas de cualquier emergencia. Contradice la decisión 18 y la regla 8 del 207 en el manual.
3. **Subida de audios sin límites** de tipo ni tamaño, y `multer` con vulnerabilidades altas.
4. **Fechas y horas desfasadas en Bolivia (UTC-4):**
   - el front calcula la fecha en UTC: desde las 20:00, emergencias y asignaciones quedan con la fecha del día siguiente;
   - en un VPS en UTC, las horas de la bitácora automática quedarán 4 horas adelantadas.
5. **Brechas con el manual en el 201 y el 207:**
   - no hay periodos operacionales ni un 201 por periodo;
   - el 207 no tiene destino del traslado;
   - el traspaso de mando no exige el 207 finalizado;
   - el catálogo de cargos usa la terminología estadounidense (ICS) y le faltan cargos del manual, como el **Encargado del ACV**, que es quien registra el 207.

> **Actualización 2026-10-14 (tarde): los P0 S1-S8 están resueltos** en `back-sci` (commits `9840452`…`cd65cfa`) y `front-sci` (`b3bd1ef`, `5ea877f`), con tests unitarios y un escenario E2E (`09-security`) que ataca la API directamente. Ver la columna "Estado" de §2.

Leyenda de prioridad:
- **P0:** antes de cualquier uso con datos reales.
- **P1:** necesario para cerrar la Etapa 1 conforme al manual y a las decisiones.
- **P2:** mejora importante.
- **P3:** posterior.

## 2. Seguridad y autorización

Mapa completo de rutas del backend, revisado servicio por servicio.

| # | Hallazgo | Evidencia | Prioridad / Estado |
|:-:|:---|:---|:-:|
| S1 | `PATCH /emergency/:id/state` no valida quién cambia el estado: solo exige rol para *reabrir*. Un `basic` ajeno puede finalizar o cancelar cualquier emergencia | `EmergencyService.changeState` | ✅ Resuelto |
| S2 | `PATCH /emergency/:id` (nombre, coordenadas, PC, E) sin control de permisos | `EmergencyController.update` no recibe el usuario | ✅ Resuelto |
| S3 | `PATCH` y `DELETE /resource/:id` sin control de permisos. Además, **el DELETE no devuelve el stock**: borrar un despacho deja las unidades descontadas para siempre | `ResourceService.update/delete` | ✅ Resuelto: CI o manager+ y la anulación devuelve el stock |
| S4 | `PATCH /victim/:id` sin control: cualquiera edita la filiación de cualquier víctima | `VictimService.update` | ✅ Resuelto: EACV, LUMED, CI o manager+ |
| S5 | Lectura de datos de víctimas abierta a todo usuario autenticado: `GET /victim/:id`, `GET /victim/:id/registration`, `GET /form207/:id/registration`, `GET /form207/:id` | Guard de clase sin filtro por participación | ✅ Resuelto (decisión 18) |
| S6 | `POST /sync/batch` llama a los servicios **sin el rol del usuario**, así que se saltan todas las validaciones tácticas (F201, F207, triage, bitácora). Es un *bypass* disponible desde ya, no solo en la etapa móvil. Además `form201Service.update` se invoca sin el usuario | `SyncService.processBatch` | ✅ Resuelto |
| S7 | `POST /action/:id/audio` (adjuntar audio a una acción) sin control de pertenencia | `ActionService.attachAudioToAction` | ✅ Resuelto |
| S8 | `POST /action/upload-audio` sin `fileFilter` ni `limits`: acepta cualquier archivo de cualquier tamaño en el disco del servidor | `ActionController` (multer) | ✅ Resuelto: 25 MB, lista blanca de formatos, UUID validado antes de tocar el disco |
| S9 | Dependencias del backend en producción: 4 críticas (transitivas) y 19 altas. Directas con riesgo alto: `multer`, `typeorm`, `@nestjs/platform-express`, `bcrypt`, `nodemailer`, `joi`. **`mongoose` está instalado y no se usa** (el módulo está comentado) | `npm audit --omit=dev` | ✅ Resuelto: 0 críticas y 0 altas; quedan 11 moderadas de Nest 10 |
| S10 | Angular 19: 7 vulnerabilidades altas sin parche en la versión 19. Se corrigen al subir de versión mayor | `pnpm audit --prod` | ✅ Resuelto: Angular 21.2 |
| S11 | Sin cabeceras de seguridad (`helmet`) y Swagger (`/docs`) expuesto también en producción | `main.ts` | ✅ Resuelto |
| S12 | `POST /checkToken` y `/expiredToken` reciben el token en la *query string*, que queda en los logs de acceso | `AuthController` | P2 |
| S13 | `GET /data-fire` (módulo en pausa) con CRUD abierto a cualquier usuario autenticado | `DataFireController` | P2 (pausado) |

Correcto y verificado: alta y edición de inventario solo para manager+; F201 y evaluación inicial solo para CI o manager+; F207 para CI, LUMED o manager+; triage, bitácora y despacho para personal asignado; devolución para CI o manager+; un `admin` no puede ver ni modificar cuentas `suadmin`; rate limiting.

## 3. Conformidad con el Manual SCI

### 3.1 Formulario SCI-201

| Campo del manual | Estado | Detalle |
|:---|:-:|:---|
| 1 Nombre del incidente | ✅ | `emergency.name` |
| 2 Fecha **y hora** de preparación | ✅ | `prepared_at` (UTC), ingresado en hora de la institución; los 201 anteriores usan la hora de creación. Se repite en el encabezado de las 4 páginas junto con las casillas 1 y 11 |
| 3 Lugar + coordenadas | ✅ | Desde la emergencia |
| 4 Evaluación inicial (naturaleza, amenazas, área, **aislamiento**) | 🟡 | El aislamiento vive en la evaluación inicial, no en el 201. La hoja impresa lo toma de ahí |
| 5 Objetivos iniciales (lista) | ✅ | Se escriben uno por línea (también se reconoce "1. … 2. …") y se muestran e imprimen como lista numerada. Sigue siendo un texto en la base: no hay estado por objetivo |
| 6 Ubicación del PC | ✅ | `coordinates_pc` |
| 7 Ubicación de las Áreas de Espera (**lista**) | 🟡 | Solo un punto E |
| 8-9 Rutas de ingreso y egreso | ✅ | Texto |
| 10 Mensaje de seguridad | ✅ | |
| 11 Comandante del incidente | ✅ | El CI del organigrama capturado en ese 201 (si no figura, el CI activo) |
| 12 **Mapa o croquis** | ✅ | Imagen del croquis (JPG/PNG/WebP hasta 10 MB, firma del archivo verificada, se sirve solo con sesión) y mapa de situación dibujable: perímetro de seguridad y símbolos del Anexo D (PC, E, ACV, B, C, H, H1, origen, peligro, primeros auxilios, agua, recurso), con el incidente, el PC y el E de la emergencia como referencia y el norte arriba. Editable mientras el 201 esté abierto; en la hoja imprimible va en página aparte (decisión 3) |
| 13 **Resumen de las acciones** (cronológico) | ✅ | Página 3 de la hoja: la bitácora en orden cronológico, solo del periodo de ese mando (hasta `finalized_at` del 201; lo anterior al primer 201 va al primero). Las entradas no tienen aún los campos estrategia/táctica/problema/área por separado |
| 14 Organización actual (**historial secuencial**) | 🟡 | Snapshot solo al crear el 201. El manual pide registrar cada cambio de estructura |
| Un 201 **por periodo** (= por mando, decisión 24) | ✅ | Índice `uq_form201_open_per_emergency`: como máximo un 201 **abierto**. Tras el traspaso el nuevo CI inicia el suyo; los anteriores se consultan e imprimen desde el selector de mandos (`/print/form201/:id?form=<id>`) |
| El 201 completo es obligatorio para transferir el mando | 🟡 | Se exige *finalizado*; no se validan los campos mínimos (1-6, 10, 11) |

### 3.2 Formulario SCI-207

| Campo o regla del manual | Estado | Detalle |
|:---|:-:|:---|
| 2 Lugar de registro: **ACV o UM** | ✅ | La web precarga "Área de Concentración de Víctimas (ACV)" y sugiere "Unidad Médica (UM)"; es editable (decisión 25). Si llega vacío, el backend usa el ACV |
| 3-5 Nombre, sexo, edad (estimada) | ✅ | Admite "NN" y "No determinado" |
| 6 Clasificación por colores | ✅ | Con re-evaluaciones e historial: el manual lo sugiere y está implementado |
| 7 **Lugar de traslado** | ✅ | Tabla `health_facility`: se crea sobre la marcha al registrar el traslado y alimenta el autocompletado (`GET /health-facility?search=`). Sin distinguir tildes, mayúsculas ni espacios, para no duplicar centros (decisión 27) |
| 8 Trasladado por (**medio**: ambulancia, aéreo, embarcación, otro) | ✅ | `transport_means` enumerado; `transferred_by` queda como la unidad o el vehículo (p. ej. "SAMU 12") |
| Regla: traslado ⇔ medio (ambos o ninguno) | ✅ | Validada en el backend (400) y en el formulario web |
| 9 Fecha y hora por paciente | ✅ | Por registro |
| 10 Preparado por | ✅ | `attendant` |
| Quién registra: **EACV**, **LUMED** o **CI** (decisión 26) | ✅ | Más `manager`/`admin`. El cargo EACV ya está en el seeder |
| Totales por color | ✅ | Resumen por clasificación vigente |
| Totales por destino o medio | ✅ | En la hoja imprimible, según el traslado vigente de cada víctima (una re-evaluación posterior no lo borra) |
| Datos sensibles con acceso restringido | ✅ | Ver S5 |

### 3.3 Mando, organización y periodos

| Requisito | Estado | Detalle |
|:---|:-:|:---|
| Un único CI activo | ✅ | Índice único parcial en la BD |
| Comando Unificado (varios CI) | ❌ | No contemplado. No fue pedido; anotado para decidir |
| Transferencia de mando formal y registrada | ✅ | Transaccional, con entrada en la bitácora |
| Transferir exige **F201 y F207 finalizados** + flujo "finalizar todo" | ✅ | `GET /emergency/:id/forms/pending` y `POST /emergency/:id/forms/finalize-all` (CI o manager+, transaccional, con entrada en la bitácora). El modal lista los códigos abiertos, advierte y exige confirmar antes de cerrar y traspasar |
| Checklist de transferencia del manual | ❌ | No hay checklist ni confirmación de quien recibe |
| Catálogo de cargos del manual | ✅ | 43 cargos con la terminología y siglas del manual (Anexo A): EAE, EACV, CROA, helibase, helipunto, registrador, radioperador y operador de ambulancia incluidos. Fuente única en `back-sci/src/sci_module/charges/sci-charge-catalog.ts`, sincronizada al arrancar: renombra en el mismo registro (las asignaciones se conservan), fusiona duplicados y retira lo que no son cargos (Rescatista, Paramédico, Bombero, Voluntario). Los cargos que cree un administrador no se tocan |
| Jerarquía con un único superior | 🟡 | Los cargos tienen nivel y peso, pero no superior (sin `parentId`) |
| Periodo Inicial (1-4 h) y Periodos Operacionales (12-24 h) | ➖ | **Fuera de esta etapa** (decisión 24): el traspaso de mando marca el cambio de periodo |
| Reunión de Evaluación | ➖ | Fuera de esta etapa (decisión 24) |

## 4. Bugs funcionales detectados

| # | Bug | Impacto | Prioridad |
|:-:|:---|:---|:-:|
| B1 | Fecha calculada en UTC en el front (`toISOString().split('T')[0]`) al crear emergencias y asignar personal | Desde las 20:00 en Bolivia se registra el día siguiente | ✅ Resuelto: todo se guarda en UTC y la web convierte a la zona de la institución |
| B2 | Horas de la bitácora automática con `getHours()` del proceso Node | En un VPS en UTC quedan +4 h | ✅ Resuelto: el proceso se fija en UTC y la zona de la institución (`APP_TIMEZONE`) se usa solo para mostrar y reportar |
| B6 | El listado de emergencias no traía el usuario de las asignaciones: `findCommanderName` lanzaba una excepción y la fila quedaba sin Comandante ni Fecha/Hora | Columna Comandante del listado siempre rota | ✅ Resuelto |
| B7 | Fechas de calendario (F201, F207) mostradas con `DatePipe`, que las interpreta como medianoche UTC | `2026-10-03` se veía `02/10/2026` en UTC-4 | ✅ Resuelto |
| B3 | KPI "víctimas" del dashboard siempre en 0 (`victimsCount` nunca llega) | Dato falso en pantalla | P2 |
| B4 | `DELETE /resource/:id` no restituye el stock | Inventario descuadrado | ✅ Resuelto (ver S3) |
| B5 | La hoja imprimible del 201 no incluye el resumen de acciones | Formulario incompleto respecto al oficial | ✅ Resuelto: página 3 |
| B8 | El buscador de personal usaba `GET /user` (solo manager+): un CI con rol `basic` no podía asignar personal ni traspasar el mando desde la web | Flujos del CI bloqueados | ✅ Resuelto: `GET /user/directory` para cualquier usuario autenticado, solo con nombre, apellido, grado y disponibilidad (sin correo ni rol) |
| B9 | Crear un 207 sin lugar de registro devolvía 500 (el DTO lo admite opcional, la columna es NOT NULL) | El móvil u otro cliente no podía abrir un 207 | ✅ Resuelto: ACV por defecto |
| B10 | Los modales tenían `aria-labelledby` con el título como id (con espacios), así que el diálogo no tenía nombre accesible | Lectores de pantalla y pruebas | ✅ Resuelto: id único por modal |
| B11 | Los correos de activación enlazaban a `/auth/activate`, ruta que no existe en el front (es `/activate`) | Nadie podía activar su cuenta desde el correo | ✅ Resuelto: rutas corregidas en el backend y redirección en el front para los enlaces ya enviados |
| B14 | El `AuthGuard` solo decodificaba el JWT (`jwt.decode`), sin verificar la firma: cualquiera podía fabricar un token con el `sub` de otro usuario, incluido un superadministrador. Además un refresh token servía como token de acceso, y sin `JWT_EXPIRATION` el token no expiraba | Toma de cualquier cuenta | ✅ Resuelto: firma HS256, expiración y `typ` verificados en HTTP y WebSocket; 8 h por defecto y renovación automática en la web |
| B15 | El `ValidationPipe` global no tenía `whitelist`: los DTO aceptaban propiedades no declaradas | Asignación masiva de campos | ✅ Resuelto: se descartan |
| B13 | `POST /action` y `POST /action/:id/audio` aceptaban cualquier `pathAudio` del cliente: se podía enlazar a una acción propia el audio de otra persona | Acceso a notas de voz ajenas | ✅ Resuelto: el audio debe ser del mismo usuario y emergencia, y existir en el almacenamiento |
| B12 | `AuthService.recoverPassword(email)` entregaba un access token a quien supiera un correo. No estaba expuesto, pero bastaba con conectarlo a una ruta | Toma de cuentas | ✅ Resuelto: eliminado y reemplazado por el flujo con token de un solo uso |

## 5. Funcionalidades faltantes en la web

| Funcionalidad | Prioridad | Nota |
|:---|:-:|:---|
| Editar los datos de una emergencia ya creada (nombre, PC, E) | ✅ | Botón "Editar" en la cabecera del Centro de Mando (CI o manager+, emergencia abierta): nombre, tipo, lugar, inicio en hora de la institución y puntos del incidente, PC y E en el mapa |
| Recuperación de contraseña | ✅ | `POST /forgot-password` (respuesta idéntica exista o no la cuenta, 3/min) y `POST /reset-password` (token de un solo uso, 1 h, guardado con hash; al usarse revoca los refresh tokens). Pantallas `/forgot-password` y `/recover-password`; el token se quita de la barra de direcciones. Los access tokens ya emitidos siguen vigentes hasta que expiran |
| Administración del catálogo de cargos SCI (admin) | P2 | El backend lo soporta; no hay pantalla |
| Centro de notificaciones (incluidos los conflictos de sincronización) | P2 | El backend genera notificaciones; la web no las muestra |
| Croquis del 201 (imagen + mapa dibujable) | ✅ | Decisión 3 |
| Flujo "finalizar todo y traspasar el mando" | ✅ | Decisión 6 |
| Reportes y estadísticas operativas | P2 | Falta definir cuáles (pregunta 6) |

## 6. Pruebas y calidad

| Suite | Resultado | Observación |
|:---|:---|:---|
| Unitarias backend (Jest) | ✅ 380/380 | Al 2026-10-16 |
| Unitarias front (Karma) | ✅ 524/524 | |
| E2E de punta a punta (Playwright) | ✅ 31/31 | Base `sci_e2e` y carpeta de archivos propias |
| **E2E del backend (supertest)** | ❌ **24/49 fallan** | **Desactualizada**: arma módulos a mano con mocks que no siguieron a los servicios (dependencias faltantes, métodos inexistentes en los mocks, reglas que cambiaron). Ya fallaban los 24 antes del 2026-10-03; los 6 que rompieron mis cambios están corregidos. Hoy esta suite no protege nada: hay que repararla o reemplazarla por los E2E de Playwright más tests de integración con BD real |

Deuda técnica relevante:
- **`synchronize: true` también en producción**, sin migraciones. Es una decisión del proyecto, pero en producción un cambio de entidad puede borrar columnas con datos. Recomiendo como mínimo respaldo automático previo a cada despliegue; idealmente, migraciones.
- **Fechas guardadas como texto** (`date` y `hour` como `varchar` en emergencias y acciones): complica los reportes por rango (decisión 19) y las zonas horarias.

## 7. Preparación para producción (VPS)

- [ ] HTTPS con un *reverse proxy* (Nginx/Caddy) delante de Node, incluido el WebSocket.
- [ ] `APP_PROD=true` y secretos solo en el entorno. La API **no arranca** con `JWT_AUTH` de menos de 32 caracteres o de ejemplo, `ADMIN_PASSWORD` débil o CORS abierto. El primer superadministrador se crea al arrancar desde `ADMIN_EMAIL`/`ADMIN_PASSWORD`. (El proceso corre en UTC por diseño: no fijar `TZ`; la zona de la institución es `APP_TIMEZONE`.)
- [ ] Respaldos automáticos de PostgreSQL y de `data/` (audios e imágenes), con prueba de restauración.
- [x] `helmet` y Swagger apagado en producción (salvo `SWAGGER_ENABLED=true`). `/seed` responde 404. `TRUST_PROXY=1` detrás del proxy y `DB_SSL` según dónde esté Postgres.
- [ ] Límites de subida ✅ (audio 25 MB, imágenes 10 MB, firma del archivo verificada); falta monitorear el espacio en disco.
- [x] Dependencias: backend sin altas ni críticas en producción (quedan 11 moderadas propias de Nest 10: migrar a Nest 11 en una etapa posterior); front en Angular 21.2 sin vulnerabilidades conocidas. Angular 22 exige Node 24.15+.
- [ ] Rotar la clave de Google Maps (está en el historial público) y restringirla por dominio.

## 8. Preparación para la etapa móvil e IA

| Pieza | Estado |
|:---|:---|
| Subida de audio con idempotencia | ✅ |
| Sincronización por lotes + conflictos | ✅ Con las mismas reglas que la web (S6) |
| Push FCM | ✅ Integrado (sin probar con dispositivo) |
| Interfaz `SpeechToTextProvider` intercambiable | ❌ |
| Cola de procesamiento asíncrono de audios | ❌ |
| Extracción hacia 201/207 + borradores pendientes de confirmación | ❌ |
| Flujo de confirmación (autor / CI / LUMED) | ❌ |
| Storage abstraído (S3) | ✅ Por claves; falta el proveedor S3 (P1 #12) |

## 9. Backlog priorizado sugerido

**P0 — Seguridad e integridad** ✅ *Resuelto el 2026-10-14*:
1. Autorización en `changeState` y `PATCH` de la emergencia (CI o manager+), en `PATCH`/`DELETE` de recursos (y que el DELETE restituya el stock o se prohíba), en `PATCH` de víctimas y en adjuntar audio.
2. Restricción de lectura de víctimas y del 207 (decisión 18).
3. Propagar usuario y rol en `/sync/batch`.
4. Límites de tipo y tamaño en la subida de audio y actualización de `multer`.

**P1 — Cierre de la Etapa 1 conforme al manual y a las decisiones:**
5. ✅ Fechas y horas en UTC con zona horaria de la institución (B1, B2, B6, B7).
6. ✅ Traspaso de mando: exige F201 y F207 finalizados, flujo "finalizar todo" con advertencias, y el nuevo CI abre **su propio F201** (B8, B9 y B10 aparecieron al probarlo de punta a punta).
7. ✅ 207: destino del traslado con **tabla de centros de salud y autocompletado**, medio de traslado enumerado, regla "ambos o ninguno", lugar de registro ACV/UM editable y registro restringido a EACV, LUMED, CI y `manager`/`admin`.
8. ✅ Catálogo de cargos alineado al manual: EAE, EACV, terminología LAC, sin entradas que no son cargos. Migración de datos de los cargos existentes (aplicada también en la base de desarrollo).
9. ✅ Croquis del 201: imagen y mapa dibujable (sin la Drawing Library de Google, que ya no está disponible).
10. ✅ Hoja del 201 en las 4 páginas del formulario oficial: resumen de acciones, hora de preparación, objetivos como lista y encabezado repetido.
11. ✅ Pantallas: editar emergencia y recuperación de contraseña.
12. ✅ Storage abstraído por claves (`IStorageService`: put/get/exists/delete, el contrato de S3) para fotos de perfil, croquis y notas de voz; hoy en disco (`LocalStorageService`, raíz `STORAGE_LOCAL_ROOT`). Para S3 falta solo un proveedor de `STORAGE_SERVICE` con `@aws-sdk/client-s3`. El E2E usa `front-sci/e2e/.state/storage` y la vacía en cada corrida. Sin migración de datos: las claves respetan la estructura existente de `data/` y los audios antiguos (`data/audios/...`) se leen igual.
13. ✅ Seguridad de producción: verificación real del JWT (B14), whitelist (B15), helmet, Swagger, `/seed` fuera de producción, chequeo de secretos al arrancar, `TRUST_PROXY`, `DB_SSL`, dependencias actualizadas y Angular 21. Queda del lado del despliegue: HTTPS, respaldos, rotar la clave de Google Maps y monitoreo de disco.

**P2:**
- KPI de víctimas.
- Centro de notificaciones.
- Administración de cargos.
- Reparar o reemplazar la suite E2E del backend.
- Fechas como `timestamptz`.
- Primeros reportes (propuesta en [08](./08_reportes_propuesta.md)). Requieren antes guardar las marcas de tiempo que hoy faltan: liberación de personal y cambios de estado de la emergencia.

**P3:** App móvil Flutter y pipeline de IA (Etapa 2).
