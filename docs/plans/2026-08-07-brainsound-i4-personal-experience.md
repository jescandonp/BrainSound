# BrainSound I4 Personal Experience Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Completar la experiencia personal local con calibración, navegación, tres temporizadores, favoritos, recientes, historial, valoración, recomendación explicable, progreso y rachas persistentes.

**Architecture:** Repositorios pequeños aíslan IndexedDB mediante `idb`; temporizadores, recomendación y rachas permanecen puros con reloj/zona inyectables. React orquesta casos de uso y estados degradados, pero no accede directamente a la base.

**Tech Stack:** React 19.2.8, TypeScript 7.0.2, `idb` 8.0.3, IndexedDB, Vitest, fake-indexeddb para integración, Playwright Chromium/WebKit.

---

**SPEC:** `docs/specs/2026-08-07-brainsound-i4-personal-experience-spec.md` — aprobada el 2026-08-07.

**Estado:** propuesta para aprobación; no ejecutable sin aprobación explícita.

**Dependencia de ejecución:** I3 completada. Rama/worktree `feat/i4-personal-experience`.

## Mapa de archivos

```text
src/storage/database.ts                       Esquema/open/migración v1
src/storage/repositories/preferences.ts       Preferencias local
src/storage/repositories/favorites.ts         Favoritos
src/storage/repositories/recent.ts            Últimos 20
src/storage/repositories/sessions.ts          Historial
src/features/timer/timer-config.ts            Validación
src/features/timer/session-timer.ts           Fases monotónicas
src/recommendation/recommendation-engine.ts    Puntaje/explicación
src/features/progress/progress-summary.ts      Minutos/rachas
src/features/onboarding/**                    Calibración
src/features/navigation/**                    Cinco áreas
src/features/favorites/**                     Lista/acciones
src/features/progress/**                      Resumen
src/features/settings/**                      Ajustes
tests/storage/**                              IndexedDB real simulado
e2e/personal-journey.spec.ts                  Primer uso/habitual
e2e/storage-degraded.spec.ts                  Modo memoria
```

### Task 1: Instalar únicamente las dependencias aprobadas para persistencia

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`
- Modify: `vitest.config.ts`
- Create: `tests/setup-indexeddb.ts`

- [ ] **Step 1: Agregar versiones exactas**

Run:

```powershell
npm install idb@8.0.3
npm install --save-dev fake-indexeddb@6.2.5
```

Expected: lockfile contiene exactamente ambas versiones; no otra dependencia de almacenamiento.

- [ ] **Step 2: Configurar entorno de integración**

Create `tests/setup-indexeddb.ts`:

```ts
import 'fake-indexeddb/auto';
```

Add to Vitest:

```ts
setupFiles: ['tests/setup-indexeddb.ts'],
```

- [ ] **Step 3: Verificar y commit**

```powershell
npm ls idb fake-indexeddb
npm run typecheck
git add package.json package-lock.json vitest.config.ts tests/setup-indexeddb.ts
git commit -m "build: add approved local persistence dependencies"
```

### Task 2: Crear esquema IndexedDB v1 y probar doble apertura

**Files:**
- Create: `src/storage/database.test.ts`
- Create: `src/storage/database.ts`

- [ ] **Step 1: Escribir RED de stores/índices**

```ts
import { afterEach, describe, expect, it } from 'vitest';
import { deleteDB } from 'idb';
import { openBrainSoundDatabase } from './database';
afterEach(() => deleteDB('brainsound'));
it('crea v1 y converge en doble apertura', async () => {
  const first = await openBrainSoundDatabase(); first.close();
  const second = await openBrainSoundDatabase();
  expect([...second.objectStoreNames]).toEqual(['favorites','meta','preferences','recent','sessions']);
  expect(second.transaction('sessions').store.indexNames).toEqual(['by-date','by-experience','by-mode']);
  second.close();
});
```

- [ ] **Step 2: RED**

Run: `npm run test -- src/storage/database.test.ts`

Expected: FAIL.

- [ ] **Step 3: Implementar esquema tipado**

Create `database.ts` with `BrainSoundDb extends DBSchema`; stores/keys exactly from SPEC. `openDB('brainsound', 1, { upgrade })` creates missing stores conditionally and indices `startedAt`, `experienceId`, `mode`. Export DB value types `UserPreferences`, `FavoriteRecord`, `RecentRecord`, `SessionRecord`, `MetaRecord` from `storage-types.ts` if the file exceeds 150 lines.

- [ ] **Step 4: GREEN, doble ejecución y commit**

```powershell
npm run test -- src/storage/database.test.ts
npm run typecheck
git add src/storage
git commit -m "feat: create versioned BrainSound database"
```

### Task 3: Persistir preferencias con reparación campo a campo

**Files:**
- Create: `src/storage/repositories/preferences.test.ts`
- Create: `src/storage/repositories/preferences.ts`

- [ ] **Step 1: RED de defaults y reparación**

Assert empty DB returns Deep Focus, medium, open, volume `.7`, system reduced-motion; invalid volume `2` repairs only volume while preserving preferred mode.

- [ ] **Step 2: Implementar defaults explícitos**

```ts
export const defaultPreferences = (reducedMotion: boolean): UserPreferences => ({ schemaVersion: 1, calibrationCompleted: false, preferredMode: 'deep-focus', preferredIntensity: 'medium', defaultTimer: { kind: 'open' }, initialVolume: 0.7, reducedMotion });
```

`PreferencesRepository.get()` reads key `local`, repairs each field with type guards, and writes repaired value. `save()` rejects volume outside 0..1 and invalid timer.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/storage/repositories/preferences.test.ts
git add src/storage/repositories/preferences.ts src/storage/repositories/preferences.test.ts
git commit -m "feat: persist and repair local preferences"
```

