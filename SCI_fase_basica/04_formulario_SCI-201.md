# 04 – Formulario SCI-201: Resumen del Incidente

## 1. Ficha

| Atributo | Valor |
|---|---|
| Código | SCI-201 |
| Nombre | Resumen del Incidente |
| Páginas | 4 (1: datos generales · 2: mapa/croquis · 3: resumen de acciones · 4: organización actual) |
| Lo llena | El **Comandante del Incidente** o la persona que designe |
| Cuándo | Desde el **Periodo Inicial** y se **sigue actualizando en todos los periodos operacionales** |
| Obligatorio | Completo **al transferir el mando**, junto con una explicación verbal detallada |
| Se usa en | Briefing / Informe del Incidente, Reunión de Evaluación (Inicial y de cada PO), Reunión de Objetivos y Estrategias, preparación de la Reunión de Planificación |
| Distribución | Tras informar al CI, **copia a la Unidad de Documentación** (si está establecida); **se adjunta al Informe Final** |
| Lo usan como base | Unidad de Situación o JSP, para elaborar el PAI |

## 2. Propósito

- Proveer al CI, al Staff de Comando y a las Secciones la **información básica sobre la situación del incidente y los recursos empeñados**.
- Servir como **registro permanente** de toda la respuesta.
- Ser el **punto de partida** de otros formularios y documentos del SCI.
- Página 2 (Mapa o Croquis) puede convertirse luego en el **Mapa de Situación del Incidente**.
- Página 3 (Resumen de las Acciones) sirve como **seguimiento continuo** de las acciones de respuesta.

> **Importante:** si se pasa a un periodo operacional, se debe **mantener actualizada** la información del SCI-201 y seguirlo usando.

## 3. Campos e instrucciones

### Página 1 – Datos generales

| N° | Campo | Instrucción del manual | Tipo de dato sugerido | Obligatorio |
|---|---|---|---|---|
| 1 | Nombre del incidente | Nombre asignado al incidente | Texto corto | Sí |
| 2 | Fecha y hora de preparación | Día, mes, año; hora en reloj de 24 h | Fecha-hora | Sí |
| 3 | Lugar del incidente | Lo más exacto posible: ubicación geográfica y referencias | Texto + coordenadas (lat/long) | Sí |
| 4 | Evaluación inicial | Naturaleza del incidente, amenazas, área afectada y aislamiento | Texto largo (o 4 subcampos) | Sí |
| 5 | Objetivo(s) inicial(es) | Cortos, concisos y claros, para el manejo inicial, tomando en cuenta los recursos disponibles en la escena | Lista de objetivos | Sí (≥1) |
| 6 | Ubicación del PC | Ubicación geográfica del PC instalado | Texto + coordenadas | Sí |
| 7 | Ubicación del E | Ubicación geográfica del lugar o lugares seleccionados como Áreas de Espera | Lista de ubicaciones | Recomendado |
| 8 | Ruta de ingreso | Vías, avenidas, calles, etc. para que los recursos solicitados ingresen de forma segura | Texto / polilínea | Recomendado |
| 9 | Ruta de egreso | Vías para que los recursos salgan de la escena de forma segura | Texto / polilínea | Recomendado |
| 10 | Mensaje general de seguridad | Basado en las amenazas de la evaluación inicial: peligros conocidos y acciones a ejecutar por el personal | Texto largo | Sí |
| 11 | Comandante del incidente (nombre y apellidos) | Nombre del CI o de los integrantes del Comando Unificado | Persona(s) | Sí (se repite en todas las páginas) |

### Página 2 – Mapa o croquis

| N° | Campo | Instrucción | Tipo sugerido |
|---|---|---|---|
| 12 | Mapa o croquis | Dibujar un croquis o usar una imagen georreferenciada o un mapa con: **perímetro de seguridad, ubicación de los recursos, instalaciones del incidente, el norte** y otra información especial | Mapa con capas (polígono de perímetro, marcadores de instalaciones y recursos) o imagen adjunta |

Usar los **símbolos SCI** para mapas (ver `07_…anexos.md`): PC, E, B, C, H, H1, ACV, peligros, etc.

### Página 3 – Resumen de las acciones

