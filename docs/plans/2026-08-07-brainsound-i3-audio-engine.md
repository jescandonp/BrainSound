# BrainSound I3 Hybrid Audio Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reproducir las quince experiencias con capas locales, tres intensidades y transiciones continuas, conservando el sintetizador I1 como degradación segura.

**Architecture:** Un `SessionPlanner` puro transforma el manifiesto validado en `SessionPlan`; `AssetBufferRepository` decodifica assets locales; `HybridAudioEngine` posee el grafo y `TransitionCoordinator` programa rampas. Un puerto de recuperación decide conservar el grafo anterior o usar fallback sin exponer nodos Web Audio al controlador ni a React.

**Tech Stack:** TypeScript estricto, Web Audio API nativa, React, Vitest, Playwright, catálogo I2 validado y fixtures WAV sintéticos propios.

---

**SPEC:** `docs/specs/2026-08-07-brainsound-i3-audio-engine-spec.md` — aprobada el 2026-08-07.

**Estado:** propuesta para aprobación; no ejecutable sin aprobación explícita.

**Dependencia de ejecución:** I2 completada. Rama/worktree `feat/i3-audio-engine` desde el cierre I2.

## Mapa de archivos

```text
src/audio/planning/session-plan.ts               Contratos y validación
src/audio/planning/session-planner.ts            Manifiesto → plan
src/audio/assets/asset-buffer-repository.ts       Fetch/decodificación/cache memoria
src/audio/engine/audio-context-port.ts            Superficie Web Audio inyectable
src/audio/engine/hybrid-audio-engine.ts           Grafo y ownership
src/audio/transitions/transition-coordinator.ts   Rampas 8–12 s
src/audio/hybrid-audio-port.ts                    Recuperación y fallback
src/audio/diagnostics/audio-diagnostics.ts        Medidas efímeras
src/features/player/session-controller.ts         Estados ampliados I3
src/features/player/IntensitySelector.tsx          Intensidad accesible
tests/audio/fixtures/**                           WAV propios
e2e/audio-session.spec.ts                         Intensidad/transición/fallback
reports/audio/**                                  Evidencia especializada
```

### Task 1: Definir y validar `SessionPlan` con TDD

**Files:**
- Create: `src/audio/planning/session-plan.test.ts`
- Create: `src/audio/planning/session-plan.ts`

- [ ] **Step 1: Escribir RED de límites**

```ts
// src/audio/planning/session-plan.test.ts
import { describe, expect, it } from 'vitest';
import { validateSessionPlan } from './session-plan';

describe('validateSessionPlan', () => {
  it('acepta límites aprobados', () => {
    expect(validateSessionPlan({ experienceId: 'deep-focus-01', intensity: 'medium', masterGain: 0.6, filterHz: 1400, crossfadeSeconds: 10, layers: [{ assetId: 'music-1', gain: 0.6, playbackRate: 1, startOffsetSeconds: 0 }] })).toEqual([]);
  });
  it('acumula capas duplicadas y parámetros fuera de rango', () => {
    const issues = validateSessionPlan({ experienceId: 'deep-focus-01', intensity: 'deep', masterGain: 1, filterHz: 0, crossfadeSeconds: 4, layers: [{ assetId: 'x', gain: 2, playbackRate: 2, startOffsetSeconds: -1 }, { assetId: 'x', gain: 0.5, playbackRate: 1, startOffsetSeconds: 0 }] });
    expect(issues).toEqual(expect.arrayContaining(['masterGain', 'crossfadeSeconds', 'layers.assetId.duplicate', 'layers.0.gain', 'layers.0.playbackRate', 'layers.0.startOffsetSeconds']));
  });
});
```

- [ ] **Step 2: Run RED**

Run: `npm run test -- src/audio/planning/session-plan.test.ts`

Expected: FAIL por módulo ausente.

- [ ] **Step 3: Implementar contrato y validación pura**

