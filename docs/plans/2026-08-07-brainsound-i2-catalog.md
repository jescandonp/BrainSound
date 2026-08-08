# BrainSound I2 Catalog and Licensing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar un catálogo local validado de quince experiencias con trazabilidad de licencia, integridad, presupuesto y exploración/búsqueda, manteniendo el audio sintético de I1 como reproducción activa.

**Architecture:** Los manifiestos JSON entran como `unknown`, pasan por validación estructural y de archivos, y solo entonces forman un `CatalogRepository`. La aplicación consume contratos tipados y búsqueda pura; los scripts Node verifican disco, SHA-256, presupuesto y generan reportes deterministas sin descargar ni modificar activos.

**Tech Stack:** Node.js 24.18.1, TypeScript 7.0.2, React 19.2.8, Vite 8.1.5, Vitest 4.1.10, Playwright 1.61.1, APIs nativas `node:fs`, `node:crypto` y `fetch` local.

---

**SPEC:** `docs/specs/2026-08-07-brainsound-i2-catalog-spec.md` — aprobada el 2026-08-07.

**Estado:** aprobado por el usuario el 2026-08-07; ejecución condicionada al cierre satisfactorio de I1.

**Dependencia de ejecución:** I1 completada y verificada. Trabajar en una rama/worktree `feat/i2-catalog` creada desde el cierre de I1.

## Compuertas de alcance

- No reproducir capas reales en I2; la acción de escuchar continúa usando el fallback sintético por modo.
- No descargar activos desde el validador ni desde la aplicación.
- No aceptar una fila del registro por inferencia: fuente, autor, licencia y permiso deben ser visibles en la fuente primaria.
- No superar 314.572.800 bytes de activos únicos.
- Un cambio de esquema, formatos permitidos, presupuesto o IDs vuelve primero a la SPEC I2.

## Mapa de archivos

```text
docs/catalog/asset-register.md                 Evidencia humana de fuentes/licencias
public/catalog/index.json                     Índice versionado
public/catalog/experiences/*.json             Quince manifiestos
public/catalog/assets/**                      Activos aprobados
src/catalog/catalog-types.ts                  Tipos públicos
src/catalog/catalog-schema.ts                 Validación de unknown
src/catalog/catalog-search.ts                 Búsqueda y filtros
src/catalog/catalog-repository.ts             Carga validada
src/features/explore/ExploreScreen.tsx        Exploración accesible
src/features/explore/ExperienceCard.tsx       Tarjeta de catálogo
scripts/catalog-files.ts                      Hash/tamaño/rutas
scripts/validate-catalog.ts                   Compuerta CLI
scripts/report-catalog.ts                     Reporte determinista
reports/catalog/catalog-validation.md         Evidencia generada
tests/catalog/fixtures/**                     Casos válidos/inválidos
e2e/catalog.spec.ts                           Quince experiencias y búsqueda
e2e/catalog-fallback.spec.ts                  Catálogo inválido y fallback
```

### Task 1: Crear la compuerta verificable de investigación de activos

**Files:**
- Create: `docs/catalog/asset-register.md`
- Modify: `docs/plans/2026-08-07-brainsound-i2-catalog.md`

- [ ] **Step 1: Crear el registro con criterios explícitos**

Create `docs/catalog/asset-register.md`:

```md
# Registro de activos de BrainSound

## Regla de admisión

Una fila solo pasa a `APROBADO` cuando la página primaria muestra autor, licencia y permiso compatible con redistribución y modificación. La URL de descarga por sí sola no es evidencia. No se usa contenido de Brain.fm.

## Fuentes consultadas

Registrar por cada consulta: fecha, dominio primario, consulta usada y resultado. No registrar una fuente agregadora como evidencia final.

## Activos

| Asset ID | Kind | Source page | Direct file | Author | License ID | License page | Attribution | Verified on | Bytes | SHA-256 | Status |
|---|---|---|---|---|---|---|---|---|---:|---|---|

Estados válidos: `CANDIDATO`, `RECHAZADO`, `APROBADO`.

## Rechazos

Conservar asset ID candidato, fuente, fecha y motivo exacto. Un rechazo no se reutiliza sin evidencia nueva.

## Aprobación humana

La ejecución no continúa a manifiestos hasta que todas las filas usadas estén en `APROBADO` y el total proyectado sea menor o igual a 314.572.800 bytes.
```

- [ ] **Step 2: Investigar únicamente en fuentes primarias**

Para cada candidato, abrir la página del autor/repositorio oficial, comprobar visualmente licencia y permiso, descargar a una zona de revisión fuera de `public/catalog`, calcular evidencia y registrar el resultado. Use:

```powershell
$reviewedAssetPath = Read-Host 'Ruta absoluta observada del archivo descargado'
Get-FileHash -Algorithm SHA256 -LiteralPath $reviewedAssetPath
Get-Item -LiteralPath $reviewedAssetPath | Select-Object FullName,Length
```

Expected: cada archivo propuesto tiene hash de 64 hexadecimales y tamaño observado; la ruta proviene de la revisión humana, no del plan.

- [ ] **Step 3: Revisar el registro antes de tocar el catálogo**

Run:

```powershell
Select-String -LiteralPath docs/catalog/asset-register.md -Pattern 'APROBADO'
Select-String -LiteralPath docs/catalog/asset-register.md -Pattern 'Brain.fm'
```

Expected: existen filas aprobadas suficientes para cubrir `music`, `ambient` y `nature`; la única mención de Brain.fm es la prohibición. Si falta evidencia, detener I2 y solicitar decisión, sin crear manifiestos.

- [ ] **Step 4: Commit de evidencia**

```powershell
git add docs/catalog/asset-register.md
git commit -m "docs: verify catalog asset licensing evidence"
```

### Task 2: Definir contratos tipados y reglas de IDs

**Files:**
- Create: `src/catalog/catalog-types.test.ts`
- Create: `src/catalog/catalog-types.ts`

- [ ] **Step 1: Escribir pruebas fallidas de contratos auxiliares**