### Task 4: Implementar favoritos, recientes e historial

**Files:**
- Create: `src/storage/repositories/favorites.test.ts`
- Create: `src/storage/repositories/favorites.ts`
- Create: `src/storage/repositories/recent.test.ts`
- Create: `src/storage/repositories/recent.ts`
- Create: `src/storage/repositories/sessions.test.ts`
- Create: `src/storage/repositories/sessions.ts`

- [ ] **Step 1: RED por agregado**

Test favorites toggle idempotently; recent moves repeated ID to first and trims to 20; sessions create via `crypto.randomUUID`, query newest-first, rate only 1..5/null and preserve a catalog-missing record.

- [ ] **Step 2: Implementar repositorios pequeños**

Public methods must be exactly:

```ts
FavoritesRepository.list(): Promise<readonly FavoriteRecord[]>;
FavoritesRepository.set(experienceId: ExperienceId, favorite: boolean, now: string): Promise<void>;
RecentRepository.record(experienceId: ExperienceId, startedAt: string): Promise<void>;
RecentRepository.list(): Promise<readonly RecentRecord[]>;
SessionsRepository.add(record: SessionRecord): Promise<void>;
SessionsRepository.rate(id: string, rating: 1|2|3|4|5|null): Promise<void>;
SessionsRepository.list(): Promise<readonly SessionRecord[]>;
```

