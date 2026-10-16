# Plataforma SCI — Documentación general del proyecto

Documentación transversal de la plataforma de gestión de emergencias basada en el **Sistema de Comando de Incidentes (SCI)**. Describe qué es el proyecto, qué busca, qué incluye cada etapa y cómo encajan sus partes.

La documentación técnica detallada de cada aplicación sigue viviendo en su repositorio:
- `back-sci/docs/`: endpoints, reglas de implementación, especificaciones por fase.
- `front-sci/docs/`: reglas de UI, design system y especificaciones por fase.

| Documento | Contenido |
|:---|:---|
| [01 — Visión y objetivos](./01_vision_y_objetivos.md) | Problema, usuarios, objetivos y propuesta de valor |
| [02 — Alcance y etapas](./02_alcance_y_etapas.md) | Qué entra en cada etapa y qué queda fuera |
| [03 — Funcionalidades](./03_funcionalidades.md) | Catálogo funcional por módulo y su estado conocido |
| [04 — Arquitectura](./04_arquitectura.md) | Componentes, tecnologías, flujo de datos, offline e IA |
| [05 — Reglas de negocio y roles](./05_reglas_y_roles.md) | Roles del sistema, cargos SCI y permisos por acción |
| [06 — Decisiones y preguntas abiertas](./06_preguntas_abiertas.md) | Decisiones tomadas y lo que aún falta definir |
| [07 — Inspección (2026-10-14)](./07_inspeccion.md) | Qué cumple, qué funciona y qué falta; backlog priorizado |
| [08 — Reportes (propuesta)](./08_reportes_propuesta.md) | Reportes y estadísticas recomendados, internos y públicos, y los datos que requieren |
| [09 — Despliegue](./09_despliegue.md) | VPS con HTTPS (Caddy), servicio del backend, front, respaldos y restauración, migraciones, actualización y monitoreo |
| [sistema/](./sistema/README.md) | Documentación técnica unificada (antes duplicada en back y front) |
| [SCI — Fase básica](./SCI_fase_basica/00_INDICE.md) | Referencia oficial: *Manual de Campo SCI* (USAID/OFDA/LAC, 2013) resumido para el software |

> **Estado (2026-10-14):** actualizado con las decisiones del responsable del proyecto. Lo que sigue marcado **[A confirmar]** está listado en [06](./06_preguntas_abiertas.md). La plataforma se usará en **Bolivia** y sigue el *Manual de Campo SCI* de USAID/OFDA/LAC.