Create `src/catalog/catalog-types.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { experienceIds, isExperienceId } from './catalog-types';

describe('catalog types', () => {
  it('declara exactamente cinco IDs por modo', () => {
    expect(experienceIds).toHaveLength(15);
    expect(experienceIds.filter((id) => id.startsWith('deep-focus-'))).toHaveLength(5);
    expect(experienceIds.filter((id) => id.startsWith('creativity-'))).toHaveLength(5);
    expect(experienceIds.filter((id) => id.startsWith('relaxation-'))).toHaveLength(5);
  });

  it('rechaza IDs fuera del contrato', () => {
    expect(isExperienceId('deep-focus-01')).toBe(true);
    expect(isExperienceId('sleep-01')).toBe(false);
    expect(isExperienceId('deep-focus-06')).toBe(false);
  });
});
```

- [ ] **Step 2: Verificar fallo RED**

Run: `npm run test -- src/catalog/catalog-types.test.ts`

Expected: FAIL porque `catalog-types.ts` no existe.

- [ ] **Step 3: Implementar los contratos completos de la SPEC**

Create `src/catalog/catalog-types.ts`:

```ts
import type { BrainSoundMode } from '../shared/domain/mode';

export type Intensity = 'soft' | 'medium' | 'deep';
export type LayerKind = 'music' | 'ambient' | 'nature';

export const experienceIds = [
  'deep-focus-01', 'deep-focus-02', 'deep-focus-03', 'deep-focus-04', 'deep-focus-05',
  'creativity-01', 'creativity-02', 'creativity-03', 'creativity-04', 'creativity-05',
  'relaxation-01', 'relaxation-02', 'relaxation-03', 'relaxation-04', 'relaxation-05',
] as const;

export type ExperienceId = (typeof experienceIds)[number];

export interface CatalogAsset {
  readonly id: string;
  readonly path: string;
  readonly kind: LayerKind;
  readonly bytes: number;
  readonly sha256: string;
  readonly sourceUrl: string;
  readonly author: string;
  readonly licenseId: string;
  readonly licenseUrl: string;
  readonly attribution: string;
  readonly verifiedOn: string;
}

export interface IntensityProfile {
  readonly intensity: Intensity;
  readonly masterGain: number;
  readonly filterHz: number;
  readonly crossfadeSeconds: number;
  readonly layers: readonly {
    readonly assetId: string;
    readonly gain: number;
    readonly playbackRate: number;
  }[];
}

export interface CatalogExperience {
  readonly schemaVersion: 1;
  readonly id: ExperienceId;
  readonly mode: BrainSoundMode;
  readonly title: string;
  readonly description: string;
  readonly tags: readonly string[];
  readonly intensities: readonly Intensity[];
  readonly durationSeconds: number;
  readonly loop: boolean;
  readonly assets: readonly CatalogAsset[];
  readonly intensityProfiles: readonly IntensityProfile[];
}

export interface CatalogIndex {
  readonly schemaVersion: 1;
  readonly catalogVersion: string;
  readonly generatedAt: string;
  readonly experiences: readonly string[];
}

export interface ValidationIssue {
  readonly path: string;
  readonly code: 'required' | 'format' | 'duplicate' | 'integrity' | 'budget';
  readonly message: string;
}

export interface ValidationResult<T> {
  readonly valid: boolean;
  readonly value: T | null;
  readonly issues: readonly ValidationIssue[];
}

export function isExperienceId(value: string): value is ExperienceId {
  return (experienceIds as readonly string[]).includes(value);
}
```

- [ ] **Step 4: Verificar GREEN**

Run: `npm run test -- src/catalog/catalog-types.test.ts`

Expected: dos pruebas PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/catalog/catalog-types.ts src/catalog/catalog-types.test.ts
git commit -m "feat: define versioned catalog contracts"
```

### Task 3: Validar JSON desconocido y devolver todos los problemas

**Files:**
- Create: `src/catalog/catalog-schema.test.ts`
- Create: `src/catalog/catalog-schema.ts`

- [ ] **Step 1: Escribir casos RED estructurales**

Create `src/catalog/catalog-schema.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { parseCatalogExperience } from './catalog-schema';

describe('parseCatalogExperience', () => {
  it('acumula campos requeridos y formatos inválidos', () => {
    const result = parseCatalogExperience({ schemaVersion: 2, id: 'sleep-01', assets: [] }, 'fixture.json');
    expect(result.valid).toBe(false);
    expect(result.issues.map(({ path }) => path)).toEqual(expect.arrayContaining([
      'fixture.json.schemaVersion', 'fixture.json.id', 'fixture.json.mode', 'fixture.json.title',
    ]));
  });

  it('rechaza rutas inseguras y perfiles fuera de rango', () => {
    const result = parseCatalogExperience({
      schemaVersion: 1,
      id: 'deep-focus-01', mode: 'deep-focus', title: 'Focus', description: 'Focus estable',
      tags: ['focus'], intensities: ['soft', 'medium', 'deep'], durationSeconds: 60, loop: true,
      assets: [{
        id: 'asset-1', path: '../secret.wav', kind: 'music', bytes: 10, sha256: 'x',
        sourceUrl: '', author: '', licenseId: '', licenseUrl: '', attribution: '', verifiedOn: '2099-01-01',
      }],
      intensityProfiles: [{
        intensity: 'soft', masterGain: 1, filterHz: 800, crossfadeSeconds: 4,
        layers: [{ assetId: 'missing', gain: 2, playbackRate: 2 }],
      }],
    }, 'fixture.json', new Date('2026-08-07T00:00:00Z'));
    expect(result.valid).toBe(false);
    expect(result.issues.length).toBeGreaterThan(5);
  });
});
```

- [ ] **Step 2: Ejecutar RED**

Run: `npm run test -- src/catalog/catalog-schema.test.ts`

Expected: FAIL porque el parser no existe.

- [ ] **Step 3: Implementar validadores pequeños y parser**

Create `src/catalog/catalog-schema.ts` with these exported entry points and exact invariants:

```ts
import {
  experienceIds, isExperienceId,
  type CatalogExperience, type CatalogIndex, type ValidationIssue, type ValidationResult,
} from './catalog-types';

