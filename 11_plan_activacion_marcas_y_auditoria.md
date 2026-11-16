# 11 — Plan de implementación: cuentas, marcas de tiempo y auditoría (2026-11-16)

Plan ejecutable, con las decisiones ya tomadas. Reemplaza la versión anterior de este documento, que
dejaba cinco preguntas abiertas.

## 1. Decisiones tomadas

| # | Decisión | Consecuencia |
|:-:|:---|:---|
| 1 | **La invitación por correo queda apagada** (`REQUIRE_EMAIL_ACTIVATION=false`). El administrador crea al usuario con una contraseña y se la entrega | No se envía ningún correo de activación. Toda la interfaz de activación es interfaz para un flujo apagado: se retira en lugar de construirse (ver §4) |
| 2 | Los usuarios ya creados **se quedan como están**, activos | Sin migración de cuentas |
| 3 | Zona administrativa: **departamento, municipio y distrito** | Tres campos (§6) |
| 4 | Catálogo de tipos de emergencia: crece solo con autocompletado, y un administrador puede fusionar duplicados | Mismo patrón que `health_facility` |
| 5 | La auditoría se guarda **un año** | Hace falta una tarea de purga (§7) |
| 6 | **La contraseña que asigna el administrador se cambia en el primer ingreso** | Marca en la base, guard que bloquea el resto del sistema y pantalla intermedia (§5). Tiene efectos sobre las pruebas y la demostración: ver §5.1 |
| 7 | La pantalla de auditoría la ve **solo `suadmin`** | Un `admin` no verá los cambios de rol ni de correo |
| 8 | El distrito es **texto libre** en esta primera versión | No depende del municipio. Se puede endurecer después sin perder lo registrado |
| 9 | Los usuarios **que ya existen no** cambian su contraseña | La marca se enciende solo para los que se creen desde ahora |
| 10 | **El administrador no puede cambiar la contraseña de otra persona.** Solo asigna la inicial al crear la cuenta | Se quita `password` del camino de edición (§5.2). Trae una consecuencia operativa: ver la duda 1 de §10 |
| 11 | La auditoría registra **solo ediciones**, no consultas. Y quién exporta reportes en PDF | Las consultas de datos de víctimas quedan fuera (§7) |
| 12 | Quien olvida su contraseña la recupera **solo** por «¿Olvidaste tu contraseña?» | Sin contraseñas temporales ni ninguna otra vía: nadie más que la persona recupera el acceso (§10) |

### Lo que la decisión 1 cambia en el plan

Que la invitación quede apagada tiene una consecuencia que conviene decir en voz alta: **el
administrador conoce la contraseña de todas las personas**, porque él las asigna. Mientras no exista
forma de cambiarla uno mismo, esa contraseña sigue siendo conocida por un tercero para siempre.

Por eso **el cambio de contraseña desde el perfil (§5) deja de ser una comodidad y pasa a ser
necesario**. Era lo que faltaba en el panel de perfil.

## 2. Verificaciones hechas

**La recuperación de contraseña funciona de punta a punta.** Probada en la base de desarrollo contra
la API real, con el enlace leído del registro del servidor (sin enviar correos): la solicitud
responde 200, el enlace se genera, el token válido cambia la contraseña, se entra con la nueva, la
anterior deja de servir y el token no se puede reutilizar. Las seis comprobaciones pasan.

Dos cosas que descubrí al probarlo:

- **Ningún test cubría el camino feliz.** El E2E `13-edit-and-recovery` solo prueba tokens
  inventados y que la respuesta sea idéntica exista o no la cuenta. Que el restablecimiento
  *funcione* no estaba verificado por nada. Entra en el plan como T-1.
- **La política de contraseñas es inconsistente:** activar cuenta exige 6 caracteres, restablecer
  exige 8, crear usuario 6. Debería ser una sola regla.

**El perfil no permite cambiar la contraseña, y no es solo la pantalla:** `UserService.updateProfile`
no acepta el campo y no existe ningún endpoint para que una persona cambie su propia contraseña. Hoy
solo puede hacerlo el administrador por ella, o pasando por "olvidé mi contraseña".

## 3. Bloque 1 — Arreglos chicos ✅ *Hecho el 2026-11-16*

Commits `95ac41d` y `bccbd9a` en `back-sci`, `bb04bb4` en `front-sci`. Pruebas: **472** unitarias del
backend (eran 452), **623** del front y **42** E2E (era 41), todas en verde.

