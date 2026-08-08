# BrainSound I3 — SPEC de motor híbrido, intensidad y continuidad

- **SPEC maestra:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Iteración:** I3 de 5
- **Estado:** aprobada explícitamente por el usuario el 2026-08-07; planificación autorizada
- **Dependencias:** I1 completada e I2 completada con catálogo validado

## 1. Objetivo

Incorporar el motor normal de reproducción de BrainSound: mezcla local de capas musicales, ambientales y naturales, tres intensidades, precarga y transiciones continuas. El sintetizador de I1 permanece como degradación segura cuando una experiencia no puede prepararse.

I3 demuestra calidad técnica y continuidad del audio. No introduce todavía personalización persistente, favoritos, progreso ni descarga transaccional.

## 2. Supuestos explícitos

- Solo se reproducen activos aprobados por I2 y verificados antes de llegar al motor.
- Cada manifiesto declara los activos, puntos de loop y parámetros permitidos para sus intensidades.
- La reproducción normal es completamente local; no se transmite audio desde terceros.
- El inicio se mide desde la acción del usuario hasta que el grafo entra en estado audible.
- Los tests automatizados usan `AudioPort`, reloj y decodificador deterministas; la auditoría perceptual es adicional y manual.

## 3. Alcance funcional

### Incluido

1. Decodificación y caché en memoria de buffers locales.
2. Grafo Web Audio con buses por capa, bus maestro, filtro y limitador.
3. Plan de sesión determinista por experiencia e intensidad.
4. Intensidades `soft`, `medium` y `deep` aplicadas mediante parámetros del manifiesto.
5. Inicio, pausa, reanudación, detención y cambio de experiencia.
6. Fundido cruzado de 8 a 12 segundos en cambios de experiencia o intensidad.
7. Precarga acotada de la experiencia actual y la siguiente candidata.
8. Recuperación al sintetizador de I1 ante asset, decodificación o grafo fallido.
9. Métricas locales de diagnóstico en memoria, sin telemetría.

### Excluido

- Calibración y selección personalizada.
- Historial, valoraciones, favoritos y persistencia de sesión.
- Descarga, reanudación, rollback y actualización de catálogo.
- Ecualizador manual o controles avanzados para el usuario.
- Afirmaciones clínicas o parametrización “neural” no sustentada.

## 4. Contratos del motor

```ts
export type Intensity = 'soft' | 'medium' | 'deep';

export interface LayerMix {
  readonly assetId: string;
  readonly gain: number;
  readonly playbackRate: number;
  readonly startOffsetSeconds: number;
}

export interface SessionPlan {
  readonly experienceId: string;
  readonly intensity: Intensity;
  readonly layers: readonly LayerMix[];
  readonly masterGain: number;
  readonly filterHz: number;
  readonly crossfadeSeconds: number;
}

export interface HybridAudioPort {
  prepare(plan: SessionPlan): Promise<void>;
  start(): Promise<void>;
  transition(plan: SessionPlan): Promise<void>;
  pause(): Promise<void>;
  resume(): Promise<void>;
  stop(): Promise<void>;
}
```

Reglas:

- `gain` está entre `0` y `1`; `masterGain` entre `0` y `0.9`.
- `playbackRate` está entre `0.95` y `1.05` para evitar alteraciones notorias.
- `crossfadeSeconds` está entre `8` y `12`.
- Un plan contiene al menos una capa y no repite `assetId`.
- El orden de capas se deriva del manifiesto y es estable para la misma solicitud.
- La intensidad modifica únicamente rangos autorizados por el manifiesto.

## 5. Semántica de intensidad

- **Suave (`soft`):** menor densidad, movimiento y brillo dentro del rango declarado.
- **Media (`medium`):** mezcla de referencia declarada por la experiencia.
- **Profunda (`deep`):** mayor densidad y movimiento, sin superar ganancia ni filtro permitidos.

Cada manifiesto define parámetros concretos para las tres intensidades. El motor no inventa valores cuando faltan: rechaza el plan y activa degradación segura.

## 6. Flujo de sesión y estados

```text
idle → preparing → playing ↔ paused → stopping → idle
              ↘ fallback-playing ↗
playing --transition--> transitioning --success--> playing
                                  --failure--> fallback-playing
```

- `prepare` valida y decodifica antes de declarar reproducción.
- `start` entra en audible con rampa, nunca con salto de ganancia.
- `transition` mantiene el grafo anterior hasta que el nuevo esté preparado.
- Una transición fallida conserva la experiencia anterior cuando siga sana; usa fallback solo si no existe una ruta normal funcional.
- `stop` libera fuentes y referencias a buffers de sesión; no elimina Cache Storage.

## 7. Arquitectura y estructura

