# SCI – Fase Básica: Base de conocimiento para un Sistema de Gestión

> Fuente: *Manual de Campo SCI* – Curso Intermedio de Sistema de Comando de Incidentes (USAID/OFDA/LAC), 3.ª edición, junio 2013 (Rev. 08-2013).
> Este conjunto de archivos extrae y reorganiza el contenido del manual para que sirva como especificación funcional de un software de gestión de incidentes basado en el SCI.

## Contenido

| Archivo | Contenido |
|---|---|
| `01_conceptos_y_principios.md` | Definición del SCI, principios, 8 funciones, niveles de la organización, tipos de recursos, PAI y periodos |
| `02_fases_y_flujo_operativo.md` | Fases del SCI (Preparación → Desmovilización), los 8 pasos del primer respondedor, transferencia de mando, la "P" de la planificación, reuniones y briefings |
| `03_estructura_y_roles.md` | Organigrama completo, relaciones de reporte y responsabilidades de cada cargo |
| `04_formulario_SCI-201.md` | Resumen del Incidente: propósito, campos, reglas y modelo de datos |
| `05_formulario_SCI-207.md` | Registro de Pacientes Atendidos: propósito, campos, reglas y modelo de datos |
| `06_formulario_SCI-211.md` | Registro y Control de Recursos: propósito, campos, reglas y modelo de datos |
| `07_instalaciones_comunicaciones_anexos.md` | Instalaciones, Puesto de Comando, tarjetas T, símbolos de mapa, texto claro, cambio de estado y liberación de recursos, comunicado de prensa, formularios relacionados |
| `08_diseno_sistema_gestion.md` | **Propuesta** (no es texto del manual): entidades, máquinas de estado, flujos, permisos, reglas de negocio y validaciones derivadas del manual |

## ¿Qué se entiende aquí por "fase básica"?

El manual define que **siempre existe un Periodo Inicial** (de 1 a 4 horas) en el que el primer respondedor asume el mando y gestiona la respuesta con tres formularios: **SCI-201, SCI-207 y SCI-211**. Al cerrar ese periodo, la **Reunión de Evaluación** (basada en esos tres formularios) decide si se pasa a un Periodo Operacional (12–24 h), que ya requiere el PAI escrito (SCI-202 + 205 + 206 + 215 + 214).

Por eso la fase básica de un sistema de gestión SCI queda acotada así:

- Activación y registro del incidente.
- Asunción del mando y establecimiento del Puesto de Comando (PC).
- Evaluación inicial, objetivos, estrategias y tácticas iniciales (SCI-201).
- Registro y control de recursos (SCI-211).
- Registro de pacientes atendidos o trasladados (SCI-207).
- Activación modular de la estructura (CI, Staff, Secciones…).
- Transferencia de mando.
- Reunión de Evaluación → decisión de pasar a Periodo Operacional o cerrar.
- Desmovilización y cierre documental.

## Glosario de siglas (usadas en el manual)

| Sigla | Significado |
|---|---|
| SCI | Sistema de Comando de Incidentes |
| CI | Comandante del Incidente |
| CU | Comando Unificado |
| PC | Puesto de Comando |
| E | Área de Espera |
| ACV | Área de Concentración de Víctimas |
| B / C / H / H1 | Base / Campamento / Helibase / Helipunto |
| PAI | Plan de Acción del Incidente |
| PO | Periodo Operacional |
| COE | Centro de Operaciones de Emergencia |
| OFS | Oficial de Seguridad |
| OFIP | Oficial de Información Pública |
| OFE | Oficial de Enlace |
| JSO | Jefe de la Sección de Operaciones |
| JSP | Jefe de la Sección de Planificación |
| JSL | Jefe de la Sección de Logística |
| JSAF | Jefe de la Sección de Administración y Finanzas |
| EAE | Encargado del Área de Espera |
| EACV | Encargado del Área de Concentración de Víctimas |
| CROP / CR | Coordinador de Rama de Operaciones |
| CROA | Coordinador de Rama de Operaciones Aéreas |
| SD / SG | Supervisor de División / Supervisor de Grupo |
| LEI / LFT | Líder de Equipo de Intervención / Líder de Fuerza de Tarea |
| EH / EHP | Encargado de Helibase / Encargado de Helipunto |
| LUREC | Líder de la Unidad de Recursos |
| LUSIT | Líder de la Unidad de Situación |
| LUDOC | Líder de la Unidad de Documentación |
| LUDES | Líder de la Unidad de Desmovilización |
| TESP | Técnicos Especialistas |
| CRS | Coordinador de la Rama de Servicios |
| CRAP | Coordinador de la Rama de Apoyo |
| LUCOM | Líder de la Unidad de Comunicaciones |
| TCOM | Técnico en Comunicaciones |
| LUMED / UM | Líder de la Unidad Médica / Unidad Médica |
| LUAL | Líder de la Unidad de Alimentación |
| LUSUM | Líder de la Unidad de Suministros |
| RREQ | Responsable de Requisiciones |
| RR&D | Responsable de Recepción y Distribución |
| LUINS | Líder de la Unidad de Instalaciones |
| RSINS | Responsable de Seguridad de las Instalaciones |
| RMINS | Responsable de Mantenimiento de Instalaciones |
| EBAS / ECAM | Encargado de Base / Encargado de Campamento |
| LUAT | Líder de la Unidad de Apoyo Terrestre |
| REQ | Responsable de Equipos |
| LUT | Líder de la Unidad de Tiempos |
| LUP | Líder de la Unidad de Proveeduría |
| LUCO | Líder de la Unidad de Costos |
