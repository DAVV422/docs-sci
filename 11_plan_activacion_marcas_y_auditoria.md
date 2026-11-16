# 11 — Plan: activación de cuentas, marcas de tiempo y auditoría (2026-11-16)

Tres trabajos independientes que se pueden hacer en cualquier orden. Antes del plan, las respuestas a
las preguntas sobre la activación, porque cambian lo que hay que construir.

## 1. Cómo funciona hoy la activación

**¿Cuándo se envía el correo de activación?** Al **crear el usuario**, y solo si queda **inactivo**.

**¿Y cuándo queda inactivo?** Depende de `REQUIRE_EMAIL_ACTIVATION`. Hoy está en `false` en las tres
copias del servidor, así que los usuarios se crean **activos** y **no se envía ningún correo de
activación**. Nunca. Ese es el detalle que faltaba: el enlace del login *¿Aún no activas tu cuenta o
expiró tu enlace?* lleva a una pantalla que, con la configuración actual, siempre responderá que la
cuenta ya está activa.

El administrador puede forzar el flujo mandando `isActive: false` al crear, pero desde la pantalla de
usuarios no hay forma de elegirlo.

**¿Se puede reenviar?** Sí, pero es **autoservicio**: el propio usuario entra a `/resend-activation` y
escribe su correo. El administrador no lo tiene en la pantalla de usuarios, y es él quien va a
recibir la queja.

**¿Se puede corregir el correo?** Sí, el administrador ya puede cambiarlo con `PATCH /user/:id`.
**Pero no invalida el token anterior ni manda el correo a la dirección nueva.**

### Aquí hay un agujero de seguridad

Si el administrador escribe `juan@gmai.com` en lugar de `juan@gmail.com`, y alguien controla ese
dominio, esa persona recibe un enlace **válido durante 72 horas** para fijar la contraseña de una
cuenta real del sistema. Corregir el correo hoy **no revoca ese enlace**. El dueño del buzón
equivocado sigue pudiendo entrar.

La buena noticia: **el mecanismo de revocación ya existe**. `AuthTokenService.createToken` llama a
`revokeUnusedTokens(userId, tipo)` antes de emitir uno nuevo, así que cada token nuevo mata al
anterior. Lo único que falta es **disparar la emisión cuando cambia el correo**.

**Respuesta corta a "dime si se puede soportar eso": sí, todo, y la mayor parte de la maquinaria ya
está escrita.** Falta conectarla y darle pantalla.

## 2. Parte A — Activación de cuentas

| # | Qué | Dónde | Tamaño |
|:-:|:---|:---|:-:|
| A-1 | **Elegir el modo al crear el usuario**, en lugar del interruptor global: "enviar invitación por correo" o "asignar contraseña ahora". El segundo sigue siendo necesario: hay personal de emergencia sin correo que revise | Front (`user-form-modal`) + DTO del back | M |
| A-2 | **Cambiar el correo de un usuario no activado** emite un token nuevo (lo que revoca el anterior) y envía la invitación a la dirección nueva, todo en una transacción. Si el usuario ya está activo, el cambio de correo no toca ningún token | `UserService.update` | **S** |
| A-3 | **Reenviar la invitación desde la pantalla de usuarios**: `POST /user/:id/resend-activation`, solo admin+. Distinto del público: identifica por id, no por correo, y puede responder con detalle porque quien llama ya está autenticado | Back + front | S |
| A-4 | **Columna de estado en la tabla de usuarios**: "Activo", "Pendiente de activación" con la fecha de vencimiento del enlace, "Enlace expirado". Hoy la tabla no muestra nada de esto y el administrador no puede saber quién no entró nunca | Front (`users-table`) | M |
| A-5 | Acciones contextuales en la fila: "Reenviar invitación" y "Corregir correo" visibles **solo** si está pendiente | Front | S |
| A-6 | **Cerrar la fuga de enumeración** del endpoint público: hoy responde 404 "Usuario no encontrado" si el correo no existe y 400 "ya está activa" si lo está, así que sirve para averiguar qué correos están registrados. Debe responder siempre igual, como ya hace `forgot-password` | `AuthService.resendActivationEmail` | **S** |
| A-7 | Decir en el correo **cuánto vale el enlace** (72 h por defecto) y qué hacer si expiró | Plantilla de correo | S |