Use one readwrite transaction for trim-after-insert in Recent.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/storage/repositories
npm run typecheck
git add src/storage/repositories
git commit -m "feat: persist favorites recent and sessions"
```

### Task 5: Validar los tres temporizadores y sus límites

**Files:**
- Create: `src/features/timer/timer-config.test.ts`
- Create: `src/features/timer/timer-config.ts`

- [ ] **Step 1: RED de límites**

```ts
expect(validateTimer({ kind: 'open' })).toEqual([]);
expect(validateTimer({ kind: 'countdown', durationSeconds: 60 })).toEqual([]);
expect(validateTimer({ kind: 'countdown', durationSeconds: 0 })).toContain('durationSeconds');
expect(validateTimer({ kind: 'intervals', focusSeconds: 60, restSeconds: 60, cycles: 12 })).toEqual([]);
expect(validateTimer({ kind: 'intervals', focusSeconds: 60, restSeconds: 60, cycles: 13 })).toContain('cycles');
```

- [ ] **Step 2: Implementar unión discriminada y validación**

```ts
export type TimerConfig = { readonly kind:'open' } | { readonly kind:'countdown'; readonly durationSeconds:number } | { readonly kind:'intervals'; readonly focusSeconds:number; readonly restSeconds:number; readonly cycles:number };
const validSeconds = (value:number,maxMinutes:number) => Number.isInteger(value) && value >= 60 && value <= maxMinutes * 60;
export function validateTimer(config: TimerConfig): readonly string[] {
  if (config.kind === 'open') return [];
  if (config.kind === 'countdown') return validSeconds(config.durationSeconds, 240) ? [] : ['durationSeconds'];
  const issues:string[] = [];
  if (!validSeconds(config.focusSeconds, 120)) issues.push('focusSeconds');
  if (!validSeconds(config.restSeconds, 120)) issues.push('restSeconds');
  if (!Number.isInteger(config.cycles) || config.cycles < 1 || config.cycles > 12) issues.push('cycles');
  return issues;
}
```

- [ ] **Step 3: GREEN y commit**

```powershell
npm run test -- src/features/timer/timer-config.test.ts
git add src/features/timer/timer-config.ts src/features/timer/timer-config.test.ts
git commit -m "feat: validate personal timer configurations"
```

### Task 6: Coordinar fases con reloj monotónico

**Files:**
- Create: `src/features/timer/session-timer.test.ts`
- Create: `src/features/timer/session-timer.ts`

- [ ] **Step 1: RED de una hora, pausa y fases**

With injected clock, assert 3,600,000 ms → 3,600 seconds; pause freezes; countdown completes at zero; two interval cycles emit focus/rest/focus/rest/completed and never start audio automatically.

- [ ] **Step 2: Implementar máquina pura**

```ts
export type TimerPhase = 'focus'|'rest'|'completed';
export interface TimerSnapshot { readonly elapsedSeconds:number; readonly remainingSeconds:number|null; readonly phase:TimerPhase; readonly cycle:number; readonly running:boolean; }
```

`SessionTimer.start/pause/resume/stop/snapshot` derive from injected `Clock.now()` and config, never decrement counters.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/features/timer/session-timer.test.ts
git add src/features/timer/session-timer.ts src/features/timer/session-timer.test.ts
git commit -m "feat: run monotonic countdown and intervals"
```

### Task 7: Calcular recomendación determinista y explicable

**Files:**
- Create: `src/recommendation/recommendation-engine.test.ts`
- Create: `src/recommendation/recommendation-engine.ts`

- [ ] **Step 1: RED de cada factor y desempate**

Build three candidate fixtures; assert +4 favorite, +3 preferred mode, +2 average ≥4, +1 not in last three, -3 last rating ≤2; tie by fewer plays then ID; empty history chooses first preferred-mode ID and exact reason `Basado en tu modo preferido`.

- [ ] **Step 2: Implementar tipos y puntaje**