const modes = ['deep-focus', 'creativity', 'relaxation'] as const;
const intensities = ['soft', 'medium', 'deep'] as const;
const kinds = ['music', 'ambient', 'nature'] as const;
const safePath = /^(?!.*(?:^|\/)\.\.(?:\/|$))[a-z0-9][a-z0-9/_-]*\.[a-z0-9]+$/;
const sha256 = /^[a-f0-9]{64}$/;
const catalogVersion = /^\d{4}\.\d{2}\.\d{2}\.\d+$/;

function object(value: unknown): Record<string, unknown> | null {
  return typeof value === 'object' && value !== null && !Array.isArray(value)
    ? value as Record<string, unknown> : null;
}

function issue(issues: ValidationIssue[], path: string, code: ValidationIssue['code'], message: string): void {
  issues.push({ path, code, message });
}

export function parseCatalogExperience(
  input: unknown,
  sourcePath: string,
  now = new Date(),
): ValidationResult<CatalogExperience> {
  const issues: ValidationIssue[] = [];
  const value = object(input);
  if (value === null) return { valid: false, value: null, issues: [{ path: sourcePath, code: 'format', message: 'Debe ser objeto' }] };
  if (value.schemaVersion !== 1) issue(issues, `${sourcePath}.schemaVersion`, 'format', 'Debe ser 1');
  if (typeof value.id !== 'string' || !isExperienceId(value.id)) issue(issues, `${sourcePath}.id`, 'format', 'ID fuera del contrato');
  if (typeof value.mode !== 'string' || !modes.includes(value.mode as typeof modes[number])) issue(issues, `${sourcePath}.mode`, 'format', 'Modo inválido');
  for (const field of ['title', 'description'] as const) if (typeof value[field] !== 'string' || value[field].trim() === '') issue(issues, `${sourcePath}.${field}`, 'required', 'Texto requerido');
  if (!Array.isArray(value.tags) || value.tags.some((tag) => typeof tag !== 'string')) issue(issues, `${sourcePath}.tags`, 'format', 'Tags inválidos');
  const inputIntensities = value.intensities;
  if (!Array.isArray(inputIntensities) || intensities.some((entry) => !inputIntensities.includes(entry))) issue(issues, `${sourcePath}.intensities`, 'format', 'Requiere soft, medium y deep');
  if (typeof value.durationSeconds !== 'number' || value.durationSeconds <= 0) issue(issues, `${sourcePath}.durationSeconds`, 'format', 'Duración positiva requerida');
  if (typeof value.loop !== 'boolean') issue(issues, `${sourcePath}.loop`, 'format', 'Booleano requerido');

  const assets = Array.isArray(value.assets) ? value.assets : [];
  if (assets.length === 0) issue(issues, `${sourcePath}.assets`, 'required', 'Se requiere al menos un asset');
  const assetIds = new Set<string>();
  for (const [index, candidate] of assets.entries()) {
    const asset = object(candidate); const base = `${sourcePath}.assets.${index}`;
    if (asset === null) { issue(issues, base, 'format', 'Asset debe ser objeto'); continue; }
    if (typeof asset.id !== 'string' || asset.id === '') issue(issues, `${base}.id`, 'required', 'ID requerido');
    else if (assetIds.has(asset.id)) issue(issues, `${base}.id`, 'duplicate', 'ID duplicado'); else assetIds.add(asset.id);
    if (typeof asset.path !== 'string' || !safePath.test(asset.path)) issue(issues, `${base}.path`, 'format', 'Ruta relativa insegura');
    if (typeof asset.kind !== 'string' || !kinds.includes(asset.kind as typeof kinds[number])) issue(issues, `${base}.kind`, 'format', 'Kind inválido');
    if (typeof asset.bytes !== 'number' || !Number.isInteger(asset.bytes) || asset.bytes <= 0) issue(issues, `${base}.bytes`, 'format', 'Bytes positivos requeridos');
    if (typeof asset.sha256 !== 'string' || !sha256.test(asset.sha256)) issue(issues, `${base}.sha256`, 'format', 'SHA-256 inválido');
    for (const field of ['sourceUrl', 'author', 'licenseId', 'licenseUrl', 'attribution'] as const) if (typeof asset[field] !== 'string' || asset[field].trim() === '') issue(issues, `${base}.${field}`, 'required', 'Campo requerido');
    if (typeof asset.verifiedOn !== 'string' || !/^\d{4}-\d{2}-\d{2}$/.test(asset.verifiedOn) || new Date(`${asset.verifiedOn}T00:00:00Z`) > now) issue(issues, `${base}.verifiedOn`, 'format', 'Fecha inválida o futura');
  }

  const profiles = Array.isArray(value.intensityProfiles) ? value.intensityProfiles : [];
  for (const intensity of intensities) if (!profiles.some((profile) => object(profile)?.intensity === intensity)) issue(issues, `${sourcePath}.intensityProfiles`, 'required', `Falta ${intensity}`);
  for (const [index, candidate] of profiles.entries()) {
    const profile = object(candidate); const base = `${sourcePath}.intensityProfiles.${index}`;
    if (profile === null) { issue(issues, base, 'format', 'Perfil debe ser objeto'); continue; }
    if (typeof profile.masterGain !== 'number' || profile.masterGain < 0 || profile.masterGain > 0.9) issue(issues, `${base}.masterGain`, 'format', 'Fuera de 0..0.9');
    if (typeof profile.crossfadeSeconds !== 'number' || profile.crossfadeSeconds < 8 || profile.crossfadeSeconds > 12) issue(issues, `${base}.crossfadeSeconds`, 'format', 'Fuera de 8..12');
    const layers = Array.isArray(profile.layers) ? profile.layers : [];
    if (layers.length === 0) issue(issues, `${base}.layers`, 'required', 'Capas requeridas');
    for (const [layerIndex, layerValue] of layers.entries()) {
      const layer = object(layerValue); const layerPath = `${base}.layers.${layerIndex}`;
      if (layer === null) { issue(issues, layerPath, 'format', 'Capa debe ser objeto'); continue; }
      if (typeof layer.assetId !== 'string' || !assetIds.has(layer.assetId)) issue(issues, `${layerPath}.assetId`, 'format', 'Asset no resuelto');
      if (typeof layer.gain !== 'number' || layer.gain < 0 || layer.gain > 1) issue(issues, `${layerPath}.gain`, 'format', 'Fuera de 0..1');
      if (typeof layer.playbackRate !== 'number' || layer.playbackRate < 0.95 || layer.playbackRate > 1.05) issue(issues, `${layerPath}.playbackRate`, 'format', 'Fuera de 0.95..1.05');
    }
  }
  issues.sort((a, b) => a.path.localeCompare(b.path) || a.code.localeCompare(b.code));
  return issues.length === 0
    ? { valid: true, value: value as unknown as CatalogExperience, issues: [] }
    : { valid: false, value: null, issues };
}