Nada de esto necesita migración de base de datos: `auth_token` ya tiene tipo, vencimiento, `is_used`
y el hash del token.

## 3. Parte B — Marcas de tiempo

Lo urgente no es construir reportes, es **empezar a guardar los momentos**, porque la historia no se
reconstruye después. El patrón a seguir ya existe en el proyecto: `organization_change`, que se
alimenta con un **trigger en la base**. Así queda registrado venga el cambio de la web, del traspaso
de mando o de la sincronización del móvil, sin tener que acordarse en cada servicio.

| # | Qué | Cómo | Tamaño |
|:-:|:---|:---|:-:|
| B-1 | **Cambios de estado de la emergencia** (reportada → activa → finalizada / cancelada) | Tabla `emergency_state_change` (`emergency_id`, `from_state`, `to_state`, `user_id`, `user_name`, `reason`, `occurred_at`) + trigger `AFTER UPDATE ON emergency` cuando cambia `state`. Relleno inicial: `created_at` como "reportada"; el estado actual con `updated_at` como aproximación, marcado como tal | **M** |
| B-2 | **Hora de primer arribo a la escena** | Columna `first_arrival_at` en `emergency`, nulable y editable mientras esté abierta, con un botón "Marcar arribo" en el Centro de Mando. Explícito y no derivado: la primera asignación se registra desde la central, que no sabe cuándo llegó la unidad | M |
| B-3 | **Fecha de cada devolución de recursos** | Tabla `resource_return` (`resource_id`, `amount`, `user_id`, `occurred_at`). **No una columna**: las devoluciones ya pueden ser parciales y varias, así que un solo `returned_at` mentiría. De paso queda el rastro de quién devolvió qué | M |
| B-4 | **Zona administrativa** (municipio / distrito) | Columnas en `emergency` con autocompletado que se alimenta de lo ya registrado, el mismo patrón que `health_facility`: se crea sobre la marcha y no distingue tildes ni mayúsculas | M |
| B-5 | **Catálogo de tipos de emergencia** | Tabla `emergency_type` con el mismo patrón, y migración que mapee los valores de texto libre existentes. Hoy `type` es `varchar(50)` libre con sugerencias solo en el front, así que "Incendio" e "incendio forestal" no agrupan y cualquier estadística por tipo sale mal | **M** |
| B-6 | **Historial de disponibilidad del personal** | Tabla `user_availability_change` + trigger sobre `user.is_operational` | S |

Prioridad dentro de la parte B: **B-1, B-5 y B-4 primero**. Son las que más datos pierden cada día y
las más baratas. B-6 es la que menos urge.

## 4. Parte C — Logs y auditoría

Son **dos cosas distintas** y conviene no mezclarlas:

- **Logs**: para diagnosticar un fallo. Viven en Docker, rotan y desaparecen.
- **Auditoría**: para responder "quién hizo esto". Tiene que estar en la base y poder consultarse
  meses después.

Hoy existe lo segundo solo para el dominio de la emergencia (la bitácora, `organization_change`, el
historial de triage). **No hay ningún rastro de las acciones administrativas**: quién cambió un rol,
quién cambió un correo, quién borró a un usuario, quién cambió el nombre de la institución. Con tres
instituciones y datos reales, eso es justo lo que se necesita después de un incidente.

### Lo que está mal hoy

