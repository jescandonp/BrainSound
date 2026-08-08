# BrainSound I4 — SPEC de experiencia personal, temporizadores y persistencia

- **SPEC maestra:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Iteración:** I4 de 5
- **Estado:** propuesta; requiere aprobación explícita antes de crear su plan
- **Dependencias:** I1, I2 e I3 completadas

## 1. Objetivo

Completar la experiencia personal local de BrainSound: calibración inicial, navegación del catálogo, inicio en un clic, tres temporizadores, favoritos, recientes, historial, valoraciones, recomendación explicable, progreso y rachas, todo persistido en el dispositivo sin cuentas ni backend.

I4 no implementa todavía la instalación transaccional del catálogo ni el respaldo JSON final, que corresponden a I5.

## 2. Supuestos explícitos

- Existe un catálogo validado de 15 experiencias y un motor híbrido estable.
- Solo hay un perfil local por navegador/dispositivo.
- Toda preferencia se conserva en IndexedDB; no se usan cookies para datos funcionales.
- La recomendación es determinista, local y explicable; no usa modelos remotos ni telemetría.
- Una “racha” es el número de días calendario locales consecutivos con al menos una sesión completada de 60 segundos.
- Una sesión inferior a 60 segundos queda en historial, pero no incrementa progreso ni racha.

## 3. Alcance funcional

### Incluido

1. Calibración inicial opcional y repetible desde Ajustes.
2. Navegación: Inicio, Explorar, Favoritos, Progreso y Ajustes.
3. Inicio en un clic desde recomendación, modo, experiencia, favorito o reciente.
4. Temporizador libre, cuenta regresiva e intervalos.
5. Favoritos y lista de recientes sin duplicados.
6. Historial local de sesiones y valoración de 1 a 5 al detener/completar.
7. Recomendación local con explicación textual de sus factores.
8. Resumen de minutos por modo, sesiones completadas y racha actual/máxima.
9. Ajustes de intensidad predeterminada, volumen inicial, temporizador y movimiento reducido.
10. Esquema IndexedDB versionado, migraciones y repositorios inyectables.

### Excluido

- Autenticación, perfiles múltiples, sincronización y recomendaciones remotas.
- Notificaciones, calendario, gamificación social o clasificación entre usuarios.
- Descarga/actualización transaccional del catálogo.
- Exportación/importación de datos.
- Edición de metadatos o activos del catálogo.

## 4. Primer uso y calibración

La calibración consta de tres decisiones, todas reversibles:

1. modo principal: Deep Focus, Creatividad o Relajación;
2. intensidad predeterminada: suave, media o profunda;
3. duración habitual: libre, 25, 45, 60 o 90 minutos.

El usuario puede omitirla; en ese caso se usa Deep Focus, intensidad media, temporizador libre y volumen inicial `0.7`. Movimiento reducido toma inicialmente la preferencia del sistema operativo. La interfaz explica que estas preferencias solo se guardan en el dispositivo. El volumen válido está entre `0` y `1`.

## 5. Temporizadores

```ts
export type TimerConfig =
  | { readonly kind: 'open' }
  | { readonly kind: 'countdown'; readonly durationSeconds: number }
  | {
      readonly kind: 'intervals';
      readonly focusSeconds: number;
      readonly restSeconds: number;
      readonly cycles: number;
    };
```

Reglas:

- Cuenta regresiva: 1 a 240 minutos enteros.
- Intervalos: foco y descanso de 1 a 120 minutos; 1 a 12 ciclos.
- El reloj usa la fuente monotónica de I1 y deriva menos de un segundo por hora.
- Pausar detiene la fase y el audio; reanudar conserva el tiempo restante.
- Al completar una fase de foco, la siguiente fase se muestra explícitamente; el audio no cambia sin una regla declarada.
- Terminar un temporizador no inicia otra experiencia automáticamente.
- Una sesión libre se marca completada al detenerse después de 60 segundos; una cuenta regresiva al llegar a cero; intervalos al terminar todos sus ciclos. `listenedSeconds >= 60` gobierna progreso y racha, independientemente del campo `completed`.

## 6. Modelo local y persistencia

```ts
export interface UserPreferences {
  readonly schemaVersion: 1;
  readonly calibrationCompleted: boolean;
  readonly preferredMode: BrainSoundMode;
  readonly preferredIntensity: Intensity;
  readonly defaultTimer: TimerConfig;
  readonly initialVolume: number;
  readonly reducedMotion: boolean;
}

export interface SessionRecord {
  readonly id: string;
  readonly experienceId: string;
  readonly mode: BrainSoundMode;
  readonly intensity: Intensity;
  readonly timer: TimerConfig;
  readonly startedAt: string;
  readonly endedAt: string;
  readonly listenedSeconds: number;
  readonly completed: boolean;
  readonly rating: 1 | 2 | 3 | 4 | 5 | null;
}
```

Stores IndexedDB versión 1:

- `preferences`: una fila con clave `local`.
- `favorites`: claves `experienceId`, valor fecha de alta.
- `recent`: máximo 20 experiencias, ordenadas por último inicio.
- `sessions`: historial por `id`, índices por fecha, modo y experiencia.
- `meta`: versión del esquema y datos de migración.

Los IDs de sesión usan `crypto.randomUUID()`. Fechas persistidas usan ISO 8601 UTC; rachas se calculan al leer según zona local actual.

