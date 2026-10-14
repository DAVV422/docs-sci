# 01 — Visión y objetivos

## El problema

En una emergencia intervienen varias unidades: ambulancias, bomberos, paramédicos, rescatistas, defensa civil y otras. Si no hay una estructura común de mando y registro:
- se duplican esfuerzos y se pierde información;
- la documentación oficial (los formularios SCI) se completa tarde, en papel o de memoria;
- el personal en terreno no tiene tiempo ni manos libres para escribir;
- muchas veces no hay señal.

El **Sistema de Comando de Incidentes (SCI)** resuelve la parte organizativa: un único Comandante del Incidente (CI), una estructura de cargos estándar y formularios normalizados. Lo que falta es una herramienta que permita **aplicar el SCI en tiempo real sin que registrar la información compita con atender la emergencia**.

## La propuesta

Una plataforma con dos frentes que comparten el mismo backend:

| Frente | Para quién | Uso principal |
|:---|:---|:---|
| **Web — Centro de Despacho** | Operadores de central, comandantes, jefaturas | Registrar y revisar todos los datos de la emergencia con detalle: formularios, organigrama, triage, recursos e inventario |
| **App móvil — Terreno** | Personal en la escena | Capturar información con **voz**, funcionar **sin conexión** y sincronizar al recuperar señal |

### El diferenciador: la voz como entrada principal en terreno

En la app móvil, el brigadista graba notas de voz. Un pipeline de IA:
1. transcribe el audio a texto (*speech-to-text*);
2. guarda la transcripción como entrada de **bitácora**;
3. **extrae datos estructurados** del relato (por ejemplo víctimas y su clasificación de triage, recursos usados, novedades) y los propone para las tablas correspondientes: Formulario 207, recursos y otras.

Esto convierte un relato hablado en registros SCI sin que el usuario tenga que escribir bajo presión. **Es el núcleo del valor del proyecto.**

## Usuarios

- **Personal de terreno** (rol `basic`/`advanced`): bomberos, paramédicos y rescatistas. Trabajan con estrés, guantes, poca luz o sol directo, y conectividad intermitente.
- **Comandante del Incidente (CI)**: máxima autoridad táctica de una emergencia concreta. Puede tener rol global `basic`, pero dentro de *su* emergencia tiene facultades de gestión.
- **Líder de la Unidad Médica**: gestiona el Formulario 207 en eventos con víctimas.
- **Operador de central / jefatura** (rol `manager`): gestión operativa global e inventario.
- **Administradores** (`admin`, `suadmin`): usuarios, cargos SCI y configuración.

## Objetivos

1. **Unidad de mando y trazabilidad:** cada emergencia tiene un único CI activo, un organigrama SCI y un traspaso de mando formal y registrado.
2. **Formularios SCI digitales:** el 201 (resumen del incidente) y el 207 (registro de víctimas) se generan durante la emergencia y se pueden imprimir o exportar.
3. **Registro inmutable:** cada decisión, despacho, evaluación médica y cambio de estado queda en la bitácora con fecha, hora y responsable.
4. **Captura por voz asistida por IA (móvil):** reducir al mínimo la escritura en terreno.
5. **Operación sin conexión (móvil):** capturar localmente y sincronizar sin perder ni duplicar datos.
6. **Visión común en tiempo real:** la central y el terreno ven el mismo estado de la emergencia a medida que cambia.
7. **Control de inventario:** saber qué equipamiento está disponible, dónde está desplegado y cuándo vuelve.
8. **Base para análisis futuro:** histórico de incidentes, en especial forestales y de interfaz, para estimar zonas y fechas de riesgo (etapa posterior).

## Principios de diseño acordados

Detallados en `back-sci/docs/sistema/contexto_diseno_sistema.md`:
- usabilidad bajo estrés por encima de la estética;
- selección antes que tecleo;
- valores por defecto inteligentes;
- alto contraste en modo claro y oscuro;
- los colores de triage reservados exclusivamente al triage;
- el estado de sincronización siempre visible.