| # | Problema | Por qué importa | Tamaño |
|:-:|:---|:---|:-:|
| C-1 | **Las peticiones se registran sin saber quién las hizo.** Hay `traceId` pero no `userId` | Sin esto no se puede auditar nada a partir de los logs | **S** |
| C-2 | **Se filtra el mensaje crudo de PostgreSQL al cliente.** `handlerError` devuelve `error.detail` tal cual: el usuario ve *"Key (email)=(juan@x.com) already exists"*, en inglés y exponiendo nombres de columnas y datos de otros | Mal mensaje para el usuario y revelación de información. Hay que traducirlo a mensajes en español por tipo de error | **S** |
| C-3 | **Todo se registra como `error`**, incluidos los 401 y 404 que son parte del funcionamiento normal | Los fallos de verdad quedan sepultados entre el ruido. 4xx → `warn`, 5xx → `error` | **S** |
| C-4 | **Enmascarado incompleto**: solo se ocultan `authorization` y `req.body.password`. Quedan al descubierto `token`, `refreshToken`, las contraseñas nuevas del restablecimiento y el token que `checkToken` recibe por *query string* | Credenciales en texto plano en los registros del servidor | **S** |
| C-5 | **Nivel de log fijo**, sin `LOG_LEVEL` | No se puede subir el detalle para diagnosticar sin recompilar | S |
| C-6 | **Los logs de Docker crecen sin límite**: el `compose.yml` no tiene `logging` | Llenan el disco del VPS y tumban las tres copias a la vez. Junto con el monitoreo de disco pendiente, es el riesgo operativo más tonto que queda | **S** |
| C-7 | Los mensajes se arman interpolando texto (`email=${email}`) en lugar de campos estructurados | Con campos se puede filtrar por `userId` o por tipo de evento; con texto, solo buscar a ciegas | M |
| C-8 | Los correos se escriben en claro en los logs | Dato personal en los registros. Para auditoría puede justificarse, pero debe ser una decisión tomada, no un descuido | S |

### Lo que hay que añadir

| # | Qué | Tamaño |
|:-:|:---|:-:|
| C-9 | **Tabla `audit_log`** para acciones administrativas: `actor_user_id`, `actor_name`, `action`, `entity`, `entity_id`, `before`, `after` (solo los campos que cambian, sin contraseñas), `ip`, `trace_id`, `occurred_at`. Eventos a cubrir: alta/edición/baja de usuario, cambio de rol, cambio de correo, activación, reenvío de invitación, cambio de institución, alta/baja de cargos, e inicios de sesión fallidos | **L** |
| C-10 | **Pantalla de auditoría** para admin+, filtrable por actor, acción y rango de fechas | M |
| C-11 | Documentar el `traceId` para soporte: el filtro de excepciones ya lo devuelve en cada error, así que basta pedirle al usuario "dime el código que aparece en el mensaje" para encontrar la petición exacta en los logs | S |

## 5. Orden sugerido

**Primero, lo que tapa riesgos y cuesta poco** (todo S, se puede hacer de una vez):

1. **A-2** (cambiar el correo revoca el enlace anterior) — es el agujero de seguridad.
2. **C-6** (rotación de logs) — evita llenar el disco del VPS.
3. **C-2** (no filtrar mensajes de PostgreSQL) — seguridad y mensajes decentes.
4. **C-1, C-3, C-4** (usuario en los logs, niveles correctos, enmascarado completo) — sin esto, nada
   de lo demás es auditable.
5. **A-6** (cerrar la enumeración de correos).

**Segundo, las marcas de tiempo**, porque cada día que pasa es historia perdida: **B-1, B-5, B-4**, y
después B-2, B-3, B-6.

**Tercero, la activación completa**: A-1, A-3, A-4, A-5, A-7. Es un bloque coherente que conviene
hacer junto y probar de punta a punta con correos reales.

**Cuarto, la auditoría de verdad**: C-9, C-10, C-11, y C-7 de paso.

## 6. Decisiones que hacen falta antes de empezar

1. **¿Se activa la invitación por correo?** Con Resend ya funcionando, la recomendación es que el
   administrador **elija por usuario** (A-1) en vez de un interruptor global, porque habrá personal
   sin correo. Si se prefiere simple, basta poner `REQUIRE_EMAIL_ACTIVATION=true` y que todos entren
   por invitación.
2. **¿Qué pasa con los usuarios ya creados?** Hoy están todos activos con contraseñas asignadas por
   el administrador. Se pueden dejar así, o forzar que fijen la suya en el próximo ingreso.
3. **Zona administrativa (B-4): ¿hasta qué nivel?** Departamento, municipio y distrito, o solo
   municipio. Depende de los reportes territoriales que se quieran.
4. **Catálogo de tipos (B-5): ¿quién lo administra?** Lo más práctico es que crezca solo con
   autocompletado, como los centros de salud, y que un administrador pueda fusionar duplicados.
5. **¿Cuánto se guarda la auditoría?** Sin un plazo definido, `audit_log` crece para siempre. Un año
   es lo habitual para este tipo de sistema.