```ts
// src/audio/planning/session-plan.ts
import type { ExperienceId, Intensity } from '../../catalog/catalog-types';
export interface LayerMix { readonly assetId: string; readonly gain: number; readonly playbackRate: number; readonly startOffsetSeconds: number; }
export interface SessionPlan { readonly experienceId: ExperienceId; readonly intensity: Intensity; readonly layers: readonly LayerMix[]; readonly masterGain: number; readonly filterHz: number; readonly crossfadeSeconds: number; }
export function validateSessionPlan(plan: SessionPlan): readonly string[] {
  const issues: string[] = []; const ids = new Set<string>();
  if (plan.masterGain < 0 || plan.masterGain > 0.9) issues.push('masterGain');
  if (plan.filterHz <= 0) issues.push('filterHz');
  if (plan.crossfadeSeconds < 8 || plan.crossfadeSeconds > 12) issues.push('crossfadeSeconds');
  if (plan.layers.length === 0) issues.push('layers');
  plan.layers.forEach((layer, index) => {
    if (ids.has(layer.assetId)) issues.push('layers.assetId.duplicate'); else ids.add(layer.assetId);
    if (layer.gain < 0 || layer.gain > 1) issues.push(`layers.${index}.gain`);
    if (layer.playbackRate < 0.95 || layer.playbackRate > 1.05) issues.push(`layers.${index}.playbackRate`);
    if (layer.startOffsetSeconds < 0) issues.push(`layers.${index}.startOffsetSeconds`);
  });
  return issues;
}
```

- [ ] **Step 4: GREEN y commit**

```powershell
npm run test -- src/audio/planning/session-plan.test.ts
git add src/audio/planning
git commit -m "feat: define validated hybrid session plans"
```

### Task 2: Transformar perfiles de catálogo en planes deterministas

**Files:**
- Create: `src/audio/planning/session-planner.test.ts`
- Create: `src/audio/planning/session-planner.ts`

- [ ] **Step 1: Escribir RED de determinismo y referencias**

Create a typed fixture from `CatalogExperience` and assert:

```ts
expect(planner.create(experience, 'medium')).toEqual(planner.create(experience, 'medium'));
expect(() => planner.create(experienceWithoutResolvedAsset, 'deep')).toThrow('Perfil deep inválido');
```

- [ ] **Step 2: Ejecutar RED**

Run: `npm run test -- src/audio/planning/session-planner.test.ts`

Expected: FAIL por clase ausente.

- [ ] **Step 3: Implementar planner sin efectos**

```ts
// src/audio/planning/session-planner.ts
import type { CatalogExperience, Intensity } from '../../catalog/catalog-types';
import { validateSessionPlan, type SessionPlan } from './session-plan';
export class SessionPlanner {
  create(experience: CatalogExperience, intensity: Intensity): SessionPlan {
    const profile = experience.intensityProfiles.find((entry) => entry.intensity === intensity);
    if (profile === undefined) throw new Error(`Perfil ${intensity} inválido para ${experience.id}`);
    const assetIds = new Set(experience.assets.map(({ id }) => id));
    if (profile.layers.some(({ assetId }) => !assetIds.has(assetId))) throw new Error(`Perfil ${intensity} inválido para ${experience.id}`);
    const plan: SessionPlan = { experienceId: experience.id, intensity, masterGain: profile.masterGain, filterHz: profile.filterHz, crossfadeSeconds: profile.crossfadeSeconds, layers: profile.layers.map((layer) => ({ ...layer, startOffsetSeconds: 0 })) };
    const issues = validateSessionPlan(plan);
    if (issues.length > 0) throw new Error(`Perfil ${intensity} inválido: ${issues.join(', ')}`);
    return plan;
  }
}
```

- [ ] **Step 4: GREEN y commit**

```powershell
npm run test -- src/audio/planning/session-planner.test.ts
npm run typecheck
git add src/audio/planning
git commit -m "feat: plan deterministic catalog sessions"
```

### Task 3: Decodificar y cachear buffers locales con ownership explícito

**Files:**
- Create: `src/audio/assets/asset-buffer-repository.test.ts`
- Create: `src/audio/assets/asset-buffer-repository.ts`

- [ ] **Step 1: Escribir RED de deduplicación y liberación**

```ts
import { describe, expect, it, vi } from 'vitest';
import { AssetBufferRepository } from './asset-buffer-repository';
it('deduplica cargas simultáneas y libera precarga', async () => {
  const fetchBytes = vi.fn().mockResolvedValue(new ArrayBuffer(4));
  const decode = vi.fn().mockResolvedValue({ duration: 3 } as AudioBuffer);
  const repository = new AssetBufferRepository(fetchBytes, decode);
  const [a, b] = await Promise.all([repository.get('asset', '/catalog/assets/a.wav'), repository.get('asset', '/catalog/assets/a.wav')]);
  expect(a).toBe(b); expect(fetchBytes).toHaveBeenCalledTimes(1);
  repository.retainOnly(new Set()); expect(repository.size()).toBe(0);
});
```

- [ ] **Step 2: RED**