Dos cosas se hicieron distinto de lo planeado, y a mejor:

- **1.3 no necesitó endpoint de reemplazo nuevo desde cero**: la tabla de usuarios **ya tenía** el
  botón de reenviar invitación, llamando al endpoint público. Borrarlo sin más habría roto una
  función existente, así que se creó `POST /user/:id/resend-activation` (solo admin, identifica por
  id) y se repuntó el botón. La fuga de enumeración se cierra igual, sin perder la función.
- **1.4 arrastró toda la estrategia de Passport**: `TokenValidatorService`, `JwtStrategy`,
  `ITokenStrategy` y `PassportModule` solo existían para `checkToken`. Nada usaba el guard de
  Passport —la autenticación real va por `AuthGuard` con `verifyAccessToken`—, así que se retiró el
  bloque entero en vez de dejar la mitad muerta.

| # | Qué | Dónde |
|:-:|:---|:---|
| 1.1 | **Cambiar el correo de un usuario no activado emite un token nuevo** (lo que revoca el anterior) y envía la invitación a la dirección nueva, en una transacción. Si el usuario ya está activo, no toca ningún token. Cierra el agujero: hoy, corregir un correo mal escrito deja el enlace anterior válido 72 h en el buzón equivocado | `UserService.update` |
| 1.2 | **Retirar la opción de reenviar activación del login**: el enlace, la pantalla `/resend-activation` y sus rutas | `login.component.html`, `features/auth/resend-activation/`, `auth.routes.ts` |
| 1.3 | **Eliminar el endpoint público `POST /resend-activation`**. Además de quedar sin uso, filtra qué correos están registrados (404 si no existe, 400 si ya está activa, 200 si lo manda). Se cierra la fuga borrándolo, no parcheándolo | `auth.controller.ts`, `auth.service.ts`, DTO |
| 1.4 | **Eliminar `POST /checkToken` y `POST /expiredToken`**. El front no los usa y reciben el token por *query string*, donde queda escrito en los registros de acceso del proxy | `auth.controller.ts`, `auth.service.ts` |
| 1.5 | **Una sola política de contraseña**, mínimo 8 caracteres, en un decorador compartido para activación, restablecimiento, creación y cambio propio | DTOs de `auth` y `user` |
| 1.6 | **Dejar de filtrar el mensaje crudo de PostgreSQL al cliente.** `handlerError` devuelve `error.detail` tal cual: el usuario ve *"Key (email)=(juan@x.com) already exists"*, en inglés, con nombres de columnas y el correo de otra persona. Traducir por código de error a mensajes en español | `handlerError.utils.ts` |
| 1.7 | **Identidad del usuario en los registros**: añadir `userId` y `role` a cada petición registrada, junto al `traceId` que ya existe. Sin esto no se puede auditar nada a partir de los logs | `logger.config.ts`, `TraceIdMiddleware` |
| 1.8 | **Niveles de registro correctos**: 4xx → `warn`, 5xx → `error`. Hoy todo se registra como `error`, incluidos los 401 y 404 que son funcionamiento normal, y los fallos de verdad quedan sepultados | `http-exception.filter.ts` |
| 1.9 | **Completar el enmascarado**. Hoy solo se ocultan `authorization` y `req.body.password`. Añadir `token`, `refreshToken`, `newPassword`, `req.query.token` y `req.headers.cookie` | `logger.config.ts` |
| 1.10 | **`LOG_LEVEL` configurable** por entorno, para subir el detalle sin recompilar | `logger.config.ts`, `.env.example`, `compose.yml` |
| 1.11 | **Rotación de los logs de Docker**: `logging: driver: json-file, max-size: 10m, max-file: 5` en el bloque compartido. Hoy crecen sin límite y pueden llenar el disco del VPS, tumbando las tres copias a la vez | `infra/compose.yml` |

## 4. Bloque 2 — Estados de la cuenta ✅ *Hecho el 2026-11-16*

Commit `06b6364` en `back-sci`. **486** pruebas unitarias y **42** E2E en verde.

El bloque cambió de contenido al revisarlo: lo urgente no era construir la interfaz de activación
(que sigue aplazada, más abajo), sino separar estados que estaban confundidos.

### El problema

