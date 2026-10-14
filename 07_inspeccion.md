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
| S9 | Dependencias del backend en producción: 4 críticas (transitivas) y 19 altas. Directas con riesgo alto: `multer`, `typeorm`, `@nestjs/platform-express`, `bcrypt`, `nodemailer`, `joi`. **`mongoose` está instalado y no se usa** (el módulo está comentado) | `npm audit --omit=dev` | ✅ multer 2.4 (sin salir de Nest 10). **P1** el resto |
| S10 | Angular 19: 7 vulnerabilidades altas sin parche en la versión 19. Se corrigen al subir de versión mayor | `pnpm audit --prod` | P1 |
| S11 | Sin cabeceras de seguridad (`helmet`) y Swagger (`/docs`) expuesto también en producción | `main.ts` | P1 |
| S12 | `POST /checkToken` y `/expiredToken` reciben el token en la *query string*, que queda en los logs de acceso | `AuthController` | P2 |
| S13 | `GET /data-fire` (módulo en pausa) con CRUD abierto a cualquier usuario autenticado | `DataFireController` | P2 (pausado) |

Correcto y verificado: alta y edición de inventario solo para manager+; F201 y evaluación inicial solo para CI o manager+; F207 para CI, LUMED o manager+; triage, bitácora y despacho para personal asignado; devolución para CI o manager+; un `admin` no puede ver ni modificar cuentas `suadmin`; rate limiting.

## 3. Conformidad con el Manual SCI

### 3.1 Formulario SCI-201

| Campo del manual | Estado | Detalle |
|:---|:-:|:---|
| 1 Nombre del incidente | ✅ | `emergency.name` |
| 2 Fecha **y hora** de preparación | 🟡 | El F201 guarda solo `date`; falta la hora |
| 3 Lugar + coordenadas | ✅ | Desde la emergencia |
| 4 Evaluación inicial (naturaleza, amenazas, área, **aislamiento**) | 🟡 | El aislamiento vive en la evaluación inicial, no en el 201. La hoja impresa lo toma de ahí |
| 5 Objetivos iniciales (lista) | 🟡 | Un solo texto libre, no una lista |
| 6 Ubicación del PC | ✅ | `coordinates_pc` |
| 7 Ubicación de las Áreas de Espera (**lista**) | 🟡 | Solo un punto E |
| 8-9 Rutas de ingreso y egreso | ✅ | Texto |
| 10 Mensaje de seguridad | ✅ | |
| 11 Comandante del incidente | ✅ | Derivado del CI activo |
| 12 **Mapa o croquis** | ❌ | Solo un campo URL. Decisión 3: imagen y mapa dibujable |
| 13 **Resumen de las acciones** (cronológico) | 🟡 | La bitácora existe, pero **la hoja imprimible del 201 no la incluye** (es la página 3 del formulario) |
| 14 Organización actual (**historial secuencial**) | 🟡 | Snapshot solo al crear el 201. El manual pide registrar cada cambio de estructura |
| Un 201 **por periodo** (= por mando, decisión 24) | ❌ | El índice `uq_form201_active_per_emergency` solo filtra `is_deleted = false`: permite **un único 201 en toda la vida de la emergencia**, aunque esté finalizado. Debe restringir solo los 201 **no finalizados**, para que el nuevo CI abra el suyo tras el traspaso |
| El 201 completo es obligatorio para transferir el mando | 🟡 | Se exige *finalizado*; no se validan los campos mínimos (1-6, 10, 11) |

### 3.2 Formulario SCI-207

| Campo o regla del manual | Estado | Detalle |
|:---|:-:|:---|
| 2 Lugar de registro: **ACV o UM** | 🟡 | Texto libre. Decisión 25: ACV/UM por defecto, editable |
| 3-5 Nombre, sexo, edad (estimada) | ✅ | Admite "NN" y "No determinado" |
| 6 Clasificación por colores | ✅ | Con re-evaluaciones e historial: el manual lo sugiere y está implementado |
| 7 **Lugar de traslado** | ❌ | No existe. Decisión 27: tabla de centros de salud creada sobre la marcha, con autocompletado |
| 8 Trasladado por (**medio**: ambulancia, aéreo, embarcación, otro) | 🟡 | Texto libre |
| Regla: traslado ⇔ medio (ambos o ninguno) | ❌ | Sin validación |
| 9 Fecha y hora por paciente | ✅ | Por registro |
| 10 Preparado por | ✅ | `attendant` |
| Quién registra: **EACV**, **LUMED** o **CI** (decisión 26) | ❌ | Hoy registra cualquier persona asignada. **El cargo EACV no existe en el catálogo** |
| Totales por color | ✅ | Resumen por clasificación vigente |
| Totales por destino o medio | ❌ | Depende del campo de destino |
| Datos sensibles con acceso restringido | ❌ | Ver S5 |