## 7. Recomendación explicable

El selector evalúa las experiencias disponibles del modo solicitado o preferido. Puntaje:

- `+4` si la experiencia es favorita;
- `+3` si coincide con el modo preferido;
- `+2` si su valoración promedio local es al menos 4;
- `+1` si no aparece entre las últimas tres experiencias iniciadas;
- `-3` si recibió valoración 1 o 2 en su última sesión.

Empates se resuelven por menor cantidad de reproducciones y después por ID ascendente. La explicación enumera únicamente factores que realmente aportaron o restaron puntos. Si no hay historial, se elige el primer ID del modo preferido y se explica “Basado en tu modo preferido”.

## 8. Arquitectura y estructura

```text
src/features/onboarding/**          Calibración
src/features/explore/**             Catálogo y búsqueda
src/features/favorites/**           Favoritos
src/features/progress/**            Resúmenes y rachas
src/features/settings/**            Preferencias
src/features/player/**              Integración de temporizadores/valoración
src/recommendation/**               Puntaje y explicaciones puras
src/storage/database.ts             Apertura y migración
src/storage/repositories/**         Puertos por agregado
src/storage/migrations/**           Cambios versionados
src/shared/time/**                  Día local y duración
tests/storage/**                    Integración IndexedDB
e2e/personal-journey.spec.ts        Primer uso y uso habitual
```

La UI depende de repositorios, no de `idb` ni de IndexedDB directamente. La lógica de recomendación recibe snapshots inmutables.

## 9. Base técnica y comandos

- Stack de I1–I3.
- `idb` 8.0.3 como único adaptador de IndexedDB aprobado por la SPEC maestra.

```powershell
npm ci
npm run test -- src/features src/recommendation src/storage tests/storage
npm run test:coverage
npm run test:e2e -- e2e/personal-journey.spec.ts
npm run verify
```

## 10. Estilo de código

- Repositorios con interfaces pequeñas por agregado.
- Migraciones puras respecto a su versión de entrada y verificadas con bases temporales.
- Funciones de puntuación sin reloj global; fecha y zona se inyectan.
- Estados vacíos y errores representados de forma accesible.

```ts
export interface Recommendation {
  readonly experienceId: string;
  readonly score: number;
  readonly reasons: readonly string[];
}

export interface RecommendationEngine {
  select(input: RecommendationInput): Recommendation;
}
```

## 11. Estrategia de pruebas

- **Unitarias:** validación de temporizadores, puntuación, desempate, razones y rachas.
- **Integración:** repositorios IndexedDB, migración nueva/doble apertura y consultas por índice.
- **E2E primer uso:** calibrar, iniciar recomendación, pausar, completar, valorar y ver progreso.
- **E2E habitual:** favorito, reciente, búsqueda, ajuste, recarga y persistencia.
- **Reloj:** una hora simulada con error menor de un segundo y pausas múltiples.
- **Cobertura:** 90% en temporizadores, recomendación, migraciones y repositorios; global 80%.

## 12. Manejo de errores

- IndexedDB no disponible: sesión reproducible en memoria, aviso de que no se guardará progreso.
- Migración fallida: no borrar la base; conservar versión anterior y mostrar recuperación.
- Experiencia histórica ausente del catálogo: conservar registro, mostrar título archivado si existe y excluirla de recomendaciones.
- Preferencia inválida: restaurar solo el campo afectado a su valor inicial explícito.
- Cuota insuficiente para historial: conservar preferencias/favoritos y rechazar el nuevo registro con aviso.

## 13. Criterios de aceptación

1. La calibración puede completarse, omitirse y repetirse.
2. Las cinco áreas de navegación funcionan con teclado.
3. Los tres temporizadores cumplen límites y pausa coherente.
4. Favoritos, últimos 20 recientes, preferencias e historial sobreviven recarga.
5. La recomendación produce el mismo resultado para la misma entrada y explica sus factores.
6. Una sesión de al menos 60 segundos actualiza minutos y rachas según día local.
7. Una sesión corta queda en historial sin incrementar racha.
8. El temporizador se desvía menos de un segundo por hora.
9. Una doble ejecución de migración converge sin pérdida.
10. No existen solicitudes externas ni datos personales fuera del dispositivo.
11. `npm run verify` finaliza correctamente.

## 14. Límites de trabajo

### Siempre hacer

- Validar datos al entrar y salir de almacenamiento.
- Mantener recomendación determinista y explicable.
- Probar migraciones desde base vacía y versión anterior.

### Preguntar primero

- Cambiar fórmula, límites temporales, criterio de racha o esquema persistente.
- Agregar dependencias, notificaciones o nuevas categorías de datos.
- Eliminar o truncar historial fuera del máximo de recientes definido.

### Nunca hacer

- Transmitir hábitos de escucha.
- Ocultar una recomendación como si fuera aleatoria o clínicamente optimizada.
- Borrar la base automáticamente ante una migración fallida.

## 15. Preguntas abiertas

No hay preguntas funcionales abiertas para planificar I4. Los valores iniciales y la fórmula quedan sujetos a aprobación explícita de esta SPEC.

## 16. Compuerta SDD

El plan I4 solo puede escribirse después de aprobación explícita de esta SPEC y no puede ejecutarse hasta completar I3.