export function parseCatalogIndex(input: unknown, sourcePath: string): ValidationResult<CatalogIndex> {
  const issues: ValidationIssue[] = []; const value = object(input);
  if (value === null) return { valid: false, value: null, issues: [{ path: sourcePath, code: 'format', message: 'Debe ser objeto' }] };
  if (value.schemaVersion !== 1) issue(issues, `${sourcePath}.schemaVersion`, 'format', 'Debe ser 1');
  if (typeof value.catalogVersion !== 'string' || !catalogVersion.test(value.catalogVersion)) issue(issues, `${sourcePath}.catalogVersion`, 'format', 'Versión inválida');
  if (typeof value.generatedAt !== 'string' || Number.isNaN(Date.parse(value.generatedAt))) issue(issues, `${sourcePath}.generatedAt`, 'format', 'Fecha ISO inválida');
  if (!Array.isArray(value.experiences) || value.experiences.length !== experienceIds.length) issue(issues, `${sourcePath}.experiences`, 'format', 'Requiere 15 rutas');
  return issues.length === 0 ? { valid: true, value: value as unknown as CatalogIndex, issues: [] } : { valid: false, value: null, issues };
}
```

- [ ] **Step 4: Ejecutar GREEN y typecheck**

```powershell
npm run test -- src/catalog/catalog-schema.test.ts
npm run typecheck
```

Expected: dos pruebas PASS y typecheck `0`.

- [ ] **Step 5: Commit**

```powershell
git add src/catalog/catalog-schema.ts src/catalog/catalog-schema.test.ts
git commit -m "feat: validate unknown catalog manifests"
```

### Task 4: Implementar búsqueda determinista y filtros

**Files:**
- Create: `src/catalog/catalog-search.test.ts`
- Create: `src/catalog/catalog-search.ts`

- [ ] **Step 1: Escribir RED con diacríticos y modo**

Create `src/catalog/catalog-search.test.ts` with a typed fixture and assertions:

```ts
import { describe, expect, it } from 'vitest';
import type { CatalogExperience } from './catalog-types';
import { searchCatalog } from './catalog-search';

const items = [
  { id: 'creativity-01', mode: 'creativity', title: 'Inspiración cálida', tags: ['ideas', 'energía'] },
  { id: 'deep-focus-01', mode: 'deep-focus', title: 'Concentración estable', tags: ['trabajo'] },
] as unknown as readonly CatalogExperience[];

describe('searchCatalog', () => {
  it('normaliza mayúsculas y diacríticos', () => {
    expect(searchCatalog(items, { query: 'INSPIRACION', mode: null }).map(({ id }) => id)).toEqual(['creativity-01']);
  });
  it('combina modo y tag', () => {
    expect(searchCatalog(items, { query: 'trabajo', mode: 'deep-focus' }).map(({ id }) => id)).toEqual(['deep-focus-01']);
  });
});
```

- [ ] **Step 2: Ejecutar RED**

Run: `npm run test -- src/catalog/catalog-search.test.ts`

Expected: FAIL por módulo ausente.

- [ ] **Step 3: Implementar búsqueda pura**

Create `src/catalog/catalog-search.ts`:

```ts
import type { BrainSoundMode } from '../shared/domain/mode';
import type { CatalogExperience } from './catalog-types';

export interface CatalogQuery { readonly query: string; readonly mode: BrainSoundMode | null; }
const normalize = (value: string) => value.normalize('NFD').replace(/\p{Diacritic}/gu, '').toLocaleLowerCase('es');

export function searchCatalog(items: readonly CatalogExperience[], filter: CatalogQuery): readonly CatalogExperience[] {
  const query = normalize(filter.query.trim());
  return items.filter((item) => {
    if (filter.mode !== null && item.mode !== filter.mode) return false;
    const haystack = normalize([item.title, item.description, ...item.tags].join(' '));
    return query === '' || haystack.includes(query);
  }).sort((a, b) => a.id.localeCompare(b.id));
}
```

- [ ] **Step 4: Verificar y commit**

```powershell
npm run test -- src/catalog/catalog-search.test.ts
git add src/catalog/catalog-search.ts src/catalog/catalog-search.test.ts
git commit -m "feat: add deterministic catalog search"
```

Expected: dos pruebas PASS y commit creado.

### Task 5: Verificar archivos, hashes y presupuesto desde Node

**Files:**
- Create: `scripts/catalog-files.test.ts`
- Create: `scripts/catalog-files.ts`
- Modify: `vitest.config.ts`
- Modify: `tsconfig.node.json`

- [ ] **Step 1: Escribir RED de integridad y deduplicación**

Create `scripts/catalog-files.test.ts`:

```ts
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { describe, expect, it } from 'vitest';
import { inspectCatalogFiles } from './catalog-files';