### 3.3 Mando, organización y periodos

| Requisito | Estado | Detalle |
|:---|:-:|:---|
| Un único CI activo | ✅ | Índice único parcial en la BD |
| Comando Unificado (varios CI) | ❌ | No contemplado. No fue pedido; anotado para decidir |
| Transferencia de mando formal y registrada | ✅ | Transaccional, con entrada en la bitácora |
| Transferir exige **F201 y F207 finalizados** + flujo "finalizar todo" | ❌ | Solo exige el F201 (decisión 6) |
| Checklist de transferencia del manual | ❌ | No hay checklist ni confirmación de quien recibe |
| Catálogo de cargos del manual | 🟡 | 35 cargos, con terminología ICS ("Director de Rama", "Equipo de Ataque", "Unidad de Compras/Transporte"). Faltan **EAE**, **EACV**, CROA, helibase y helipunto, registradores. Incluye entradas que no son cargos SCI ("Rescatista", "Paramédico", "Bombero", "Voluntario") |
| Jerarquía con un único superior | 🟡 | Los cargos tienen nivel y peso, pero no superior (sin `parentId`) |
| Periodo Inicial (1-4 h) y Periodos Operacionales (12-24 h) | ➖ | **Fuera de esta etapa** (decisión 24): el traspaso de mando marca el cambio de periodo |
| Reunión de Evaluación | ➖ | Fuera de esta etapa (decisión 24) |

## 4. Bugs funcionales detectados

| # | Bug | Impacto | Prioridad |
|:-:|:---|:---|:-:|
| B1 | Fecha calculada en UTC en el front (`toISOString().split('T')[0]`) al crear emergencias y asignar personal | Desde las 20:00 en Bolivia se registra el día siguiente | **P1** |
| B2 | Horas de la bitácora automática con `getHours()` del proceso Node | En un VPS en UTC quedan +4 h. Mitigación: `TZ=America/La_Paz` en el despliegue; mejor guardar `timestamptz` | **P1** |
| B3 | KPI "víctimas" del dashboard siempre en 0 (`victimsCount` nunca llega) | Dato falso en pantalla | P2 |
| B4 | `DELETE /resource/:id` no restituye el stock | Inventario descuadrado | **P0** (ver S3) |
| B5 | La hoja imprimible del 201 no incluye el resumen de acciones | Formulario incompleto respecto al oficial | P1 |

## 5. Funcionalidades faltantes en la web

| Funcionalidad | Prioridad | Nota |
|:---|:-:|:---|
| Editar los datos de una emergencia ya creada (nombre, PC, E) | P1 | El servicio existe; falta la pantalla. En terreno, el PC y el E suelen definirse después |
| Recuperación de contraseña | P1 | Solo existe la plantilla de correo |
| Administración del catálogo de cargos SCI (admin) | P2 | El backend lo soporta; no hay pantalla |
| Centro de notificaciones (incluidos los conflictos de sincronización) | P2 | El backend genera notificaciones; la web no las muestra |
| Croquis del 201 (imagen + mapa dibujable) | P1 | Decisión 3 |
| Flujo "finalizar todo y traspasar el mando" | P1 | Decisión 6 |
| Reportes y estadísticas operativas | P2 | Falta definir cuáles (pregunta 6) |

## 6. Pruebas y calidad

| Suite | Resultado | Observación |
|:---|:---|:---|
| Unitarias backend (Jest) | ✅ 272/272 | |
| Unitarias front (Karma) | ✅ 451/451 | |
| E2E de punta a punta (Playwright) | ✅ 17/17 | Base aislada `sci_e2e` |
| **E2E del backend (supertest)** | ❌ **24/49 fallan** | **Desactualizada**: arma módulos a mano con mocks que no siguieron a los servicios (dependencias faltantes, métodos inexistentes en los mocks, reglas que cambiaron). Ya fallaban los 24 antes del 2026-10-03; los 6 que rompieron mis cambios están corregidos. Hoy esta suite no protege nada: hay que repararla o reemplazarla por los E2E de Playwright más tests de integración con BD real |