`is_active = false` significaba **dos cosas distintas**: una cuenta nueva esperando su invitación y
una cuenta que un administrador suspendió. Y `findOne` no filtra las eliminadas. Con el arreglo 1.1
tal como quedó, corregirle el correo a una cuenta suspendida o eliminada **le habría mandado una
invitación para volver a entrar**; lo mismo el botón de reenviar.

### La regla

"Pendiente de activación" pasa a ser una condición **positiva**, no la ausencia de otra:

```
NOT is_deleted  AND  NOT is_active  AND  activated_at IS NULL
```

`user.activated_at` (migración `UserActivatedAt1794300000000`) guarda cuándo entró en funcionamiento
la cuenta. El relleno sella todas las que hoy funcionan, para que suspender a cualquiera de ellas no
la haga parecer nueva. Reenviar a una cuenta suspendida explica qué hacer en vez de mandar nada; a
una eliminada responde como si no existiera, para no confirmar que ahí hubo alguien.

### Dos huecos más, encontrados al revisar esto

- **Eliminar o suspender a alguien no le quitaba el acceso.** El guard no volvía a mirar el estado de
  la cuenta, así que el token seguía sirviendo hasta expirar (8 h), y la renovación tampoco lo
  miraba: se podía renovar la sesión indefinidamente. Ahora ambos lo comprueban.
- **Un enlace de activación de cuando la cuenta existía seguía activándola** aunque ya estuviera
  eliminada.

### Lo que sigue aplazado

Con la invitación apagada, esto **no se construye todavía**. Queda anotado para el día que se
encienda, porque el análisis ya está hecho:

- Elegir al crear el usuario entre "enviar invitación" y "asignar contraseña ahora".
- `POST /user/:id/resend-activation` solo para admin+, y el botón en la fila del usuario.
- Columna de estado en la tabla de usuarios: Activo / Pendiente con vencimiento / Enlace expirado.
- Decir en el correo cuánto vale el enlace.

Lo que **sí** se mantiene del bloque es 1.1, porque la API todavía admite crear un usuario con
`isActive: false` y conviene que ese camino sea seguro aunque hoy no se use.

## 5. Bloque 3 — Contraseña propia y cambio obligatorio

### Cambiar la contraseña propia

| # | Qué | Detalle |
|:-:|:---|:---|
| 3.1 | `PATCH /user/me/password`, endpoint propio en lugar de sumarlo a `updateProfile` | Recibe la **contraseña actual** y la nueva. Exigir la actual es lo que impide que una sesión robada cambie la contraseña y deje al dueño fuera |
| 3.2 | Verificar la actual, aplicar la política de 1.5 y rechazar que la nueva sea igual a la actual | |
| 3.3 | **Revocar las demás sesiones** al cambiarla, como ya hace el restablecimiento | `refresh_token` → `isRevoked` |
| 3.4 | Registrar el evento en la auditoría (§7), sin ninguna contraseña | |
| 3.5 | Sección "Seguridad" en el perfil: actual, nueva, confirmación, con las reglas a la vista y aviso de que se cerrarán las otras sesiones | `features/profile` |

### Cambio obligatorio en el primer ingreso (decisión 6)

| # | Qué | Detalle |
|:-:|:---|:---|
| 3.6 | Columna `must_change_password` en `user` | Migración, por defecto `false`: los usuarios que ya existen no cambian nada (decisión 9) |
| 3.7 | **Cuándo se enciende**: solo al crear un usuario con contraseña asignada por un administrador | `UserService.createUser`. Ya no hay un segundo caso, porque el administrador no puede cambiar contraseñas (decisión 10) |
| 3.8 | **Cuándo se apaga**: al cambiarla uno mismo (3.1), al restablecerla por "olvidé mi contraseña" y al activar la cuenta | En los tres casos la eligió la persona, no un tercero |
| 3.9 | **Guard que bloquea el sistema** mientras la marca esté encendida, devolviendo un código propio para que el front sepa a dónde llevar | Deja pasar solo `PATCH /user/me/password`, `GET /user/me`, el cierre de sesión, la renovación de token y `/config/public`. Sin el guard, bastaría usar la API para saltarse el cambio y dejar viva la contraseña que conoce el administrador |
| 3.10 | Exponer la marca en el `UserDTO` que devuelve el login | Hoy el DTO no la tiene, y el front la necesita para redirigir |
| 3.11 | Pantalla intermedia obligatoria, sin menú ni forma de escapar salvo cerrar sesión | Reutiliza el formulario de 3.5 |