```text
SessionController
  → SessionPlanner (puro)
  → AssetBufferRepository
  → HybridAudioEngine
       ├─ layer buses
       ├─ transition coordinator
       ├─ master gain
       └─ limiter → destination
  → FallbackAudioPort (I1)

src/audio/engine/**               Grafo y ciclo de vida
src/audio/planning/**             Planificación pura
src/audio/assets/**               Lectura y decodificación
src/audio/transitions/**          Rampas y coordinación
src/audio/diagnostics/**          Medidas locales
tests/audio/fixtures/**           Buffers sintéticos propios
e2e/audio-session.spec.ts         Recorrido observable
```

El `SessionController` no conoce nodos Web Audio. Los componentes React reciben estado y comandos, no el grafo.

## 8. Base técnica y comandos

I3 usa Web Audio nativo y no incorpora un framework de audio sin aprobación.

```powershell
npm ci
npm run test -- src/audio tests/audio
npm run test:coverage
npm run test:e2e -- e2e/audio-session.spec.ts
npm run verify
```

La medición especializada de fixtures debe producir un artefacto local legible con tiempos de inicio, picos y continuidad de transición.

## 9. Estilo de código

- Planificación pura separada de efectos y reloj inyectable para rampas.
- Nodos creados y liberados por una unidad dueña explícita.
- Parámetros validados antes de tocar Web Audio.
- Errores tipados por fase: `manifest`, `asset`, `decode`, `graph`, `transition`.

```ts
export type AudioFailurePhase = 'manifest' | 'asset' | 'decode' | 'graph' | 'transition';

export interface AudioFailure {
  readonly phase: AudioFailurePhase;
  readonly experienceId: string;
  readonly recoverable: boolean;
  readonly message: string;
}
```

## 10. Estrategia de pruebas

- **Unitarias:** validación de planes, límites de intensidad, selección de capas y rampas.
- **Integración:** controlador con motor determinista; preparar/iniciar/transicionar/fallar/recuperar.
- **Audio especializada:** fixtures sintéticos para detectar clipping, discontinuidad y silencio inesperado.
- **E2E:** iniciar cada modo, cambiar intensidad, cambiar experiencia, pausar y conservar continuidad declarada.
- **Compatibilidad:** Chromium y WebKit automatizados; Edge/Chrome/Safari manuales se cierran en I5.
- **Cobertura:** mínimo 90% en planificación, transición y recuperación; global 80%.

## 11. Manejo de errores

- Asset ausente o hash inválido: no decodificar; conservar audio anterior o activar fallback.
- Decodificación fallida: liberar resultados parciales y registrar fase `decode`.
- Transición fallida: cancelar rampas nuevas y mantener el grafo anterior si es válido.
- Contexto suspendido: solicitar una nueva acción del usuario; no mostrar `playing`.
- Memoria insuficiente: liberar precarga no activa y reintentar una vez; después usar fallback.
- Ningún error transmite datos fuera del dispositivo.

## 12. Criterios de aceptación

1. Las 15 experiencias producen un `SessionPlan` válido para tres intensidades.
2. Una experiencia instalada comienza en máximo 1,5 segundos en los equipos de prueba definidos.
3. Cambiar experiencia o intensidad usa un fundido de 8–12 segundos.
4. Los fixtures no presentan clipping ni silencios inesperados durante transiciones.
5. Pausa y reanudación conservan posición lógica y temporizador.
6. Una transición fallida conserva el audio funcional anterior o entra en fallback.
7. El sintetizador de I1 sigue disponible y desacoplado.
8. No hay solicitudes externas durante reproducción.
9. Diagnósticos permanecen locales y se descartan al recargar.
10. `npm run verify` finaliza correctamente.

## 13. Límites de trabajo

### Siempre hacer

- Validar plan y assets antes de modificar el grafo activo.
- Mantener una ruta de recuperación audible y responsable.
- Probar liberación de fuentes y transiciones interrumpidas.

### Preguntar primero

- Cambiar límites de ganancia, reproducción o fundido.
- Agregar procesamiento no definido, dependencias o métricas persistentes.
- Modificar el contrato de manifiesto aprobado en I2.

### Nunca hacer

- Usar audio no aprobado por I2.
- Descargar audio durante una sesión.
- Presentar parámetros como científicamente validados sin una SPEC y evidencia nuevas.

## 14. Preguntas abiertas

No hay preguntas de producto abiertas. La evaluación perceptual final se realiza con los activos aprobados de I2 y no puede sustituirse por una afirmación automatizada.

## 15. Compuerta SDD

La aprobación explícita quedó registrada el 2026-08-07 y autoriza escribir el plan I3. Ejecutarlo requiere aprobación separada del plan e I2 completada con evidencia.
