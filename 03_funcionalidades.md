# 03 — Funcionalidades

Catálogo funcional por módulo.

> La inspección del 2026-10-14 ([07](./07_inspeccion.md)) detalla brechas de autorización, conformidad con el manual y bugs que matizan varios ✅ de esta tabla, en especial S1-S8 y las secciones 3.1-3.3. El informe es la fuente vigente del estado real.

Estado conocido al 2026-10-14, **antes** de la inspección exhaustiva:
- ✅ implementado y cubierto por pruebas;
- 🟡 implementado parcial o con pendientes conocidos;
- ⏳ pendiente;
- 🔒 en pausa por decisión del proyecto.

## 1. Acceso y usuarios

| Funcionalidad | Web | Backend | Notas |
|:---|:---:|:---:|:---|
| Login con JWT y refresh token rotativo | ✅ | ✅ | Login limitado a 5 intentos/min por IP |
| Activación de cuenta por correo / reenvío | ✅ | ✅ | Configurable con `REQUIRE_EMAIL_ACTIVATION` |
| Recuperación de contraseña | ⏳ | ⏳ | Solo existe la plantilla de correo (`sendPasswordRecoveryEmail`); no hay endpoint ni pantalla |
| Perfil propio (datos y foto) | ✅ | ✅ | |
| Gestión de usuarios (alta, edición, rol, grado, estado de guardia) | ✅ | ✅ | Permisos por rol en [05](./05_reglas_y_roles.md) |

## 2. Emergencias y mando

| Funcionalidad | Web | Backend | Notas |
|:---|:---:|:---:|:---|
| Alta georreferenciada (incidente, PC, Área de Espera) | ✅ | ✅ | Google Maps |
| Máquina de estados p → a → f / c, con motivo de cancelación | ✅ | ✅ | Reapertura solo para manager+ |
| Bloqueo de edición en emergencias finalizadas o canceladas | ✅ | ✅ | |
| Evaluación inicial de riesgos | ✅ | ✅ | |
| Organigrama SCI (35 cargos, 5 niveles) | ✅ | ✅ | |
| El creador queda como CI inicial automáticamente | ✅ | ✅ | |
| Asignación y traspaso formal de mando del CI | ✅ | ✅ | Transaccional, registrado en la bitácora |
| Dashboard con mapa, KPIs y feed | ✅ | ✅ | El KPI de víctimas aún no usa el resumen de triage |

## 3. Formularios SCI

| Funcionalidad | Web | Backend | Notas |
|:---|:---:|:---:|:---|
| **F201** crear, editar y finalizar, con snapshot del organigrama | ✅ | ✅ | La creación por HTTP estaba rota hasta el 2026-10-03 (BUG-006) |
| **F201** hoja imprimible / PDF | ✅ | — | Se genera en el navegador |
| **F207** planillas con código correlativo | ✅ | ✅ | |
| **F207** registro de víctima y triage START/SALT | ✅ | ✅ | Idempotente ante reintentos |
| **F207** re-evaluación e historial por víctima | ✅ | ✅ | Log inmutable |
| **F207** resumen por clasificación vigente | ✅ | ✅ | |
| **F207** hoja imprimible / PDF | ✅ | — | |
| **F211** registro de recursos y personal | ⏳ | ⏳ | Etapa 3 |
| Validación de los campos contra el formato oficial del formulario | 🟡 | 🟡 | **[A confirmar]** qué versión oficial de los formularios se sigue |

## 4. Inventario y recursos

| Funcionalidad | Web | Backend | Notas |
|:---|:---:|:---:|:---|
| Catálogo de equipos (alta, edición y baja para manager+) | ✅ | ✅ | El stock disponible no se edita a mano |
| Despacho a una emergencia con control de stock | ✅ | ✅ | Bloqueo pesimista ante despachos simultáneos |
| Devolución parcial o total (CI o manager+) | ✅ | ✅ | Permitida aun con la emergencia finalizada |

## 5. Bitácora y voz

| Funcionalidad | Web | Móvil | Backend | Notas |
|:---|:---:|:---:|:---:|:---|
| Entradas manuales | ✅ | ⏳ | ✅ | |
| Entradas automáticas (estados, formularios, despachos, mando) | ✅ | ⏳ | ✅ | |
| Grabación y subida de nota de voz | — | ⏳ | ✅ | Solo móvil, por decisión del proyecto |
| Reproducción de notas de voz | ✅ | ⏳ | ✅ | |
| **Speech-to-text** de las notas de voz | — | ⏳ | ⏳ | **Núcleo del proyecto. No implementado** |
| **Extracción de datos con IA** hacia 207, recursos, etc. | — | ⏳ | ⏳ | **Núcleo del proyecto. No implementado** |
| Revisión humana de lo que propuso la IA | ⏳ | ⏳ | ⏳ | **[A confirmar]** quién confirma y dónde |

## 6. Tiempo real, notificaciones y offline

| Funcionalidad | Web | Móvil | Backend | Notas |
|:---|:---:|:---:|:---:|:---|
| Eventos en vivo por emergencia (WebSocket) | ✅ | ⏳ | ✅ | Funciona desde el 2026-10-03 (BUG-005) |
| Notificaciones in-app | 🟡 | ⏳ | ✅ | La web no muestra todavía un centro de notificaciones |
| Push FCM | — | ⏳ | ✅ | |
| Captura offline y sincronización por lotes | — | ⏳ | 🟡 | Falta propagar el rol en la sincronización |
| Manejo de conflictos (emergencia cerrada mientras se estaba offline) | — | ⏳ | ✅ | |

## 7. Análisis e histórico

| Funcionalidad | Estado | Notas |
|:---|:---:|:---|
| Histórico de incendios forestales y de interfaz | 🔒 | Esperando requisitos |
| Probabilidad o riesgo por zona y fecha | 🔒 | Esperando requisitos |
| Reportes y estadísticas operativas | ⏳ | **[A confirmar]** si entran en alguna etapa |
