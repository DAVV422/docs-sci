# 05 – Formulario SCI-207: Registro de Pacientes Atendidos

## 1. Ficha

| Atributo | Valor |
|---|---|
| Código | SCI-207 |
| Nombre | Registro de Pacientes Atendidos |
| Lo llena | **Líder de la Unidad Médica (LUMED)** o **Encargado del Área de Clasificación del ACV** (en el ACV, el EACV lleva el registro) |
| Lugar de registro | **ACV** (Área de Concentración de Víctimas) o **UM** (Unidad Médica) |
| ¿Acompaña al PAI? | **No** |
| Se usa en | Informe del Incidente (briefing inicial), **Reunión de Evaluación** (inicial y de cada PO); se recopila actualizado antes de la Reunión de Planificación |
| Distribución | **Todos los formularios llenos y en original se entregan a la Unidad de Documentación** |
| Relación | Complementa al **SCI-206 (Plan Médico)** cuando la Unidad Médica atiende pacientes |

## 2. Propósito

Llevar el **registro y control de los pacientes atendidos** en el ACV o en la Unidad Médica durante el incidente, y de su **traslado a una institución de asistencia médica**.

> Distinción importante:
> - En el **ACV** (Sección de Operaciones) se registran las **víctimas del incidente**.
> - En la **UM** (Sección de Logística) se atiende al **personal de respuesta** (y, excepcionalmente, a civiles en apoyo a Operaciones).
> El sistema debe distinguir ambos orígenes con el campo "Lugar de registro".

## 3. Campos e instrucciones

### Encabezado

| N° | Campo | Instrucción | Tipo sugerido | Obligatorio |
|---|---|---|---|---|
| 1 | Nombre del incidente | Nombre del incidente | Referencia al incidente | Sí |
| 2 | Lugar de registro ("Registrado en") | **ACV o UM** | Enum {ACV, UM} + instalación específica | Sí |

### Detalle (una fila por paciente)

| N° | Campo | Instrucción | Tipo sugerido | Obligatorio |
|---|---|---|---|---|
| 3 | Nombre y apellidos del paciente | Nombres y apellidos completos | Texto (admitir "No identificado" + descriptor) | Sí |
| 4 | Sexo | Masculino o Femenino | Enum {M, F} (considerar "No determinado") | Sí |
| 5 | Edad | Edad del paciente | Entero (admitir edad estimada) | Sí |
| 6 | Clasificación | Por código de colores, gravedad u otros. El formulario usa **Rojo · Amarillo · Verde · Negro** | Enum {ROJO, AMARILLO, VERDE, NEGRO} | Sí |
| 7 | Lugar de traslado | Nombre de la instalación de asistencia médica | Referencia a instalación médica (catálogo del SCI-206) | Si fue trasladado |
| 8 | Trasladado por | Ambulancia, aeronave, embarcación, etc. | Enum {AMBULANCIA, AEREO, EMBARCACION, OTRO} + recurso del SCI-211 | Si fue trasladado |
| 9 | Fecha y hora | Día, mes, año; hora en reloj de 24 h | Fecha-hora | Sí |

### Pie

| N° | Campo | Instrucción | Tipo sugerido |
|---|---|---|---|
| 10 | Preparado por | Nombres y apellidos del Encargado del Área de Clasificación del ACV o del Líder de la UM | Persona + posición |

## 4. Clasificación (triage) usada en el formulario

| Color | Significado habitual en triage* |
|---|---|
| Rojo | Prioridad 1 – atención inmediata |
| Amarillo | Prioridad 2 – atención diferible |
| Verde | Prioridad 3 – lesiones leves |
| Negro | Fallecido / no recuperable |

\* El manual solo indica "código de colores, gravedad u otros" y lista los cuatro colores; los significados son la convención estándar de triage y deben validarse con el protocolo médico local. El SCI-206 (Plan Médico) usa solo Rojo, Amarillo y Verde para la derivación.

## 5. Áreas del ACV (contexto)

El ACV se divide en 4 áreas: **Clasificación → Estabilización → Transporte → Morgue**. El SCI-207 se llena en el área de **Clasificación** y se completa con los datos de traslado (área de Transporte). Si se establece una **morgue**, se coordina con las autoridades pertinentes (pacientes clasificados Negro).

## 6. Modelo de datos sugerido

```yaml
SCI207_Registro:
  id: uuid
  incidente_id: uuid
  lugar_registro: ACV | UM
  instalacion_id: uuid          # el ACV o la UM concreta
  preparado_por: persona_id
  posicion_preparador: EACV | ENC_CLASIFICACION_ACV | LUMED
  pacientes: [Paciente]
  entregado_a_documentacion: {fecha_hora, recibido_por}?

Paciente:
  id: uuid
  nombre_completo: text         # o "NN" + descripción
  identificado: bool
  sexo: M | F | ND
  edad: int
  edad_estimada: bool
  clasificacion: ROJO | AMARILLO | VERDE | NEGRO
  historial_clasificacion: [ {fecha_hora, clasificacion, por} ]   # re-triage
  area_actual: CLASIFICACION | ESTABILIZACION | TRANSPORTE | MORGUE | TRASLADADO | ALTA
  traslado:
    lugar_destino_id: instalacion_medica_id?
    medio: AMBULANCIA | AEREO | EMBARCACION | OTRO
    recurso_id: recurso_sci211_id?     # vínculo al vehículo registrado
  fecha_hora: datetime
  es_personal_respuesta: bool          # true típicamente si lugar_registro = UM
```

## 7. Reglas de negocio derivadas

1. Cada paciente tiene **fecha y hora** obligatoria en 24 h.
2. Si existe **lugar de traslado**, también debe existir **trasladado por** (y viceversa).
3. El **lugar de traslado** debe elegirse de las instalaciones de asistencia médica definidas en el SCI-206 (o agregarse al catálogo).
4. El **medio de transporte** debería vincularse a un recurso registrado en el SCI-211 (ambulancia, aeronave…), lo que permite cruzar el estado del recurso con el traslado.
5. Solo pueden registrar: EACV / Encargado del Área de Clasificación (para ACV) y LUMED (para UM). Si esas posiciones no están activadas, lo asume quien tenga la función (en última instancia el CI o el JSO).
6. Totales automáticos por color, por destino y por medio → insumo directo para el **Informe de Situación** de la Reunión de Evaluación y para el **comunicado de prensa** (resultados operacionales).
7. Pacientes Negro → disparar la coordinación de morgue y autoridades.
8. Los datos de pacientes son **sensibles**: control de acceso restringido (CI, JSO, JSP, LUMED, EACV, LUDOC) y no publicarlos sin autorización del CI (el CI autoriza toda información que sale del incidente).
9. Al cierre, el original se entrega a **Documentación**.