Run: `npm run test -- src/audio/assets/asset-buffer-repository.test.ts`

Expected: FAIL.

- [ ] **Step 3: Implementar repositorio con promesas compartidas**

```ts
type FetchBytes = (path: string) => Promise<ArrayBuffer>;
type Decode = (bytes: ArrayBuffer) => Promise<AudioBuffer>;
export class AssetBufferRepository {
  private readonly buffers = new Map<string, Promise<AudioBuffer>>();
  constructor(private readonly fetchBytes: FetchBytes, private readonly decode: Decode) {}
  get(id: string, path: string): Promise<AudioBuffer> {
    const existing = this.buffers.get(id); if (existing !== undefined) return existing;
    const pending = this.fetchBytes(path).then(this.decode).catch((error) => { this.buffers.delete(id); throw error; });
    this.buffers.set(id, pending); return pending;
  }
  retainOnly(ids: ReadonlySet<string>): void { for (const id of this.buffers.keys()) if (!ids.has(id)) this.buffers.delete(id); }
  size(): number { return this.buffers.size; }
}
```

- [ ] **Step 4: GREEN y commit**

```powershell
npm run test -- src/audio/assets/asset-buffer-repository.test.ts
git add src/audio/assets
git commit -m "feat: cache decoded local audio buffers"
```

### Task 4: Crear el grafo híbrido y liberar todas sus fuentes

**Files:**
- Create: `src/audio/engine/audio-context-port.ts`
- Create: `src/audio/engine/hybrid-audio-engine.test.ts`
- Create: `src/audio/engine/hybrid-audio-engine.ts`

- [ ] **Step 1: Definir el puerto mínimo de contexto**

`audio-context-port.ts` exports `AudioContextPort` with `currentTime`, `destination`, `createBufferSource`, `createGain`, `createBiquadFilter`, `createDynamicsCompressor`, `resume`, `suspend`; it also exports `BrowserAudioContextPort` delegating to one native `AudioContext`.

- [ ] **Step 2: Escribir RED con nodos falsos**

Test that `prepare(plan, buffers)` creates one source/gain per layer, connects `source → layerGain → filter → master → limiter → destination`, starts only on `start()`, and `stop()` calls stop/disconnect once for every owned node.

Run: `npm run test -- src/audio/engine/hybrid-audio-engine.test.ts`

Expected: FAIL.

- [ ] **Step 3: Implementar unidad dueña del grafo**

Create `HybridAudioEngine` with methods:

```ts
prepare(plan: SessionPlan, buffers: ReadonlyMap<string, AudioBuffer>): void;
start(at?: number): Promise<void>;
pause(): Promise<void>;
resume(): Promise<void>;
stop(at?: number): Promise<void>;
setMasterGain(value: number, at: number): void;
```

Implementation requirements: validate all buffers before creating nodes; set limiter threshold `-1`; start sources at `context.currentTime` with plan offsets; use `linearRampToValueAtTime` from zero on start; stop/disconnect in `finally`; clear owned arrays after stop.

- [ ] **Step 4: Verificar ownership**

```powershell
npm run test -- src/audio/engine/hybrid-audio-engine.test.ts
npm run typecheck
```

