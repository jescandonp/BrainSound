# BrainSound I1 — SPEC de fundación PWA y sesión vertical

- **SPEC maestra:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Iteración:** I1 de 5
- **Estado:** propuesta; requiere aprobación explícita antes de validar o ejecutar su plan
- **Precedencia:** la SPEC maestra gobierna el producto; esta SPEC únicamente concreta I1

## 1. Objetivo

Construir la primera vertical verificable de BrainSound: una PWA personal, gratuita y sin cuentas que presenta Deep Focus, Creatividad y Relajación, inicia con un clic un perfil sintético local distinto por modo, permite pausar, reanudar y detener, mide el tiempo con reloj monotónico y vuelve a abrir el shell sin red después de una primera visita exitosa.

I1 valida los límites técnicos de React, Web Audio, Service Worker, Chromium y WebKit. No pretende completar el catálogo, el motor híbrido, la personalización ni la instalación transaccional del MVP.

## 2. Supuestos explícitos y dependencias

- La SPEC maestra fue aprobada el 2026-08-07 y conserva autoridad sobre alcance, arquitectura y criterios del MVP.
- I1 comienza en un repositorio documental sin aplicación creada.
- El audio sintético de I1 es la degradación segura permanente del producto, no un prototipo desechable.
- I1 no requiere activos de audio externos ni decisiones de licencia.
- I1 no depende de otra iteración; I2 depende de sus contratos de modo y catálogo de respaldo.
- No se despliega ni publica la aplicación dentro de I1.

## 3. Alcance funcional

### Incluido

1. Shell React/Vite con identidad visual Energía cromática.
2. Tres acciones accesibles: Deep Focus, Creatividad y Relajación.
3. Un perfil de síntesis local por modo, con ganancia acotada.
4. Reproductor compacto con nombre, estado, tiempo, pausa, reanudación y detención.
5. Temporizador libre basado en tiempo monotónico, no en conteo de intervalos.
6. Manifiesto PWA, icono propio y Service Worker para el shell y assets compilados.
7. Mensaje responsable: herramienta de bienestar/productividad, no tratamiento médico.
8. Pruebas unitarias, integración y E2E en Chromium y WebKit.
9. Auditoría automatizada que rechaza solicitudes a orígenes externos durante la vertical.

### Excluido

- Las quince experiencias finales y cualquier activo redistribuible.
- Intensidades suave, media y profunda.
- Mezcla híbrida, fundidos y cambio continuo entre experiencias.
- Calibración, recomendación, búsqueda, favoritos, historial, progreso y ajustes.
- IndexedDB, exportación/importación y descarga transaccional.
- Backend, cuentas, sincronización, telemetría y funciones sociales.

## 4. Comportamiento y estados

El estado de sesión admite únicamente `idle`, `playing` y `paused`.

```text
idle --start(mode)--> playing --pause--> paused --resume--> playing
  ^                         |                |
  +---------stop------------+------stop------+ 
```

- `start` solo cambia a `playing` después de que Web Audio confirme el inicio.
- `pause` suspende audio y reloj de manera coherente.
- `resume` continúa ambos desde el tiempo acumulado.
- `stop` detiene fuentes, limpia la experiencia activa y vuelve el reloj a cero.
- Un modo no soportado o un perfil ausente se rechaza antes de abrir Web Audio.
- Si Web Audio no puede comenzar, la interfaz conserva `idle` y muestra un error recuperable sin afirmar que existe reproducción.

## 5. Arquitectura y contratos

```text
App Shell
  ├─ HomeScreen ── ModeDefinition
  └─ PlayerPanel ── SessionController ── AudioPort ── WebAudioSynth
                                └──────── ElapsedTimer

Service Worker ── Cache Storage ── shell compilado
```

La UI depende de `SessionController` y `AudioPort`, nunca de nodos Web Audio. El reloj recibe un `Clock` inyectable. El registro del Service Worker se limita al build de producción.

Contrato de dominio:

```ts
export type BrainSoundMode = 'deep-focus' | 'creativity' | 'relaxation';
export type SessionStatus = 'idle' | 'playing' | 'paused';

export interface FallbackExperience {
  readonly id: string;
  readonly mode: BrainSoundMode;
  readonly title: string;
  readonly profile: {
    readonly carrierHz: number;
    readonly overtoneHz: number;
    readonly movementHz: number;
    readonly filterHz: number;
    readonly masterGain: number;
  };
}
```

Los tres perfiles tendrán IDs estables: `fallback-deep-focus`, `fallback-creativity` y `fallback-relaxation`. `masterGain` debe ser mayor que cero y máximo `0.12`.

Parámetros aprobables de I1:

