# 04 — Arquitectura

## Componentes

```
┌─────────────────────────┐      ┌─────────────────────────────┐
│  front-sci (web)        │      │  App móvil (Etapa 2)        │
│  Angular 19 · Signals   │      │  Flutter (Android e iOS)    │
│  Tailwind · Google Maps │      │  BD local + outbox offline  │
└──────────┬──────────────┘      └──────────────┬──────────────┘
           │ REST/JSON · WebSocket              │ REST · multipart (audio)
           │                                    │ sync/batch · Push FCM
           ▼                                    ▼
┌─────────────────────────────────────────────────────────────┐
│  back-sci — NestJS 10 · TypeORM                             │
│  Auth JWT · Roles + permisos tácticos por emergencia        │
│  Emergencias · Organigrama · F201 · F207 · Inventario       │
│  Bitácora + audios · Notificaciones (Socket.IO + FCM)       │
│  Sincronización offline idempotente · Auditoría             │
└──────────┬───────────────────────────────┬──────────────────┘
           ▼                               ▼
   ┌───────────────┐          ┌──────────────────────────────┐
   │ PostgreSQL    │          │ Archivos de audio en disco   │
   │ (principal)   │          │ data/audios/{user}/{emg}     │
   └───────────────┘          └──────────────────────────────┘
                                          │
                                          ▼  (Etapa 2, no implementado)
                              ┌──────────────────────────────┐
                              │ Pipeline IA                  │
                              │ speech-to-text → extracción  │
                              │ → propuesta de registros     │
                              └──────────────────────────────┘
```

## Repositorios

| Repo | Tecnología | Remoto |
|:---|:---|:---|
| `back-sci` | NestJS 10, TypeORM 0.3, PostgreSQL, Socket.IO, firebase-admin, Pino | GitHub `DAVV422/back-sci` |
| `front-sci` | Angular 19 (standalone, OnPush, Signals), Tailwind 3, `@angular/google-maps`, socket.io-client, Playwright | GitHub `DAVV422/front-sci-emergyai` |
| App móvil | Flutter (Android e iOS) | No existe aún |

La carpeta raíz `C:\Proyectos\SCI` no es un repositorio git. Dónde se versiona esta carpeta `docs/` está pendiente: ver [06](./06_preguntas_abiertas.md). Ambos repos trabajan directo en `main` con Conventional Commits.

## Decisiones técnicas vigentes

- **Esquema de BD:** TypeORM con `synchronize`, sin migraciones versionadas (decisión del proyecto, `back-sci/docs/reglas`).
- **Soft delete universal** vía `BaseEntity`. Logs inmutables (`registration`, `audit_log`) sin `updatedAt`.
- **Permisos en dos capas:** rol global (`basic` a `suadmin`) y cargo SCI por emergencia (CI, Líder Médico, personal asignado) mediante `EmergencyAccessService`. El front replica las reglas solo para mostrar u ocultar acciones; el backend siempre las vuelve a exigir.
- **Tiempo real por invalidación:** el backend emite *qué cambió* después del commit y los clientes vuelven a consultar por HTTP. Ningún dato viaja por el socket sin el serializador de la API.
- **Idempotencia offline:** cada registro creado en cliente lleva `clientGeneratedId` (UUID).
- **Rate limiting por IP:** 300 req/min global y 5-10 req/min en autenticación, configurable con `THROTTLE_LIMIT` y `THROTTLE_AUTH_LIMIT`.
- **Formularios imprimibles** generados en el navegador (vista de impresión / guardar como PDF), sin generación de PDF en el servidor.
- **Almacenamiento de archivos** (audios, imágenes del croquis): debe pasar por una interfaz de storage con un adaptador de disco hoy y uno de S3 mañana. Despliegue en un VPS con HTTPS y respaldos.
- **Secretos:** nunca en commits. Los `.env`, `firebase-service.json` y las claves de API se configuran por entorno.

## Pipeline de voz e IA (diseño acordado a alto nivel)

1. El móvil graba y guarda el audio localmente. Con conexión, lo sube con su `clientGeneratedId`.
2. El backend encola el procesamiento: es asíncrono y el usuario no espera. La transcripción ocurre **al sincronizar**, nunca en el teléfono.
3. **Speech-to-text** mediante un **puerto o interfaz** (`SpeechToTextProvider`) con adaptadores intercambiables. Inicialmente un servicio en la nube aún no elegido. Resultado en `audio.transcription`.
4. **Extracción estructurada**, también detrás de una interfaz: busca datos del **201** (situación, objetivos, acciones) o del **207** (pacientes, clasificación, traslado). Resultado en `audio.nlpExtractedData`.
5. **Propuesta y confirmación:**
   - si no hay datos de formulario, la transcripción se propone como **resumen de acción** y la confirma su autor;
   - si hay datos del 201, los confirma el **CI**;
   - si hay datos del 207, los confirma el **LUMED**, o el CI si no hay LUMED asignado.
6. Al confirmar se crean los registros definitivos. Se marca `audio.processed = true` y se notifica en tiempo real.

Riesgos que siguen abiertos:
- costo por minuto de audio del proveedor;
- privacidad al enviar datos de salud a un tercero en la nube;
- errores de la IA en datos críticos como el triage: mitigados porque siempre confirma una persona.

## Pruebas

| Tipo | Dónde | Cómo se ejecuta |
|:---|:---|:---|
| Unitarias backend (Jest) | `back-sci/src/**/*.spec.ts` | `npm test` |
| E2E backend (supertest) | `back-sci/test/` | `npm run test:e2e` (requiere BD) |
| Unitarias web (Karma/Jasmine) | `front-sci/src/**/*.spec.ts` | `npx ng test --watch=false --browsers=ChromeHeadless` |
| E2E de punta a punta (Playwright) | `front-sci/e2e/` | `pnpm e2e` (usa la base aislada `sci_e2e`) |