```ts
export interface Recommendation { readonly experienceId: ExperienceId; readonly score:number; readonly reasons:readonly string[]; }
export interface RecommendationCandidate { readonly experienceId:ExperienceId; readonly mode:BrainSoundMode; readonly favorite:boolean; readonly averageRating:number|null; readonly lastRating:number|null; readonly plays:number; }
export interface RecommendationInput { readonly candidates:readonly RecommendationCandidate[]; readonly preferredMode:BrainSoundMode; readonly recentIds:readonly ExperienceId[]; }
export class RecommendationEngine {
  select(input: RecommendationInput): Recommendation {
    if (input.candidates.length === 0) throw new Error('No hay experiencias para recomendar');
    if (input.recentIds.length === 0 && input.candidates.every(({ plays }) => plays === 0)) {
      const first = input.candidates.filter(({ mode }) => mode === input.preferredMode).sort((a,b) => a.experienceId.localeCompare(b.experienceId))[0];
      if (first === undefined) throw new Error('No hay experiencias del modo preferido');
      return { experienceId:first.experienceId, score:3, reasons:['Basado en tu modo preferido'] };
    }
    const ranked = input.candidates.map((candidate) => {
      let score = 0; const reasons:string[] = [];
      if (candidate.favorite) { score += 4; reasons.push('Está entre tus favoritos'); }
      if (candidate.mode === input.preferredMode) { score += 3; reasons.push('Coincide con tu modo preferido'); }
      if (candidate.averageRating !== null && candidate.averageRating >= 4) { score += 2; reasons.push('La has valorado positivamente'); }
      if (!input.recentIds.slice(0, 3).includes(candidate.experienceId)) { score += 1; reasons.push('Aporta variedad frente a tus sesiones recientes'); }
      if (candidate.lastRating !== null && candidate.lastRating <= 2) { score -= 3; reasons.push('Tu última valoración fue baja'); }
      return { experienceId:candidate.experienceId, score, reasons, plays:candidate.plays };
    }).sort((a,b) => b.score-a.score || a.plays-b.plays || a.experienceId.localeCompare(b.experienceId));
    const selected = ranked[0];
    if (selected === undefined) throw new Error('No hay experiencias para recomendar');
    return { experienceId:selected.experienceId, score:selected.score, reasons:selected.reasons.length === 0 ? ['Basado en tu modo preferido'] : selected.reasons };
  }
}
```

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/recommendation/recommendation-engine.test.ts
git add src/recommendation
git commit -m "feat: recommend local experiences explainably"
```

### Task 8: Calcular progreso y rachas por día local

**Files:**
- Create: `src/features/progress/progress-summary.test.ts`
- Create: `src/features/progress/progress-summary.ts`

- [ ] **Step 1: RED de elegibilidad y zona**

Assert 59 seconds stays history but contributes zero; 60 seconds contributes one minute/day; consecutive local dates count streak; gap resets current but preserves maximum; mode totals sum floor(listenedSeconds/60).

- [ ] **Step 2: Implementar con zona inyectada**

```ts
export interface ProgressSummary { readonly totalMinutes:number; readonly minutesByMode:Readonly<Record<BrainSoundMode,number>>; readonly eligibleSessions:number; readonly currentStreak:number; readonly longestStreak:number; }
export function summarizeProgress(sessions: readonly SessionRecord[], todayLocalDate: string, toLocalDate:(iso:string)=>string): ProgressSummary;
```

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/features/progress/progress-summary.test.ts
git add src/features/progress
git commit -m "feat: calculate private progress and streaks"
```

### Task 9: Construir calibración y navegación de cinco áreas

**Files:**
- Create: `src/features/onboarding/OnboardingScreen.tsx`
- Create: `src/features/navigation/AppNavigation.tsx`
- Modify: `src/app/App.tsx`
- Create: `e2e/personal-journey.spec.ts`

- [ ] **Step 1: E2E RED de omitir/completar**

Test fresh context shows calibration; `Omitir` persists explicit defaults; second fresh DB path completes Relaxation/deep/45 min; reload does not repeat; Settings can reopen calibration. Assert navigation labels Inicio, Explorar, Favoritos, Progreso, Ajustes.

- [ ] **Step 2: Implementar onboarding controlado**

`OnboardingScreen` takes `initial`, `onSave`, `onSkip`; use radio groups and duration select `[open,25,45,60,90]`; visible privacy note. `onSkip` saves `defaultPreferences(matchMedia(...).matches)`.

- [ ] **Step 3: Implementar navegación sin router adicional**

`AppNavigation` owns no data; it receives current view and `onNavigate`. App renders the five screens and repositories from one composition root.

- [ ] **Step 4: Verificar y commit**

```powershell
npx playwright test e2e/personal-journey.spec.ts --project=chromium
npm run typecheck
git add src/features/onboarding src/features/navigation src/app/App.tsx e2e/personal-journey.spec.ts
git commit -m "feat: calibrate and navigate personal BrainSound"
```

### Task 10: Integrar favoritos, temporizador, valoración y progreso