Deuda técnica relevante:
- **Almacenamiento no listo para S3.** `IStorageService` expone `getFilePath()` (ruta física) y los audios no lo usan: escriben con `diskStorage` y se leen con `fs` directo. Hace falta una interfaz basada en streams o URLs firmadas y que audios e imágenes pasen por ella (decisión 20).
- **`synchronize: true` también en producción**, sin migraciones. Es una decisión del proyecto, pero en producción un cambio de entidad puede borrar columnas con datos. Recomiendo como mínimo respaldo automático previo a cada despliegue; idealmente, migraciones.
- **Fechas guardadas como texto** (`date` y `hour` como `varchar` en emergencias y acciones): complica los reportes por rango (decisión 19) y las zonas horarias.
- **`mongoose` sin uso** en las dependencias.

## 7. Preparación para producción (VPS)

- [ ] HTTPS con un *reverse proxy* (Nginx/Caddy) delante de Node, incluido el WebSocket.
- [ ] `APP_PROD=true` (bloquea los seeders y restringe CORS), `TZ=America/La_Paz` y secretos solo en el entorno.
- [ ] Respaldos automáticos de PostgreSQL y de `data/` (audios e imágenes), con prueba de restauración.
- [ ] `helmet` y Swagger deshabilitado o protegido en producción.
- [ ] Límites de subida (tipo y tamaño) y espacio en disco monitoreado.
- [ ] Actualizar las dependencias con vulnerabilidades altas.
- [ ] Rotar la clave de Google Maps (está en el historial público) y restringirla por dominio.

## 8. Preparación para la etapa móvil e IA

| Pieza | Estado |
|:---|:---|
| Subida de audio con idempotencia | ✅ |
| Sincronización por lotes + conflictos | 🟡 Funciona, pero con el bypass de permisos (S6) |
| Push FCM | ✅ Integrado (sin probar con dispositivo) |
| Interfaz `SpeechToTextProvider` intercambiable | ❌ |
| Cola de procesamiento asíncrono de audios | ❌ |
| Extracción hacia 201/207 + borradores pendientes de confirmación | ❌ |
| Flujo de confirmación (autor / CI / LUMED) | ❌ |
| Storage abstraído (S3) | ❌ Ver §6 |

## 9. Backlog priorizado sugerido

**P0 — Seguridad e integridad** ✅ *Resuelto el 2026-10-14*:
1. Autorización en `changeState` y `PATCH` de la emergencia (CI o manager+), en `PATCH`/`DELETE` de recursos (y que el DELETE restituya el stock o se prohíba), en `PATCH` de víctimas y en adjuntar audio.
2. Restricción de lectura de víctimas y del 207 (decisión 18).
3. Propagar usuario y rol en `/sync/batch`.
4. Límites de tipo y tamaño en la subida de audio y actualización de `multer`.

**P1 — Cierre de la Etapa 1 conforme al manual y a las decisiones:**
5. Fechas y horas: corrección del front (B1) y `TZ` del servidor (B2).
6. Traspaso de mando: exigir F201 y F207 finalizados, crear el flujo "finalizar todo" y permitir que el nuevo CI abra **su propio F201** (cambiar el índice único a "un 201 no finalizado por emergencia").
7. 207: destino del traslado con **tabla de centros de salud y autocompletado**, medio de traslado enumerado, regla "ambos o ninguno", lugar de registro ACV/UM editable y registro restringido a EACV, LUMED, CI y `manager`/`admin`.
8. Catálogo de cargos alineado al manual: EAE, EACV, terminología LAC, sin entradas que no son cargos. Migración de datos de los cargos existentes.
9. Croquis del 201: imagen; luego el mapa dibujable.
10. Hoja del 201 con resumen de acciones, hora de preparación y objetivos como lista.
11. Pantallas: editar emergencia y recuperación de contraseña.
12. Storage abstraído (disco hoy, S3 mañana) para audios e imágenes.
13. Seguridad de producción: helmet, Swagger, dependencias.

**P2:**
- KPI de víctimas.
- Centro de notificaciones.
- Administración de cargos.
- Reparar o reemplazar la suite E2E del backend.
- Fechas como `timestamptz`.
- Retirar `mongoose`.
- Primeros reportes (propuesta en [08](./08_reportes_propuesta.md)). Requieren antes guardar las marcas de tiempo que hoy faltan: liberación de personal y cambios de estado de la emergencia.

**P3:** App móvil Flutter y pipeline de IA (Etapa 2).