### 5.1 Lo que el cambio obligatorio rompe si no se planifica

Esto es lo que convierte una función pequeña en media jornada de pruebas rotas:

- **Las 41 pruebas E2E se caen.** `front-sci/e2e/global-setup.ts` crea cinco usuarios por la API como
  administrador y después inicia sesión como ellos. Con la marca encendida por defecto, el guard de
  3.9 los bloquea a todos. La preparación tiene que pedir `mustChangePassword: false`.
- **La demostración queda inservible.** `back-sci/scripts/seed-demo.mjs` crea el personal de
  `demo.emerg-ia.com` con una contraseña compartida y luego entra como ellos. Esos usuarios existen
  justamente para mostrar el sistema con una contraseña conocida: tienen que quedar exentos.
- **El superadministrador inicial queda exento**, a propósito. Su contraseña viene de
  `ADMIN_PASSWORD`, un secreto de despliegue y no una entrega de persona a persona; además el
  seeder y los scripts de datos entran con ella. Ahí la mitigación correcta es rotar el secreto, no
  forzar un cambio que rompería la automatización.

Por eso `CreateUserDto` recibe `mustChangePassword` opcional: **el valor por defecto es `true`**
cuando un administrador asigna la contraseña, y la preparación de pruebas, el seeder de demostración
y el seeder inicial pasan `false` explícitamente.

### 5.2 Quitar al administrador la potestad de cambiar contraseñas (decisión 10)

| # | Qué | Detalle |
|:-:|:---|:---|
| 3.12 | Quitar `password` de `UpdateUserDto` y del camino de edición | `UserService.update` deja de aceptarlo. El administrador sigue asignando la contraseña inicial **al crear** la cuenta, nada más |
| 3.13 | Las sesiones se cierran al cambiar la contraseña uno mismo y al consumir el token de recuperación | Lo segundo ya lo hace; lo primero entra con 3.3 |

Con esto, la contraseña de una persona solo la conocen esa persona y, una única vez, quien creó la
cuenta —y además está obligada a cambiarla en el primer ingreso—. A partir de ahí nadie más la conoce
ni puede reemplazarla.

## 6. Bloque 4 — Marcas de tiempo

El patrón a seguir ya existe en el proyecto: `organization_change`, alimentado por un **trigger en la
base**. Así queda registrado venga el cambio de la web, del traspaso de mando o de la sincronización
del móvil, sin tener que acordarse en cada servicio. Cada migración rellena además lo ya existente.

| # | Qué | Cómo | Tamaño |
|:-:|:---|:---|:-:|
| 4.1 | **Cambios de estado de la emergencia** | Tabla `emergency_state_change` (`emergency_id`, `from_state`, `to_state`, `user_id`, `user_name`, `reason`, `occurred_at`) + trigger `AFTER UPDATE ON emergency` cuando cambia `state`. Relleno: `created_at` como "reportada" y el estado actual con `updated_at`, marcado como aproximado | M |
| 4.2 | **Catálogo de tipos de emergencia** | Tabla `emergency_type` con autocompletado que no distingue tildes ni mayúsculas, más fusión de duplicados para admin+. Migración que mapee los valores de texto libre existentes. Hoy `type` es `varchar(50)` libre y "Incendio" e "incendio forestal" no agrupan | M |
| 4.3 | **Zona administrativa** | `department`, `municipality`, `district` en `emergency`. Departamento con lista fija (son nueve); municipio y distrito con autocompletado que crece solo. El distrito es texto libre, sin depender del municipio (decisión 8) | M |
| 4.4 | **Hora de primer arribo** | `first_arrival_at` en `emergency`, nulable, editable mientras esté abierta, con un botón "Marcar arribo" en el Centro de Mando. Explícito y no derivado: la asignación la hace la central, que no sabe cuándo llegó la unidad | M |
| 4.5 | **Fecha de cada devolución de recursos** | Tabla `resource_return` (`resource_id`, `amount`, `user_id`, `occurred_at`). **Tabla y no columna**: las devoluciones ya pueden ser parciales y varias, así que un solo `returned_at` mentiría | M |
| 4.6 | **Historial de disponibilidad del personal** | Tabla `user_availability_change` + trigger sobre `user.is_operational` | S |