describe('inspectCatalogFiles', () => {
  it('detecta bytes/hash y cuenta una ruta compartida una vez', async () => {
    const root = await mkdtemp(join(tmpdir(), 'brainsound-catalog-'));
    await writeFile(join(root, 'tone.bin'), new Uint8Array([1, 2, 3]));
    const asset = { path: 'tone.bin', bytes: 4, sha256: '0'.repeat(64) };
    const result = await inspectCatalogFiles(root, [asset, asset]);
    expect(result.totalUniqueBytes).toBe(3);
    expect(result.issues.map(({ code }) => code)).toEqual(['integrity', 'integrity']);
  });
});
```

- [ ] **Step 2: Incluir tests de scripts y ejecutar RED**

Add `'scripts/**/*.test.ts'` to `test.include` in `vitest.config.ts` and add `"scripts"` to `tsconfig.node.json` `include`, then run:

```powershell
npm run test -- scripts/catalog-files.test.ts
```

Expected: FAIL porque `catalog-files.ts` no existe.

- [ ] **Step 3: Implementar inspección sin mutaciones**

Create `scripts/catalog-files.ts`:

```ts
import { createHash } from 'node:crypto';
import { readFile } from 'node:fs/promises';
import { resolve } from 'node:path';
import type { ValidationIssue } from '../src/catalog/catalog-types';

interface FileAsset { readonly path: string; readonly bytes: number; readonly sha256: string; }

export async function inspectCatalogFiles(root: string, assets: readonly FileAsset[]) {
  const issues: ValidationIssue[] = []; const observed = new Map<string, number>();
  const uniqueAssets = new Map(assets.map((asset) => [asset.path, asset]));
  for (const asset of uniqueAssets.values()) {
    const absolute = resolve(root, asset.path);
    try {
      const bytes = await readFile(absolute); observed.set(asset.path, bytes.byteLength);
      if (bytes.byteLength !== asset.bytes) issues.push({ path: asset.path, code: 'integrity', message: `Bytes esperados ${asset.bytes}, observados ${bytes.byteLength}` });
      const digest = createHash('sha256').update(bytes).digest('hex');
      if (digest !== asset.sha256) issues.push({ path: asset.path, code: 'integrity', message: `SHA-256 observado ${digest}` });
    } catch (error) {
      issues.push({ path: asset.path, code: 'integrity', message: error instanceof Error ? error.message : 'Archivo ilegible' });
    }
  }
  return { issues, totalUniqueBytes: [...observed.values()].reduce((sum, bytes) => sum + bytes, 0) };
}
```

- [ ] **Step 4: Verificar y commit**

```powershell
npm run test -- scripts/catalog-files.test.ts
npm run typecheck
git add scripts/catalog-files.ts scripts/catalog-files.test.ts vitest.config.ts
git commit -m "feat: verify catalog file integrity and budget"
```

Expected: una prueba PASS, typecheck `0` y commit creado.

### Task 6: Crear la compuerta CLI de catálogo completo

**Files:**
- Create: `scripts/validate-catalog.ts`
- Modify: `package.json`

- [ ] **Step 1: Implementar el CLI como composición de validadores probados**

Create `scripts/validate-catalog.ts`:

```ts
import { readFile, readdir } from 'node:fs/promises';
import { resolve } from 'node:path';
import { inspectCatalogFiles } from './catalog-files';
import { parseCatalogExperience, parseCatalogIndex } from '../src/catalog/catalog-schema';
import type { CatalogAsset, CatalogExperience, ValidationIssue } from '../src/catalog/catalog-types';

const root = resolve('public/catalog');
const issues: ValidationIssue[] = [];
const readJson = async (path: string): Promise<unknown> => JSON.parse(await readFile(path, 'utf8')) as unknown;
const indexResult = parseCatalogIndex(await readJson(resolve(root, 'index.json')), 'index.json');
issues.push(...indexResult.issues);

const experiences: CatalogExperience[] = [];
if (indexResult.value !== null) {
  for (const relativePath of indexResult.value.experiences) {
    const result = parseCatalogExperience(await readJson(resolve(root, relativePath)), relativePath);
    issues.push(...result.issues);
    if (result.value !== null) experiences.push(result.value);
  }
}