**Files:**
- Create: `src/features/favorites/FavoritesScreen.tsx`
- Create: `src/features/progress/ProgressScreen.tsx`
- Create: `src/features/settings/SettingsScreen.tsx`
- Modify: `src/features/explore/ExperienceCard.tsx`
- Modify: `src/features/player/PlayerPanel.tsx`
- Modify: `src/app/App.tsx`

- [ ] **Step 1: Tests de componente/recorrido RED**

Extend personal journey: favorite from Explore appears in Favorites; start records recent; select countdown; pause; complete; choose rating 5; Progreso increments; Settings volume `.7` and preferred intensity persist reload.

- [ ] **Step 2: Implementar casos de uso**

On start: write recent, capture start ISO and timer snapshot. On stop/completion: calculate listened seconds from monotonic timer, create one `SessionRecord`, then allow rating update. A free session is `completed` only at ≥60 s; countdown at zero; intervals after all cycles. Do not double-write on React StrictMode.

- [ ] **Step 3: Construir pantallas y estados vacíos**

Favorites empty text `Aún no tienes favoritos`; Progress zero state; Settings validates before save. Every icon action has accessible name and text feedback.

- [ ] **Step 4: Verificar y commit**

```powershell
npm run test -- src/features src/storage src/recommendation
npx playwright test e2e/personal-journey.spec.ts
git add src/features src/app/App.tsx
git commit -m "feat: complete local personal experience"
```

### Task 11: Degradar a memoria cuando IndexedDB no está disponible

**Files:**
- Create: `src/storage/memory-repositories.ts`
- Create: `src/storage/create-storage.ts`
- Create: `e2e/storage-degraded.spec.ts`

- [ ] **Step 1: E2E RED**

Override `indexedDB.open` to throw; assert alert `Tus cambios no se guardarán en este dispositivo`; start/pause/stop still works; Progress can show current in-memory session; reload resets it.

- [ ] **Step 2: Implementar fallback explícito**

`createStorage()` tries opening once and returns `{ kind:'persistent', repositories }` or `{ kind:'memory', repositories, error }`. Memory repositories implement exactly the same interfaces and never pretend persistence.

- [ ] **Step 3: Verificar y commit**

```powershell
npx playwright test e2e/storage-degraded.spec.ts
npm run typecheck
git add src/storage/memory-repositories.ts src/storage/create-storage.ts e2e/storage-degraded.spec.ts
git commit -m "fix: preserve sessions when local storage is unavailable"
```

### Task 12: Verificar, documentar y cerrar I4

**Files:**
- Modify: `README.md`
- Modify: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Modify: `docs/plans/2026-08-07-brainsound-i4-personal-experience.md`

- [ ] **Step 1: Ejecutar compuertas**

```powershell
npm run test:coverage
npm run verify
npm run test:e2e
git diff --check
```

Expected: temporizadores/recomendación/migraciones/repositorios ≥90%, global ≥80%, E2E Chromium/WebKit PASS.

- [ ] **Step 2: Verificar invariantes**

Run a simulated hour and assert deviation <1 second; open database twice; verify recent count ≤20; inspect browser network and confirm no external request.

- [ ] **Step 3: Registrar resultados observados**

Document local privacy/data semantics and three timers in README. Append exact commit, tests, coverage, drift and migration results. Set I4 completed only if green.

- [ ] **Step 4: Commit**

```powershell
git add README.md docs/plans
git commit -m "docs: close BrainSound personal experience iteration"
```

## Trazabilidad SPEC → tareas

| Criterio I4 | Tareas |
|---|---|
| IndexedDB/migración | 1, 2 |
| Preferencias y calibración | 3, 9 |
| Favoritos/recientes/historial | 4, 10 |
| Tres temporizadores/deriva | 5, 6 |
| Recomendación explicable | 7 |
| Progreso y rachas | 8, 10 |
| Cinco áreas de navegación | 9, 10 |
| Degradación en memoria | 11 |
| Verificación integral | 12 |

## Gate de aprobación antes de ejecutar

Este plan no autoriza sincronización, cuentas, notificaciones ni cambios en fórmula, rachas o esquema. Requiere aprobación explícita e I3 completada.