Expected: graph/cleanup tests PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/audio/engine
git commit -m "feat: build owned hybrid Web Audio graph"
```

### Task 5: Programar fundidos y cancelación determinista

**Files:**
- Create: `src/audio/transitions/transition-coordinator.test.ts`
- Create: `src/audio/transitions/transition-coordinator.ts`

- [ ] **Step 1: RED para 8–12 segundos y cancelación**

Use fake engines and clock. Assert old gain ramps to `0`, new to target over exactly plan `crossfadeSeconds`; failure preparing new never changes old; a second transition cancels scheduled values of the first.

- [ ] **Step 2: Ejecutar RED**

Run: `npm run test -- src/audio/transitions/transition-coordinator.test.ts`

Expected: FAIL.

- [ ] **Step 3: Implementar coordinator**

```ts
export interface TransitionEngine { setMasterGain(value: number, at: number): void; start(at?: number): Promise<void>; stop(at?: number): Promise<void>; }
export interface TransitionClock { nowSeconds(): number; }
export class TransitionCoordinator {
  private generation = 0;
  constructor(private readonly clock: TransitionClock) {}
  async run(previous: TransitionEngine, next: TransitionEngine, targetGain: number, seconds: number): Promise<void> {
    if (seconds < 8 || seconds > 12) throw new Error('Fundido fuera de 8..12');
    const generation = ++this.generation; const start = this.clock.nowSeconds();
    next.setMasterGain(0, start); await next.start(start);
    if (generation !== this.generation) { await next.stop(); return; }
    previous.setMasterGain(0, start + seconds); next.setMasterGain(targetGain, start + seconds);
    await previous.stop(start + seconds);
  }
  cancel(): void { this.generation += 1; }
}
```

- [ ] **Step 4: GREEN y commit**

```powershell
npm run test -- src/audio/transitions/transition-coordinator.test.ts
git add src/audio/transitions
git commit -m "feat: coordinate cancellable audio crossfades"
```

### Task 6: Orquestar preparación, transición y fallback

**Files:**
- Create: `src/audio/hybrid-audio-port.test.ts`
- Create: `src/audio/hybrid-audio-port.ts`

- [ ] **Step 1: RED de las dos recuperaciones**

Test A: new asset decode fails while old engine plays → old remains. Test B: first experience fails → `fallback.start(getFallbackExperience(mode))` called. Assert typed failure phase.

- [ ] **Step 2: Implementar puerto de alto nivel**

```ts
export type AudioFailurePhase = 'manifest' | 'asset' | 'decode' | 'graph' | 'transition';
export interface AudioFailure { readonly phase: AudioFailurePhase; readonly experienceId: string; readonly recoverable: boolean; readonly message: string; }
```

`HybridSessionAudioPort` receives planner, repository, engine factory, transition coordinator and I1 `AudioPort`. It exposes `start(experience,intensity)`, `transition(experience,intensity)`, `pause`, `resume`, `stop`; it prepares completely before swapping `activeEngine`, retains old on recoverable transition failure, and uses I1 fallback only with no healthy active engine.

- [ ] **Step 3: Verificar**

```powershell
npm run test -- src/audio/hybrid-audio-port.test.ts
npm run typecheck
git add src/audio/hybrid-audio-port.ts src/audio/hybrid-audio-port.test.ts
git commit -m "feat: recover hybrid sessions through safe fallback"
```

Expected: all recovery tests PASS.

### Task 7: Ampliar controlador y UI para intensidad/transición

**Files:**
- Modify: `src/features/player/session-controller.ts`
- Modify: `src/features/player/session-controller.test.ts`
- Create: `src/features/player/IntensitySelector.tsx`
- Modify: `src/features/player/PlayerPanel.tsx`
- Modify: `src/app/App.tsx`

- [ ] **Step 1: RED de estados I3**

Extend controller tests to assert:

```ts
expect(states).toEqual(['preparing', 'playing', 'transitioning', 'playing']);
expect(controller.getSnapshot().source).toBe('hybrid');
```

and fallback failure produces `{ status: 'playing', source: 'fallback' }`.

- [ ] **Step 2: Implementar estado discriminado**

```ts
export type SessionStatus = 'idle' | 'preparing' | 'playing' | 'paused' | 'transitioning';
export type SessionSource = 'hybrid' | 'fallback';
export interface SessionState { readonly status: SessionStatus; readonly experience: CatalogExperience | null; readonly intensity: Intensity; readonly source: SessionSource | null; readonly error: AudioFailure | null; }
```

Update controller operations so state changes before/after awaited ports and failures never claim hybrid source.

- [ ] **Step 3: Crear selector accesible**

```tsx
export function IntensitySelector({ value, disabled, onChange }: { readonly value: Intensity; readonly disabled: boolean; readonly onChange: (value: Intensity) => void }) {
  return <fieldset disabled={disabled}><legend>Intensidad</legend>{(['soft','medium','deep'] as const).map((item) => <label key={item}><input type="radio" name="intensity" checked={value === item} onChange={() => onChange(item)} />{{ soft: 'Suave', medium: 'Media', deep: 'Profunda' }[item]}</label>)}</fieldset>;
}
```

- [ ] **Step 4: Integrar catálogo real**

Replace I2 `getFallbackExperience(experience.mode)` on catalog play with controller `start(experience, selectedIntensity)`. Keep home mode cards mapped to a deterministic first experience by ID. Player displays `Audio híbrido` or `Modo de respaldo` as text.

- [ ] **Step 5: Verificar y commit**

```powershell
npm run test -- src/features/player/session-controller.test.ts
npm run typecheck
git add src/features/player src/app/App.tsx
git commit -m "feat: control hybrid intensity and transitions"
```

### Task 8: Medir inicio, picos y continuidad sin telemetría

**Files:**
- Create: `src/audio/diagnostics/audio-diagnostics.test.ts`
- Create: `src/audio/diagnostics/audio-diagnostics.ts`
- Create: `scripts/report-audio-diagnostics.ts`
- Create: `reports/audio/audio-diagnostics.md`

- [ ] **Step 1: RED de métricas efímeras**

Test recorder accepts `prepare-start`, `audible`, `peak`, `transition-start/end`, returns immutable snapshot, and `clear()` empties it.

- [ ] **Step 2: Implementar recorder en memoria**

```ts
export interface AudioDiagnostic { readonly kind: 'start-ms' | 'peak' | 'transition-ms'; readonly experienceId: string; readonly value: number; }
export class AudioDiagnostics { private entries: AudioDiagnostic[] = []; record(entry: AudioDiagnostic): void { this.entries.push(entry); } snapshot(): readonly AudioDiagnostic[] { return this.entries.map((entry) => ({ ...entry })); } clear(): void { this.entries = []; } }
```

Wire it only through injected diagnostics; do not persist or send it. Report script consumes deterministic fixture results and writes max start, max peak and transition duration.

- [ ] **Step 3: Verificar presupuesto técnico**

Run tests with fake clock for `1499 ms` PASS and `1501 ms` FAIL; peak must remain `<= 1`; transitions each 8–12 seconds.

- [ ] **Step 4: Commit**

```powershell
git add src/audio/diagnostics scripts/report-audio-diagnostics.ts reports/audio
git commit -m "test: measure local audio performance gates"
```

### Task 9: Recorridos E2E y auditoría perceptual

**Files:**
- Create: `e2e/audio-session.spec.ts`
- Create: `docs/qa/audio-audit.md`

- [ ] **Step 1: E2E observable**

Create tests that start `deep-focus-01`, assert source hybrid, choose Profunda, wait for transition state to return playing, change to `deep-focus-02`, pause/resume/stop, and route one asset to 404 to assert old session or fallback remains active.

- [ ] **Step 2: Ejecutar navegadores**

```powershell
npx playwright test e2e/audio-session.spec.ts
```

Expected: Chromium/WebKit PASS; no assertion depends on audible perception.

- [ ] **Step 3: Auditar audio real sin inventar resultado**

Create `docs/qa/audio-audit.md` with rows for all 15 experiences × 3 intensities and columns: browser, device, start ms, clipping, silence, loop seam, transition, reviewer, date, result. Fill only observed results. Any `FAIL` blocks I3.

- [ ] **Step 4: Commit**

```powershell
git add e2e/audio-session.spec.ts docs/qa/audio-audit.md
git commit -m "test: audit hybrid audio sessions"
```

### Task 10: Verificación y cierre I3

**Files:**
- Modify: `README.md`
- Modify: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Modify: `docs/plans/2026-08-07-brainsound-i3-audio-engine.md`

- [ ] **Step 1: Ejecutar compuertas**

```powershell
npm run validate:catalog
npm run test:coverage
npm run verify
npm run test:e2e
git diff --check
```

Expected: planificación/transición/recuperación ≥90%, global ≥80%, 15×3 planes válidos, E2E ambos navegadores y diff limpio.

- [ ] **Step 2: Revisar evidencia especializada**

Confirm every audit row has observed result; max start ≤1500 ms; no clipping or silence unexpected; all crossfades 8–12 s. Do not convert an unmeasured row to PASS.

- [ ] **Step 3: Documentar y cerrar**

Update README with hybrid/fallback behavior. Append observed commit, test counts, max start/peak and audit result to `## Resultados de ejecución`. Set roadmap I3 `Completada` only when every gate passes.

- [ ] **Step 4: Commit**

```powershell
git add README.md docs/plans reports/audio docs/qa/audio-audit.md
git commit -m "docs: close BrainSound hybrid audio iteration"
```

## Trazabilidad SPEC → tareas

| Criterio I3 | Tareas |
|---|---|
| 15×3 planes válidos | 1, 2 |
| Buffers y ownership | 3, 4 |
| Fundidos 8–12 s | 5 |
| Recuperación/fallback | 6, 7 |
| Intensidad y cambio | 7 |
| Inicio ≤1,5 s y sin clipping | 8, 9 |
| Compatibilidad y evidencia | 9, 10 |

## Gate de aprobación antes de ejecutar

Este plan no autoriza audio no aprobado, telemetría ni afirmaciones clínicas. Requiere aprobación explícita y I2 completada antes de crear el worktree.