| ID | Carrier | Overtone | Movement | Filter | Master gain |
|---|---:|---:|---:|---:|---:|
| `fallback-deep-focus` | 110 Hz | 220 Hz | 0.18 Hz | 900 Hz | 0.08 |
| `fallback-creativity` | 146.83 Hz | 293.66 Hz | 0.12 Hz | 1400 Hz | 0.07 |
| `fallback-relaxation` | 98 Hz | 196 Hz | 0.07 Hz | 700 Hz | 0.06 |

## 6. Base técnica y comandos

- Node.js 24.18.1 LTS.
- TypeScript 7.0.2 en modo estricto.
- React/React DOM 19.2.8.
- Vite 8.1.5.
- Vitest 4.1.10 y cobertura V8.
- Playwright Test 1.61.1.
- Web Audio, Service Worker y Cache Storage nativos.

```powershell
npm ci
npm run dev
npm run lint
npm run typecheck
npm run test
npm run test:coverage
npm run build
npm run validate:catalog
npm run test:e2e
npm run verify
```

## 7. Estructura autorizada

```text
public/                         Manifest, icono y Service Worker
src/app/                       Composición de la aplicación
src/audio/                     Puerto, plan puro y adaptador Web Audio
src/catalog/                   Catálogo sintético de respaldo
src/features/home/             Selección de modo
src/features/player/           Controlador y reproductor
src/features/timer/            Reloj monotónico y refresco visual
src/pwa/                       Registro del Service Worker
src/shared/domain/             Modos y tipos compartidos
src/styles/                    Tokens y estilos globales
e2e/                           Recorridos Playwright
```

## 8. Estilo de código

- Exportaciones nombradas y archivos de dominio en kebab-case.
- Efectos del navegador detrás de adaptadores inyectables.
- Funciones puras para validación, plan de síntesis y cálculo temporal.
- Componentes React sin acceso directo a nodos Web Audio o Cache Storage.
- Ningún `any`, conversión insegura o promesa ignorada sin `void` explícito.

Ejemplo de límite:

```ts
export interface AudioPort {
  start(experience: FallbackExperience): Promise<void>;
  pause(): Promise<void>;
  resume(): Promise<void>;
  stop(): Promise<void>;
}
```

## 9. Estrategia de pruebas

- **Unitarias:** modos, perfiles, límites de ganancia, plan de síntesis y reloj monotónico.
- **Integración:** controlador con dobles de audio y reloj; transiciones y notificaciones.
- **E2E:** selección por ratón/teclado, reproducción declarada, pausa/reanudación/detención, manifiesto y recarga offline.
- **Privacidad:** ninguna solicitud a orígenes distintos del host local.
- **Cobertura:** mínimo global de 80% y mínimo de 90% para dominio, temporizador y controlador.

## 10. Manejo de errores

- Audio bloqueado: conservar `idle`, explicar cómo reintentar y no simular reproducción.
- Perfil inválido: rechazar antes de crear nodos.
- Service Worker no disponible: mantener la sesión online y mostrar estado no instalable sin bloquear audio.
- Caché fallida: no eliminar una caché funcional durante la misma activación.
- Error inesperado: mensaje recuperable y detalle técnico solo en consola local.

## 11. Criterios de aceptación

1. Los tres modos aparecen con texto, no solo color.
2. Cada modo inicia su perfil sintético estable mediante una sola acción.
3. Pausa y reanudación coordinan audio y reloj.
4. Una hora simulada produce exactamente 3.600 segundos, sin deriva acumulativa.
5. Detener vuelve a `idle` y tiempo cero.
6. La aplicación instalada recarga el shell sin red tras una visita completa.
7. La vertical no emite solicitudes externas.
8. Funciones principales utilizables con teclado y movimiento reducido respetado.
9. `npm run verify` y E2E Chromium/WebKit finalizan correctamente.
10. No se agrega ninguna funcionalidad excluida.

## 12. Límites de trabajo

### Siempre hacer

- Escribir prueba fallida antes de lógica de dominio o coordinación.
- Mantener la degradación sintética independiente del catálogo futuro.
- Ejecutar verificación antes de cerrar la iteración.

### Preguntar primero

- Agregar dependencias de producción o cambiar contratos públicos de I1.
- Cambiar navegadores, estados de sesión o límites de ganancia.
- Publicar o desplegar.

### Nunca hacer

- Copiar audio, marca, interfaz o contenido de Brain.fm.
- Presentar la experiencia como tratamiento médico.
- Introducir almacenamiento personal, telemetría o red externa en I1.

## 13. Preguntas abiertas

No hay preguntas funcionales abiertas para planificar I1. Los valores concretos de los tres perfiles están definidos en esta SPEC y el plan no puede alterarlos sin nueva aprobación.

## 14. Compuerta SDD

No se puede validar, ejecutar ni considerar vigente el plan de I1 hasta que el usuario apruebe explícitamente esta SPEC. Una modificación de alcance actualiza primero la SPEC maestra y luego esta SPEC.
