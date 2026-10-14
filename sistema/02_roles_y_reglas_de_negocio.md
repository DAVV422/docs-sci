# 02. Roles, Permisos y Reglas de Negocio del SCI

Este documento establece las definiciones oficiales de roles, jerarquías de acceso y reglas de negocio del Sistema de Comando de Incidentes. Es fundamental para que los desarrolladores de Frontend y Mobile implementen los controles de navegación, botones de acción y validaciones de interfaz de forma alineada con la seguridad del backend.

---

## 1. Jerarquía de Roles de Sistema

> **Convención de nomenclatura**: los roles se definen, almacenan y transmiten siempre en **minúsculas** (`suadmin`, `admin`, `manager`, `advanced`, `basic`), tanto en la base de datos como en payloads JSON, query params (`role=admin`) y en el Frontend (`SystemRole` en `auth.model.ts`).

El sistema cuenta con **5 roles de acceso técnico global**:

```
           ┌──────────────────────┐
           │       suadmin        │ (Super Administrador: control total, incluye cuentas admin/suadmin)
           └──────────┬───────────┘
                      ▼
           ┌──────────────────────┐
           │        admin         │ (Gestión administrativa: NO ve ni gestiona cuentas suadmin)
           └──────────┬───────────┘
                      ▼
           ┌──────────────────────┐
           │       manager        │ (Gestión operativa: disponibilidad y grado de manager/advanced/basic)
           └──────────┬───────────┘
                      ▼
           ┌──────────────────────┐
           │       advanced       │ (Gestión Operativa de Incidentes sin tocar Usuarios)
           └──────────┬───────────┘
                      ▼
           ┌──────────────────────┐
           │        basic         │ (Operador de Terreno / Acceso acotado a su Emergencia)
           └──────────────────────┘
```

**Diferencia clave `admin` vs. `suadmin`**: un `admin` gestiona usuarios `admin`, `manager`, `advanced` y `basic` exactamente igual que un `suadmin`, pero **no ve, no crea, no edita, no cambia de estado ni elimina cuentas `suadmin`**, ni puede promover a nadie a ese rol. El backend oculta las cuentas `suadmin` en los listados y responde `403 Forbidden` (o `404` al consultar por ID) ante cualquier intento de un `admin` de interactuar con ellas. Ningún rol puede ejecutar acciones de gestión (editar, cambiar estado, eliminar, asignar grado) sobre su propia cuenta desde el módulo de usuarios; esos cambios se hacen desde "Mi Perfil".

### Tabla Comparativa de Capacidades Globales