Orden dentro del bloque: **4.1, 4.2 y 4.3 primero**, que son las que más historia pierden cada día.

## 7. Bloque 5 — Auditoría

**Logs y auditoría son cosas distintas.** Los logs sirven para diagnosticar un fallo y desaparecen con
la rotación; la auditoría responde "quién hizo esto" y tiene que durar un año en la base.

Hoy el rastro existe solo para el dominio de la emergencia (bitácora, historial de la organización,
historial de triage). **No hay ninguno de las acciones administrativas.**

| # | Qué | Detalle |
|:-:|:---|:---|
| 5.1 | Tabla `audit_log` | `actor_user_id`, `actor_name`, `action`, `entity`, `entity_id`, `before`, `after` (solo los campos que cambian, nunca contraseñas ni tokens), `ip`, `trace_id`, `occurred_at` |
| 5.2 | Eventos a cubrir: **solo ediciones**, no consultas (decisión 11) | Alta, edición y baja de usuario; cambio de rol; cambio de correo; cambio de contraseña propia; cambio de institución; alta y baja de cargos; e inicios de sesión fallidos |
| 5.2b | **Quién exporta reportes en PDF** (decisión 11) | Las hojas del 201 y del 207 se generan en el navegador, así que no hay petición que registrar: hace falta que la pantalla de impresión avise al abrirse. Es el único caso de "consulta" que se audita, y se registra la intención de exportar, no la lectura de los datos |
| 5.3 | **Purga al año** | Tarea programada que borra lo anterior a 12 meses. Sin ella la tabla crece para siempre |
| 5.4 | Pantalla de auditoría **solo para `suadmin`** (decisión 7) | Filtros por actor, acción y rango de fechas. Un `admin` no la ve, porque la tabla incluye cambios de rol y de correo hechos por otros administradores |
| 5.5 | Documentar el `traceId` para soporte | El filtro de excepciones ya lo devuelve en cada error: basta pedir "dime el código que aparece en el mensaje" para encontrar la petición exacta |
| 5.6 | Registros estructurados | Pasar de interpolar texto (`email=${email}`) a campos, al menos en autenticación y usuarios, para poder filtrar por `userId` o por tipo de evento |

## 8. Pruebas

| # | Qué |
|:-:|:---|
| T-1 | **E2E del camino feliz de la recuperación**: solicitar, usar el enlace, entrar con la nueva, que la anterior falle y que el token no se reutilice. Hoy no lo cubre nada |
| T-2 | E2E del cambio de contraseña desde el perfil, incluido que cierre las otras sesiones |
| T-3 | Unitarias de 1.1 (cambiar el correo revoca el token anterior) y 1.6 (los errores de PostgreSQL no se filtran) |
| T-4 | E2E de que las rutas eliminadas en 1.3 y 1.4 devuelven 404 |
| T-5 | E2E del cambio obligatorio: con la marca encendida el sistema queda bloqueado, y al cambiar la contraseña se desbloquea |
| T-6 | Que la preparación de los E2E y el seeder de demostración sigan funcionando con la marca (§5.1) |

## 9. Orden de ejecución

1. **Bloque 1** completo (arreglos chicos) + T-3 y T-4.
2. **Bloque 3**, primero el cambio propio (3.1–3.5) con T-1 y T-2, y después el obligatorio
   (3.6–3.11) con T-5 y T-6. Va pronto porque la decisión 1 lo vuelve necesario, y en ese orden
   porque el obligatorio se apoya en el formulario del propio.
3. **Bloque 4**, empezando por 4.1, 4.2 y 4.3.
4. **Bloque 5** (auditoría).

El bloque 3 es el único con riesgo de arrastre: el guard de 3.9 toca todas las peticiones, así que
conviene correr las tres suites completas antes de darlo por cerrado.

## 10. Dudas que quedan

Ninguna. Todas quedaron resueltas (decisiones 6 a 12).

**Decisión 12:** el único camino para quien olvida su contraseña es **«¿Olvidaste tu contraseña?»**,
con el enlace que llega por correo. No habrá contraseñas temporales ni ninguna otra vía: nadie más
que la persona vuelve a tener acceso a su cuenta.

Consecuencia a tener presente: quien no pueda recibir correo no tiene forma de recuperar el acceso, y
el procedimiento pasa a ser crear una cuenta nueva. Por eso conviene que los correos de alta sean
buzones que la persona realmente revise.