const ids = experiences.map(({ id }) => id);
for (const duplicate of ids.filter((id, index) => ids.indexOf(id) !== index)) {
  issues.push({ path: duplicate, code: 'duplicate', message: 'ID de experiencia duplicado' });
}
const assets = experiences.flatMap(({ assets }) => assets) as CatalogAsset[];
const fileResult = await inspectCatalogFiles(resolve(root, 'assets'), assets);
issues.push(...fileResult.issues);
if (fileResult.totalUniqueBytes > 314_572_800) {
  issues.push({ path: 'catalog', code: 'budget', message: `Presupuesto excedido: ${fileResult.totalUniqueBytes}` });
}
issues.sort((a, b) => a.path.localeCompare(b.path) || a.code.localeCompare(b.code));
if (issues.length > 0) {
  for (const entry of issues) process.stderr.write(`${entry.code}\t${entry.path}\t${entry.message}\n`);
  process.exitCode = 1;
} else {
  const diskFiles = await readdir(resolve(root, 'experiences'));
  process.stdout.write(`Catálogo válido: ${experiences.length} experiencias, ${assets.length} referencias, ${fileResult.totalUniqueBytes} bytes, ${diskFiles.length} manifiestos\n`);
}
```

- [ ] **Step 2: Cambiar scripts de package.json**

Set these exact entries:

```json
{
  "scripts": {
    "validate:catalog": "node scripts/validate-catalog.ts",
    "report:catalog": "node scripts/report-catalog.ts"
  }
}
```

Preserve every existing script not shown.

- [ ] **Step 3: Ejecutar sobre ausencia de catálogo para confirmar fallo controlado**

Run: `npm run validate:catalog`

Expected: FAIL con ruta `public/catalog/index.json`; no debe crear archivos.

- [ ] **Step 4: Commit**

```powershell
git add scripts/validate-catalog.ts package.json package-lock.json
git commit -m "feat: add catalog validation command"
```

### Task 7: Materializar quince manifiestos exclusivamente desde evidencia aprobada

**Files:**
- Create: `public/catalog/index.json`
- Create: `public/catalog/experiences/deep-focus-01.json` through `deep-focus-05.json`
- Create: `public/catalog/experiences/creativity-01.json` through `creativity-05.json`
- Create: `public/catalog/experiences/relaxation-01.json` through `relaxation-05.json`
- Create: `public/catalog/assets/**`
- Create: `docs/catalog/approved-assets.json`
- Create: `scripts/materialize-catalog.ts`

- [ ] **Step 1: Copiar solo binarios con fila APROBADO**

For every approved row in `docs/catalog/asset-register.md`, read its `kind`, create the matching `public/catalog/assets/music`, `ambient` or `nature` directory, and copy the observed reviewed file using its approved kebab-case asset ID. Immediately recompute:

```powershell
Get-ChildItem -LiteralPath public/catalog/assets -File -Recurse | Get-FileHash -Algorithm SHA256
Get-ChildItem -LiteralPath public/catalog/assets -File -Recurse | Measure-Object -Property Length -Sum
```

Expected: hashes equal the approved rows; sum is at most `314572800`. Any mismatch returns the asset to `CANDIDATO` and blocks this task.

- [ ] **Step 2: Crear el índice exacto de rutas estables**

Create `public/catalog/index.json` with schema/version/date observed on execution and this exact ordered experiences array:

```json
{
  "schemaVersion": 1,
  "catalogVersion": "2026.08.07.1",
  "generatedAt": "2026-08-07T00:00:00.000Z",
  "experiences": [
    "experiences/deep-focus-01.json", "experiences/deep-focus-02.json", "experiences/deep-focus-03.json", "experiences/deep-focus-04.json", "experiences/deep-focus-05.json",
    "experiences/creativity-01.json", "experiences/creativity-02.json", "experiences/creativity-03.json", "experiences/creativity-04.json", "experiences/creativity-05.json",
    "experiences/relaxation-01.json", "experiences/relaxation-02.json", "experiences/relaxation-03.json", "experiences/relaxation-04.json", "experiences/relaxation-05.json"
  ]
}
```

If execution date differs, update both date fields according to the SPEC and keep the array unchanged.

- [ ] **Step 3: Convertir evidencia aprobada a JSON sin datos inventados**

Create `docs/catalog/approved-assets.json` as a JSON array whose entries match `CatalogAsset` exactly. Every value must be copied from an `APROBADO` row and the observed file; no sample row is permitted. Validate the file before continuing:

```powershell
node -e "const a=require('./docs/catalog/approved-assets.json');if(!Array.isArray(a)||a.length<3||!['music','ambient','nature'].every(k=>a.some(x=>x.kind===k)))process.exit(1)"
```

Expected: exit code `0`, at least one approved asset for each kind.

- [ ] **Step 4: Generar los quince manifiestos de forma determinista**

Create `scripts/materialize-catalog.ts`:

```ts
import { mkdir, readFile, writeFile } from 'node:fs/promises';
import { resolve } from 'node:path';
import { experienceIds, type CatalogAsset, type CatalogExperience } from '../src/catalog/catalog-types';

const assets = JSON.parse(await readFile(resolve('docs/catalog/approved-assets.json'), 'utf8')) as CatalogAsset[];
const byKind = (kind: CatalogAsset['kind']) => assets.filter((asset) => asset.kind === kind).sort((a, b) => a.id.localeCompare(b.id));
if (['music', 'ambient', 'nature'].some((kind) => byKind(kind as CatalogAsset['kind']).length === 0)) throw new Error('Se requiere al menos un asset aprobado por kind');
const labels = { 'deep-focus': 'Concentración', creativity: 'Creatividad', relaxation: 'Relajación' } as const;
const tags = { 'deep-focus': ['concentracion','trabajo'], creativity: ['ideas','energia'], relaxation: ['pausa','calma'] } as const;
const manifests = experienceIds.map((id, index): CatalogExperience => {
  const mode = id.startsWith('deep-focus') ? 'deep-focus' : id.startsWith('creativity') ? 'creativity' : 'relaxation';
  const selected = (['music','ambient','nature'] as const).map((kind) => byKind(kind)[index % byKind(kind).length]!);
  const number = Number(id.slice(-2));
  const layers = (gain: number) => selected.map((asset) => ({ assetId: asset.id, gain, playbackRate: 1 }));
  return { schemaVersion: 1, id, mode, title: `${labels[mode]} ${number}`, description: `Capas locales para ${labels[mode].toLocaleLowerCase('es')}.`, tags: tags[mode], intensities: ['soft','medium','deep'], durationSeconds: 300, loop: true, assets: selected, intensityProfiles: [
    { intensity:'soft', masterGain:0.45, filterHz:900, crossfadeSeconds:10, layers:layers(0.35) },
    { intensity:'medium', masterGain:0.6, filterHz:1400, crossfadeSeconds:10, layers:layers(0.5) },
    { intensity:'deep', masterGain:0.75, filterHz:2000, crossfadeSeconds:10, layers:layers(0.65) },
  ] };
});
await mkdir(resolve('public/catalog/experiences'), { recursive: true });
await Promise.all(manifests.map((manifest) => writeFile(resolve(`public/catalog/experiences/${manifest.id}.json`), `${JSON.stringify(manifest, null, 2)}\n`, 'utf8')));
```

Run: `node scripts/materialize-catalog.ts`

Expected: exactly 15 manifest files; all asset metadata comes from `approved-assets.json`.

- [ ] **Step 5: Validar el conjunto completo**

Run: `npm run validate:catalog`

Expected: `Catálogo válido: 15 experiencias` and exit code `0`.

- [ ] **Step 6: Commit allowlist**

```powershell
git add public/catalog docs/catalog/asset-register.md docs/catalog/approved-assets.json scripts/materialize-catalog.ts
git commit -m "content: add licensed BrainSound catalog manifests"
```

### Task 8: Generar reporte determinista de licencias y presupuesto

**Files:**
- Create: `scripts/report-catalog.ts`
- Create: `reports/catalog/catalog-validation.md`

- [ ] **Step 1: Implementar reporte desde manifiestos validados**

Create `scripts/report-catalog.ts`:

```ts
import { mkdir, readFile, writeFile } from 'node:fs/promises';
import { resolve } from 'node:path';
import type { CatalogExperience } from '../src/catalog/catalog-types';

const root = resolve('public/catalog');
const index = JSON.parse(await readFile(resolve(root, 'index.json'), 'utf8')) as { catalogVersion: string; experiences: string[] };
const experiences = await Promise.all(index.experiences.map(async (path) => JSON.parse(await readFile(resolve(root, path), 'utf8')) as CatalogExperience));
const assets = new Map(experiences.flatMap(({ assets }) => assets).map((asset) => [asset.path, asset]));
const sorted = [...assets.values()].sort((a, b) => a.path.localeCompare(b.path));
const total = sorted.reduce((sum, asset) => sum + asset.bytes, 0);
const lines = [
  '# Reporte de catálogo BrainSound', '',
  `- Versión: ${index.catalogVersion}`, `- Experiencias: ${experiences.length}`,
  `- Activos únicos: ${sorted.length}`, `- Bytes únicos: ${total}`, '',
  '| Path | Kind | Author | License | Attribution | Bytes | SHA-256 |',
  '|---|---|---|---|---|---:|---|',
  ...sorted.map((asset) => `| ${asset.path} | ${asset.kind} | ${asset.author} | [${asset.licenseId}](${asset.licenseUrl}) | ${asset.attribution} | ${asset.bytes} | ${asset.sha256} |`),
  '',
];
await mkdir(resolve('reports/catalog'), { recursive: true });
await writeFile(resolve('reports/catalog/catalog-validation.md'), lines.join('\n'), 'utf8');
```

- [ ] **Step 2: Generar dos veces y comprobar determinismo**

```powershell
npm run report:catalog
git hash-object reports/catalog/catalog-validation.md
npm run report:catalog
git hash-object reports/catalog/catalog-validation.md
```

Expected: ambos hashes son idénticos.

- [ ] **Step 3: Commit**

```powershell
git add scripts/report-catalog.ts reports/catalog/catalog-validation.md package.json package-lock.json
git commit -m "docs: generate catalog licensing report"
```

### Task 9: Cargar únicamente catálogo validado en la aplicación

**Files:**
- Create: `src/catalog/catalog-repository.test.ts`
- Create: `src/catalog/catalog-repository.ts`

- [ ] **Step 1: Escribir RED para éxito y rechazo**

Create `src/catalog/catalog-repository.test.ts` using injected fetch responses:

```ts
import { describe, expect, it, vi } from 'vitest';
import { CatalogRepository } from './catalog-repository';

describe('CatalogRepository', () => {
  it('rechaza un índice inválido sin exponer catálogo parcial', async () => {
    const fetchJson = vi.fn().mockResolvedValue({ schemaVersion: 2 });
    const repository = new CatalogRepository(fetchJson);
    await expect(repository.load()).rejects.toThrow('Catálogo inválido');
    expect(repository.getSnapshot()).toEqual([]);
  });
});
```

- [ ] **Step 2: Ejecutar RED**

Run: `npm run test -- src/catalog/catalog-repository.test.ts`

Expected: FAIL por clase ausente.

- [ ] **Step 3: Implementar repositorio atómico**

Create `src/catalog/catalog-repository.ts`:

```ts
import { parseCatalogExperience, parseCatalogIndex } from './catalog-schema';
import type { CatalogExperience } from './catalog-types';

type FetchJson = (path: string) => Promise<unknown>;
const browserFetchJson: FetchJson = async (path) => {
  const response = await fetch(path);
  if (!response.ok) throw new Error(`No se pudo leer ${path}: ${response.status}`);
  return response.json() as Promise<unknown>;
};

export class CatalogRepository {
  private snapshot: readonly CatalogExperience[] = [];
  constructor(private readonly fetchJson: FetchJson = browserFetchJson) {}
  getSnapshot = (): readonly CatalogExperience[] => this.snapshot;

  async load(): Promise<readonly CatalogExperience[]> {
    const index = parseCatalogIndex(await this.fetchJson('/catalog/index.json'), 'index.json');
    if (index.value === null) throw new Error('Catálogo inválido: índice');
    const next: CatalogExperience[] = [];
    for (const path of index.value.experiences) {
      const parsed = parseCatalogExperience(await this.fetchJson(`/catalog/${path}`), path);
      if (parsed.value === null) throw new Error(`Catálogo inválido: ${path}`);
      next.push(parsed.value);
    }
    this.snapshot = next;
    return this.snapshot;
  }
}
```

- [ ] **Step 4: Verificar y commit**

```powershell
npm run test -- src/catalog/catalog-repository.test.ts
npm run typecheck
git add src/catalog/catalog-repository.ts src/catalog/catalog-repository.test.ts
git commit -m "feat: load catalog atomically"
```

Expected: prueba PASS y typecheck `0`.

### Task 10: Construir Explorar sin activar audio híbrido

**Files:**
- Create: `src/features/explore/ExperienceCard.tsx`
- Create: `src/features/explore/ExploreScreen.tsx`
- Modify: `src/app/App.tsx`
- Modify: `src/styles/global.css`
- Create: `e2e/catalog.spec.ts`

- [ ] **Step 1: Escribir E2E RED**

Create `e2e/catalog.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('muestra quince experiencias y filtra por modo y texto', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('button', { name: 'Explorar' }).click();
  await expect(page.getByRole('article')).toHaveCount(15);
  await page.getByRole('button', { name: 'Creatividad' }).click();
  await expect(page.getByRole('article')).toHaveCount(5);
  await page.getByRole('searchbox', { name: 'Buscar experiencias' }).fill('texto observado del manifiesto');
  await expect(page.getByText('No encontramos experiencias')).toBeVisible();
});
```

- [ ] **Step 2: Ejecutar RED**

Run: `npx playwright test e2e/catalog.spec.ts --project=chromium`

Expected: FAIL porque no existe navegación Explorar.

- [ ] **Step 3: Crear componentes accesibles**

Create `src/features/explore/ExperienceCard.tsx`:

```tsx
import type { CatalogExperience } from '../../catalog/catalog-types';
export function ExperienceCard({ experience, onPlay }: { readonly experience: CatalogExperience; readonly onPlay: (experience: CatalogExperience) => void }) {
  return <article className="experience-card"><h3>{experience.title}</h3><p>{experience.description}</p><button type="button" onClick={() => onPlay(experience)}>Escuchar {experience.title}</button></article>;
}
```

Create `src/features/explore/ExploreScreen.tsx`:

```tsx
import { useMemo, useState } from 'react';
import { searchCatalog } from '../../catalog/catalog-search';
import type { CatalogExperience } from '../../catalog/catalog-types';
import type { BrainSoundMode } from '../../shared/domain/mode';
import { ExperienceCard } from './ExperienceCard';

export function ExploreScreen({ catalog, onPlay }: { readonly catalog: readonly CatalogExperience[]; readonly onPlay: (item: CatalogExperience) => void }) {
  const [query, setQuery] = useState(''); const [mode, setMode] = useState<BrainSoundMode | null>(null);
  const results = useMemo(() => searchCatalog(catalog, { query, mode }), [catalog, query, mode]);
  return <main className="app-shell"><h1>Explorar</h1><label>Buscar experiencias<input aria-label="Buscar experiencias" type="search" value={query} onChange={(event) => setQuery(event.target.value)} /></label><div className="filter-row"><button onClick={() => setMode(null)}>Todas</button><button onClick={() => setMode('deep-focus')}>Deep Focus</button><button onClick={() => setMode('creativity')}>Creatividad</button><button onClick={() => setMode('relaxation')}>Relajación</button></div>{results.length === 0 ? <p>No encontramos experiencias</p> : <section className="experience-grid">{results.map((item) => <ExperienceCard key={item.id} experience={item} onPlay={onPlay} />)}</section>}</main>;
}
```

- [ ] **Step 4: Integrar carga/navegación y mantener fallback**

In `src/app/App.tsx`, instantiate `CatalogRepository`, load in `useEffect`, add view state `'home' | 'explore'`, and when `onPlay` receives a catalog experience call:

```ts
void session.start(getFallbackExperience(experience.mode));
```

Do not reference `experience.assets` in I2. Add buttons `Inicio` and `Explorar`, loading/error states, and pass the loaded snapshot to `ExploreScreen`.

- [ ] **Step 5: Estilos y verificación**

Append compact grid/search/focus styles to `src/styles/global.css`, then run:

```powershell
npx playwright test e2e/catalog.spec.ts
npm run test
npm run typecheck
```

Expected: E2E PASS en Chromium/WebKit, unitarias PASS y typecheck `0`.

- [ ] **Step 6: Commit**

```powershell
git add src/app/App.tsx src/features/explore src/styles/global.css e2e/catalog.spec.ts
git commit -m "feat: explore validated BrainSound catalog"
```

### Task 11: Probar degradación ante catálogo inválido y cerrar I2

**Files:**
- Create: `e2e/catalog-fallback.spec.ts`
- Modify: `README.md`
- Modify: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Modify: `docs/plans/2026-08-07-brainsound-i2-catalog.md`

- [ ] **Step 1: Crear E2E de fallo atómico**

Create `e2e/catalog-fallback.spec.ts`:

```ts
import { expect, test } from '@playwright/test';
test('un catálogo inválido no impide usar el fallback de inicio', async ({ page }) => {
  await page.route('**/catalog/index.json', (route) => route.fulfill({ json: { schemaVersion: 2 } }));
  await page.goto('/');
  await expect(page.getByRole('alert')).toContainText('catálogo');
  await page.getByRole('button', { name: /Deep Focus/ }).click();
  await expect(page.getByRole('heading', { name: 'Pulso profundo en curso' })).toBeVisible();
});
```

- [ ] **Step 2: Ejecutar verificación completa**

```powershell
npm run validate:catalog
npm run report:catalog
npm run verify
npm run test:e2e
git diff --check
```

Expected: catálogo válido de 15, reporte determinista, cobertura crítica mínima 90%, E2E Chromium/WebKit PASS y diff check limpio.

- [ ] **Step 3: Documentar evidencia real**

In `README.md`, document `npm run validate:catalog`, `npm run report:catalog`, location of license report, and explicitly state that I2 still plays the I1 fallback. Append to this plan a `## Resultados de ejecución` section populated only with observed commit, test counts, bytes and report hash.

- [ ] **Step 4: Cerrar estado solo con evidencia**

Change I2 in the roadmap to `Completada` only if every gate passes; otherwise use `En ejecución` and record the exact blocker.

- [ ] **Step 5: Commit**

```powershell
git add e2e/catalog-fallback.spec.ts README.md docs/plans reports/catalog
git commit -m "docs: close BrainSound catalog iteration"
```

## Trazabilidad SPEC → tareas

| Criterio I2 | Tareas |
|---|---|
| Evidencia primaria/licencias | 1, 7, 8 |
| 15 IDs y tres perfiles | 2, 3, 7 |
| Integridad y 300 MB | 5, 6, 7 |
| Reporte determinista | 8 |
| Repositorio atómico | 9 |
| Explorar/filtro/búsqueda | 4, 10 |
| Fallback ante inválido | 9, 11 |
| Verificación integral | 11 |

## Gate de aprobación antes de ejecutar

Gate de plan superado el 2026-08-07. I2 permanece bloqueada hasta cerrar I1. La selección de activos sigue sometida a evidencia primaria y revisión humana durante la ejecución.