| Funcionalidad | `suadmin` | `admin` | `manager` | `advanced` | `basic` |
|:---|:---:|:---:|:---:|:---:|:---:|
| **Crear Usuarios Nuevos** (`POST /api/user`) | ✅ | ✅ (no `suadmin`) | ❌ | ❌ | ❌ |
| **Eliminar Usuarios** (`DELETE /api/user/:id`) | ✅ | ✅ (no `suadmin`) | ❌ | ❌ | ❌ |
| **Editar datos administrativos** (`PATCH /api/user/:id`: email, rol, grado, estado, disponibilidad) | ✅ | ✅ (no `suadmin`) | ❌ | ❌ | ❌ |
| **Cambiar disponibilidad operativa** (`PATCH /api/user/status/:id`) | ✅ | ✅ (no `suadmin`) | ✅ (no `admin`/`suadmin`) | ❌ | ❌ |
| **Actualizar grado institucional** (`PATCH /api/user/grades`) | ✅ (todos) | ✅ (no `suadmin`) | ✅ (solo `manager`/`advanced`/`basic`) | ❌ | ❌ |
| **Ver Lista Global de Usuarios** (`GET /api/user`, `GET /api/user/admin/all`) | ✅ (ve todo) | ✅ (no ve `suadmin`) | ✅ (no auditoría) | ❌ | ❌ |
| **Crear / Editar Catálogo de Equipos** (`Equipment`) | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Crear Emergencias** (`POST /api/emergency`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Ver Emergencias Globales** (`GET /api/emergency`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Administrar Cargos del Organigrama** (`Charge`) | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 2. Estado de Cuenta (`isActive`) vs. Disponibilidad Operativa (`isOperational`)

Estos dos campos son independientes entre sí y no deben confundirse en la interfaz:

- **`isActive` — Estado de Cuenta / Acceso**: define si el usuario tiene permitido **iniciar sesión** en la plataforma. `isActive = false` significa cuenta suspendida o pendiente de activación por correo; el backend rechaza el login con `401 Unauthorized`. Se gestiona exclusivamente desde el modal de edición de `admin`/`suadmin` (`PATCH /api/user/:id`). En la vista de Gestión de Usuarios, **solo el `suadmin` filtra y visualiza la columna de estado sobre este campo**.
- **`isOperational` — Disponibilidad Operativa / Guardia**: define si el usuario está **en servicio** (de guardia, disponible para ser convocado a emergencias) o **fuera de servicio** (descanso). Un usuario con `isOperational = false` **puede iniciar sesión con normalidad**; solo no está disponible para nuevas convocatorias. Se cambia con `PATCH /api/user/status/:id`, acción visible para `admin` y `manager` en la tabla. Para `admin` y `manager`, la columna y el filtro de estado de la vista de usuarios operan sobre este campo (no ven `isActive`).
- **Eliminación (Soft Delete)**: `DELETE /api/user/:id` marca `isDeleted = true`. El usuario no existe operativamente y sus credenciales quedan invalidadas.
- **Regla clave**: un usuario fuera de servicio (`isOperational = false`) **SÍ puede ser asignado a una emergencia** y operar con normalidad según su cargo SCI, igual que uno inactivo por guardia.

---

## 3. Permisos Comunes Dentro de una Emergencia

Cualquier usuario con sesión activa que esté **asignado al personal de una emergencia** (`AttendEntity` activa), sin importar si su rol de sistema es `basic`, `advanced`, `manager`, `admin` o `suadmin`, cuenta con los siguientes permisos en ese incidente:

1. **Bitácora y Notas de Voz (`Action`)**: Puede registrar acciones, eventos y notas de voz con audio en la línea de tiempo del incidente.
2. **Registro de Víctimas (`Victim` & `Registration`)**: Puede registrar los datos básicos de personas lesionadas y asentar evaluaciones de triage START/SALT dentro de las planillas del Formulario 207 activas.
3. **Gestión de Recursos (`Resource`)**: Puede solicitar la asignación y recepción de equipamiento para el frente de trabajo en el que se desempeña.

---

## 4. Reglas Específicas del Sistema de Comando de Incidentes (SCI)

### 4.1 Comandante del Incidente (CI) y Elevación Táctica

El **Comandante del Incidente** es la máxima autoridad táctica en el lugar del suceso.
- **Responsabilidad del Formulario 201**: Es el único responsable de iniciar, actualizar y finalizar el Formulario 201 (Resumen del Incidente).
- **Evaluación Inicial**: Es el encargado de registrar la evaluación preliminar de riesgos y recursos requeridos (`InitialAssessment`).
- **Facultades sobre el Formulario 207**: Puede crear y finalizar planillas del Formulario 207.
- **Asignación de Personal**: Puede convocar brigadistas y asignar sus cargos SCI.
- **Regla de Elevación Táctica**: Si un usuario cuyo rol de sistema es `basic` asume el cargo SCI de *Comandante del Incidente*, obtiene **permisos de gestión total sobre esa emergencia específica** (equivalentes a `manager`/`admin`), pero única y exclusivamente limitados al perímetro de ese incidente.

---

### 4.2 Regla Especial del Creador de la Emergencia

1. Al crearse una nueva emergencia (`POST /api/emergency`), el usuario creador se establece automáticamente como el **Comandante de Incidente inicial**.
2. En ese momento, es el único operador con facultad para añadir y convocar personal al incidente hasta que designe o traspase el mando de CI a otra persona.
3. **Persistencia del Privilegio**: Incluso si el creador traspasa el cargo de Comandante de Incidente a otro oficial, **el creador conserva de forma permanente el permiso de añadir personal a la emergencia** por su condición de originador del incidente.

---

### 4.3 Líder de la Unidad Médica (`medical_unit_leader`)

- En el organigrama SCI, la Unidad Médica pertenece a la Rama de Servicios de la Sección de Logística.
- El usuario con cargo SCI de *Líder de Unidad Médica* tiene **permiso expreso para crear y finalizar el Formulario 207** (`POST /api/emergency/:id/form207`), aliviando la carga administrativa del Comandante de Incidente en eventos con víctimas en masa.
- Los roles de sistema `manager`, `admin` y `suadmin` también pueden crear el Formulario 207.

---

### 4.4 Asignación de Personal y Rol por Defecto

- Cuando se asigna personal a una emergencia (`POST /api/attend`) sin especificar un cargo particular en la interfaz, el sistema asigna automáticamente el cargo **Equipo de Ataque (`EQ-ATK` / `attack_team`)**, correspondiente al Nivel 5 del organigrama.
- **Unicidad de Mando**: En una misma emergencia **solo puede existir un Comandante del Incidente activo a la vez**. Para nombrar a un nuevo CI, se debe desactivar o reasignar la asignación anterior.

---

### 4.5 Máquina de Estados de la Emergencia y Bloqueo de Edición (`assertEditable`)

Las emergencias operan bajo una máquina de estados estricta:

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Pendiente   │ ───►  │    Activa    │ ───►  │  Finalizada  │ (Solo Lectura)
│     (p)      │       │     (a)      │       │     (f)      │
└──────┬───────┘       └──────┬───────┘       └──────────────┘
       │                      │
       └──────────────┬───────┴─────────────► ┌──────────────┐
                      │                       │  Cancelada   │ (Solo Lectura)
                      └─────────────────────► │     (c)      │
                                              └──────────────┘
```

- **Regla de Cierre Inmutable (`assertEditable`)**: Cuando una emergencia pasa al estado **Finalizada (`f`)** o **Cancelada (`c`)**:
  - Se bloquea la creación y edición de Formularios 201 y 207.
  - Se bloquea la asignación de personal (`attend`) y despacho de recursos (`resource`).
  - Se bloquea el registro de triage de nuevas víctimas.
  - Cualquier intento de modificación retornará `HTTP 400 Bad Request` indicando que la emergencia se encuentra cerrada.