| N° | Campo | Instrucción | Tipo sugerido |
|---|---|---|---|
| 13 | Resumen de las acciones | En forma **cronológica** (fecha y hora), **comenzando con el primer recurso que llega**: estrategias y tácticas utilizadas, cualquier problema específico y en qué áreas se presenta | Bitácora: lista de entradas {fecha-hora, descripción, estrategia/táctica, problema, área} |

### Página 4 – Organización actual

| N° | Campo | Instrucción | Tipo sugerido |
|---|---|---|---|
| 1-3 | Nombre del incidente, fecha y hora de preparación | Se repiten en esta página | — |
| 14 | Organización actual | Graficar las estructuras y **mantenerlas en forma secuencial** (cada cambio de estructura queda registrado) | Organigrama versionado (árbol de posiciones con persona asignada) |

### Nota del manual
Este formulario se usa en incidentes de **corta y larga duración**. Si se requiere, se agregan **páginas adicionales numeradas** (especialmente para los puntos 12, 13 y 14).

## 4. Mapeo con los 8 pasos del primer respondedor

| Paso | Campos que se llenan |
|---|---|
| 1. Informar arribo | 1 (nombre del incidente), 2, entrada inicial en 13 |
| 2. Asumir el mando y establecer el PC | 11, 6, entrada en 13, raíz del organigrama en 14 |
| 3. Evaluar la situación | 3, 4 |
| 4. Establecer el perímetro | 4 (aislamiento), 12 (perímetro), 10 |
| 5. Objetivos | 5 |
| 6. Estrategias y tácticas | 13 |
| 7. Recursos e instalaciones | 7, 8, 9, 12; recursos → SCI-211 |
| 8. Transferir el mando | Formulario completo y actualizado + 14 |

## 5. Modelo de datos sugerido

```yaml
SCI201:
  id: uuid
  incidente_id: uuid
  version: int                 # cada actualización/transferencia genera una versión
  fecha_hora_preparacion: datetime
  lugar:
    descripcion: text
    lat: decimal
    lng: decimal
    referencias: text
  evaluacion_inicial:
    naturaleza: text
    amenazas: text
    area_afectada: text
    aislamiento: text
  objetivos_iniciales: [ {id, texto, especifico, observable, alcanzable, evaluable, estado} ]
  ubicacion_pc: {descripcion, lat, lng}
  areas_espera: [ {nombre, descripcion, lat, lng} ]
  ruta_ingreso: {descripcion, geometria?}
  ruta_egreso: {descripcion, geometria?}
  mensaje_seguridad: text
  comandantes: [persona_id]    # 1 = CI, >1 = Comando Unificado
  mapa:
    imagen_adjunta?: file
    perimetro?: geojson_polygon
    elementos: [ {tipo_simbolo, etiqueta, lat, lng, fecha_hora?} ]
    norte_indicado: bool
  resumen_acciones: [ {fecha_hora, descripcion, estrategia?, tactica?, problema?, area?, registrado_por} ]
  organizacion_actual: snapshot_organigrama   # árbol de posiciones activas y asignaciones
  paginas_adicionales: [file]
  entregado_a_documentacion: {fecha_hora, recibido_por}?
```

## 6. Reglas de negocio derivadas

1. Un incidente **no puede existir sin un SCI-201** abierto desde su creación.
2. Campos 1, 2, 3, 4, 5, 6, 10 y 11 deben estar completos antes de permitir una **transferencia de mando** o la **Reunión de Evaluación**.
3. Cada **transferencia de mando** genera una nueva versión del 201 firmada por quien entrega y quien recibe.
4. El campo 13 es **solo de agregado** (append-only); las entradas no se borran, se corrigen con una nueva entrada.
5. El campo 14 se guarda como **historial secuencial**: cada activación/desactivación de posición crea un snapshot.
6. Al pasar a Periodo Operacional, el 201 **sigue vivo** y alimenta al SCI-202 (objetivos, mensaje de seguridad, organización).
7. Toda hora en formato de 24 h; fechas día/mes/año.
8. Al cerrar el incidente, el 201 final se adjunta al **Informe Final** y se archiva en Documentación.
