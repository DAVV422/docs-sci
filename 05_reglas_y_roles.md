# 05 — Reglas de negocio y roles

Resumen operativo. La fuente detallada es `back-sci/docs/sistema/02_roles_y_reglas_de_negocio.md`.

## Dos capas de permisos

1. **Rol global del sistema.** Lo asigna un administrador y define el acceso técnico.
2. **Cargo SCI dentro de una emergencia.** Se asigna por emergencia mediante el organigrama y otorga facultades **solo en ese incidente**.

| Rol global | Resumen |
|:---|:---|
| `suadmin` | Control total, incluido crear otros administradores |
| `admin` | Usuarios hasta `admin`, cargos SCI, configuración |
| `manager` | Gestión operativa total, inventario, estado de guardia del personal |
| `advanced` | Gestión operativa de incidentes, sin usuarios |
| `basic` | Operador de terreno, acotado a las emergencias donde participa |

## Permisos por acción dentro de una emergencia

"Rol global" significa `manager`, `admin` o `suadmin`.

| Acción | CI activo | Líder Unidad Médica | Personal asignado | Creador | Rol global |
|:---|:---:|:---:|:---:|:---:|:---:|
| Cambiar el estado de la emergencia | ✅ | | | | ✅ |
| Evaluación inicial y Formulario 201 | ✅ | | | | ✅ |
| Convocar o reasignar personal | ✅ | | | ✅ (permanente) | ✅ |
| Asignar CI sin ceremonia | | | | ✅ | ✅ |
| Traspasar el mando de CI | ✅ | | | | ✅ |
| Crear y finalizar planillas F207 | ✅ | ✅ | | | ✅ |
| Registrar triage, bitácora y despachar recursos | ✅ | ✅ | ✅ | | ✅ |
| Ver víctimas de la emergencia | ✅ | ✅ | ✅ | ✅ | ✅ (`advanced`+ también). Un `basic` no asignado no las ve: *decisión del 2026-10-14, pendiente de implementar* |
| Devolver recursos al inventario | ✅ | | | | ✅ |
| Gestionar el catálogo de inventario | | | | | ✅ |

## Reglas clave

- **Un solo CI activo** por emergencia, garantizado con un índice único parcial en la BD.
- El **creador** de la emergencia queda como CI inicial y conserva para siempre la facultad de convocar personal.
- **Elevación táctica:** un usuario `basic` que es CI tiene facultades de gestión solo dentro de esa emergencia.
- **Traspaso de mando:** requiere **F201 y F207 finalizados**, con un flujo de "finalizar todo" previo al traspaso. *Hoy el código solo exige el F201: brecha registrada.*
- **Cierre inmutable:** con la emergencia finalizada o cancelada no se pueden crear ni editar formularios, triage ni bitácora. La devolución de recursos sí se permite.
- **No se puede finalizar** una emergencia con formularios pendientes de finalizar.
- **Triage:** clasificación START/SALT en 4 colores (rojo, amarillo, verde, negro). Cada re-evaluación es un registro nuevo y el estado vigente es el último.
- **Stock:** el disponible solo cambia por despacho o devolución y nunca puede superar el total.
