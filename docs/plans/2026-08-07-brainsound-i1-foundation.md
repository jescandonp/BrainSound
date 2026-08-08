# BrainSound I1 Foundation Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir una PWA instalable de BrainSound que permita iniciar con un clic cualquiera de los tres modos, reproducir un paisaje sintético local de respaldo, pausar o reanudar, medir el tiempo y recargar la aplicación sin conexión.

**Architecture:** La interfaz React depende de contratos de dominio y de un `AudioPort`, no de Web Audio directamente. Un `SessionController` coordina el adaptador sintético y un reloj monotónico; el Service Worker conserva el shell estático para offline. Esta iteración deja contratos estables para que I2 incorpore manifiestos y I3 reemplace el audio normal por capas híbridas sin reescribir la UI.

**Tech Stack:** Node.js 24.18.1 LTS, TypeScript 7.0.2, React 19.2.8, Vite 8.1.5, Vitest 4.1.10, Playwright 1.61.1, ESLint 10.8.0 y Web Audio/Service Worker nativos.

**Estado:** suspendido; no ejecutable hasta aprobar `docs/specs/2026-08-07-brainsound-i1-foundation-spec.md` y volver a revisar este plan.

---

## Contexto y compuertas

- SPEC maestra aprobada: `docs/specs/2026-08-07-brainsound-mvp-design.md`.
- SPEC de I1 propuesta: `docs/specs/2026-08-07-brainsound-i1-foundation-spec.md`; su aprobación es previa a la revisión de este plan.
- Hoja de ruta: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`.
- Ejecutar en un worktree nuevo con rama `feat/i1-foundation`; no implementar directamente sobre `master`.
- No agregar cuentas, backend, IndexedDB, audios externos, analítica, router o librería de estado en I1.
- El sintetizador creado aquí es la degradación segura permanente definida por la SPEC; no es código desechable.
- Un cambio de alcance vuelve primero a la SPEC. Un cambio interno de I1 actualiza primero este plan.

## Resultado demostrable de I1

1. La aplicación muestra Deep Focus, Creatividad y Relajación con identidad Energía cromática.
2. Un clic inicia un perfil sintético distinto para el modo elegido.
3. El reproductor compacto muestra modo, estado y tiempo; permite pausar, reanudar y detener.
4. El tiempo usa un reloj monotónico y no acumula deriva por intervalos.
5. Manifest y Service Worker permiten recargar el shell sin red después de la primera visita.
6. No se realizan solicitudes a dominios externos.
7. Lint, tipos, unitarias, cobertura, build, validación del catálogo de respaldo y E2E pasan.

## Mapa de archivos de I1

```text
.nvmrc                                      Versión Node fijada
package.json                                Scripts y dependencias exactas
package-lock.json                           Bloqueo reproducible
tsconfig.json                               Referencias TypeScript
tsconfig.app.json                           Compilación estricta de navegador
tsconfig.node.json                          Configuración de herramientas
eslint.config.js                            Reglas estáticas
vite.config.ts                              Build React
vitest.config.ts                            Unitarias y cobertura
playwright.config.ts                        E2E Chromium/WebKit
index.html                                  Entrada y metadatos PWA
public/manifest.webmanifest                 Manifest instalable
public/icons/brainsound.svg                 Icono original del proyecto
public/service-worker.js                    Caché offline del shell
src/main.tsx                                Bootstrap React y registro PWA
src/app/App.tsx                             Composición de inicio y sesión
src/shared/domain/mode.ts                   Contrato de los tres modos
src/catalog/fallback-catalog.ts             Tres perfiles sintéticos locales
src/audio/audio-port.ts                     Puerto independiente de Web Audio
src/audio/create-synth-plan.ts              Plan puro de síntesis
src/audio/web-audio-synth.ts                Adaptador Web Audio
src/features/home/ModeCard.tsx              Acción accesible de modo
src/features/home/HomeScreen.tsx            Inicio con un clic
src/features/player/session-controller.ts   Estado y coordinación
src/features/player/use-session-controller.ts Adaptador React
src/features/player/PlayerPanel.tsx          Reproductor compacto
src/features/timer/elapsed-timer.ts          Reloj monotónico
src/features/timer/use-session-clock.ts      Refresco visual del tiempo
src/pwa/register-service-worker.ts           Registro del Service Worker
src/styles/tokens.css                        Colores, tipografía y medidas
src/styles/global.css                        Layout, foco y movimiento reducido
e2e/home.spec.ts                             Inicio y modos
e2e/session.spec.ts                          Reproductor vertical
e2e/offline.spec.ts                          Recarga sin red
e2e/quality.spec.ts                          Teclado, movimiento y red externa
README.md                                    Uso y verificación
```

### Task 1: Fijar runtime, dependencias y scripts

**Files:**
- Create: `.nvmrc`
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `tsconfig.app.json`
- Create: `tsconfig.node.json`
- Create: `package-lock.json` mediante `npm install`

- [ ] **Step 1: Verificar el runtime requerido**

Run:

```powershell
node --version
npm --version
```

Expected: Node imprime `v24.18.1`; npm finaliza con exit code `0`. Si Node difiere, ejecutar `nvm install 24.18.1; nvm use 24.18.1` y repetir.

- [ ] **Step 2: Fijar Node y crear el manifiesto del proyecto**

Create `.nvmrc`:

```text
24.18.1
```

Create `package.json`:

```json
{
  "name": "brainsound",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "engines": {
    "node": "24.18.1"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview --host 127.0.0.1",
    "lint": "eslint . --max-warnings=0",
    "typecheck": "tsc -b --pretty false",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "validate:catalog": "vitest run src/catalog/fallback-catalog.test.ts",
    "verify": "npm run lint && npm run typecheck && npm run test:coverage && npm run build && npm run validate:catalog"
  },
  "dependencies": {
    "react": "19.2.8",
    "react-dom": "19.2.8"
  },
  "devDependencies": {
    "@eslint/js": "10.0.1",
    "@playwright/test": "1.61.1",
    "@types/node": "24.13.3",
    "@types/react": "19.2.17",
    "@types/react-dom": "19.2.3",
    "@vitejs/plugin-react": "6.0.4",
    "@vitest/coverage-v8": "4.1.10",
    "eslint": "10.8.0",
    "typescript": "7.0.2",
    "typescript-eslint": "8.65.0",
    "vite": "8.1.5",
    "vitest": "4.1.10"
  }
}
```

- [ ] **Step 3: Crear configuración TypeScript estricta**

Create `tsconfig.json`:

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

Create `tsconfig.app.json`:

```json
{
  "compilerOptions": {
    "composite": true,
    "target": "ES2023",
    "useDefineForClassFields": true,
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "types": ["vite/client"],
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"]
}
```

Create `tsconfig.node.json`:

```json
{
  "compilerOptions": {
    "composite": true,
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["vite.config.ts", "vitest.config.ts", "playwright.config.ts", "e2e"]
}
```

- [ ] **Step 4: Instalar exactamente lo declarado**

Run:

```powershell
npm install
npm ls --depth=0
```

Expected: exit code `0`, `package-lock.json` creado y todas las versiones de primer nivel coinciden con `package.json`.

- [ ] **Step 5: Commit**

```powershell
git add .nvmrc package.json package-lock.json tsconfig.json tsconfig.app.json tsconfig.node.json
git commit -m "build: initialize BrainSound toolchain"
```

### Task 2: Crear el shell compilable de React y Vite

**Files:**
- Create: `vite.config.ts`
- Create: `index.html`
- Create: `src/main.tsx`
- Create: `src/app/App.tsx`

- [ ] **Step 1: Crear la configuración de Vite**

Create `vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    target: 'es2023',
    sourcemap: true,
  },
});
```

- [ ] **Step 2: Crear la entrada HTML**

Create `index.html`:

```html
<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="theme-color" content="#17111f" />
    <title>BrainSound</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 3: Crear el bootstrap y componente mínimo**

Create `src/main.tsx`:

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { App } from './app/App';

const root = document.getElementById('root');

if (root === null) {
  throw new Error('No se encontró el elemento #root');
}

createRoot(root).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Create `src/app/App.tsx`:

```tsx
export function App() {
  return (
    <main>
      <h1>BrainSound</h1>
    </main>
  );
}
```

- [ ] **Step 4: Verificar compilación**

Run:

```powershell
npm run typecheck
npm run build
```

Expected: ambos comandos terminan con exit code `0`; Vite crea `dist/index.html` y al menos un asset JavaScript.

- [ ] **Step 5: Commit**

```powershell
git add vite.config.ts index.html src/main.tsx src/app/App.tsx
git commit -m "feat: add BrainSound React shell"
```

### Task 3: Configurar lint, unitarias, cobertura y E2E

**Files:**
- Create: `eslint.config.js`
- Create: `vitest.config.ts`
- Create: `playwright.config.ts`

- [ ] **Step 1: Crear lint estricto para TypeScript**

Create `eslint.config.js`:

```js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['coverage/**', 'dist/**', 'playwright-report/**', 'public/**', 'test-results/**'] },
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    files: ['src/**/*.ts', 'src/**/*.tsx', 'e2e/**/*.ts', 'vite.config.ts', 'vitest.config.ts', 'playwright.config.ts'],
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      '@typescript-eslint/consistent-type-imports': 'error',
      '@typescript-eslint/no-confusing-void-expression': 'off',
    },
  },
);
```

- [ ] **Step 2: Configurar Vitest y umbral de cobertura**

Create `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      include: [
        'src/shared/domain/mode.ts',
        'src/catalog/fallback-catalog.ts',
        'src/audio/create-synth-plan.ts',
        'src/features/player/session-controller.ts',
        'src/features/timer/elapsed-timer.ts',
      ],
      reporter: ['text', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
    },
  },
});
```

- [ ] **Step 3: Configurar Playwright contra build de producción**

Create `playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: false,
  retries: 0,
  use: {
    baseURL: 'http://127.0.0.1:4173',
    trace: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run build && npm run preview -- --port 4173',
    url: 'http://127.0.0.1:4173',
    reuseExistingServer: false,
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
});
```

- [ ] **Step 4: Instalar navegadores y verificar lint/configuración E2E**

Run:

```powershell
npm run lint
npx playwright install chromium webkit
npx playwright --version
```

Expected: lint termina con exit code `0`; Chromium y WebKit quedan disponibles; Playwright imprime `Version 1.61.1`.

- [ ] **Step 5: Commit**

```powershell
git add eslint.config.js vitest.config.ts playwright.config.ts
git commit -m "test: configure BrainSound quality gates"
```

### Task 4: Definir modos y catálogo sintético de respaldo con TDD

**Files:**
- Create: `src/shared/domain/mode.test.ts`
- Create: `src/shared/domain/mode.ts`
- Create: `src/catalog/fallback-catalog.test.ts`
- Create: `src/catalog/fallback-catalog.ts`

- [ ] **Step 1: Escribir pruebas fallidas del dominio**

Create `src/shared/domain/mode.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { getModeDefinition, isBrainSoundMode, modeDefinitions } from './mode';

describe('BrainSound modes', () => {
  it('defines exactly the three approved modes', () => {
    expect(modeDefinitions.map(({ id }) => id)).toEqual([
      'deep-focus',
      'creativity',
      'relaxation',
    ]);
  });

  it('resolves labels and rejects unknown values', () => {
    expect(getModeDefinition('creativity').label).toBe('Creatividad');
    expect(isBrainSoundMode('sleep')).toBe(false);
  });
});
```

Create `src/catalog/fallback-catalog.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { fallbackCatalog, getFallbackExperience } from './fallback-catalog';

describe('fallback catalog', () => {
  it('contains one unique synthesized experience per approved mode', () => {
    expect(fallbackCatalog).toHaveLength(3);
    expect(new Set(fallbackCatalog.map(({ id }) => id)).size).toBe(3);
    expect(new Set(fallbackCatalog.map(({ mode }) => mode)).size).toBe(3);
  });

  it('returns a safe profile with bounded gain', () => {
    const experience = getFallbackExperience('deep-focus');
    expect(experience.profile.masterGain).toBeGreaterThan(0);
    expect(experience.profile.masterGain).toBeLessThanOrEqual(0.12);
  });
});
```

- [ ] **Step 2: Ejecutar las pruebas para verificar que fallan**

Run:

```powershell
npm run test -- src/shared/domain/mode.test.ts src/catalog/fallback-catalog.test.ts
```

Expected: FAIL porque `mode.ts` y `fallback-catalog.ts` aún no existen.

- [ ] **Step 3: Implementar el contrato de modos**

Create `src/shared/domain/mode.ts`:

```ts
export const modeDefinitions = [
  {
    id: 'deep-focus',
    label: 'Deep Focus',
    description: 'Reduce distracciones para trabajo sostenido.',
    gradient: 'var(--gradient-focus)',
  },
  {
    id: 'creativity',
    label: 'Creatividad',
    description: 'Mantiene energía flexible para explorar ideas.',
    gradient: 'var(--gradient-creativity)',
  },
  {
    id: 'relaxation',
    label: 'Relajación',
    description: 'Disminuye estímulos para una pausa consciente.',
    gradient: 'var(--gradient-relaxation)',
  },
] as const;

export type BrainSoundMode = (typeof modeDefinitions)[number]['id'];
export type ModeDefinition = (typeof modeDefinitions)[number];

export function isBrainSoundMode(value: string): value is BrainSoundMode {
  return modeDefinitions.some(({ id }) => id === value);
}

export function getModeDefinition(mode: BrainSoundMode): ModeDefinition {
  const definition = modeDefinitions.find(({ id }) => id === mode);
  if (definition === undefined) {
    throw new Error(`Modo no soportado: ${mode}`);
  }
  return definition;
}
```

- [ ] **Step 4: Implementar el catálogo de degradación segura**

Create `src/catalog/fallback-catalog.ts`:

```ts
import type { BrainSoundMode } from '../shared/domain/mode';

export interface SynthProfile {
  readonly carrierHz: number;
  readonly overtoneHz: number;
  readonly movementHz: number;
  readonly filterHz: number;
  readonly masterGain: number;
}

export interface FallbackExperience {
  readonly id: string;
  readonly mode: BrainSoundMode;
  readonly title: string;
  readonly profile: SynthProfile;
}

export const fallbackCatalog = [
  {
    id: 'fallback-deep-focus',
    mode: 'deep-focus',
    title: 'Pulso profundo',
    profile: { carrierHz: 110, overtoneHz: 220, movementHz: 0.18, filterHz: 900, masterGain: 0.08 },
  },
  {
    id: 'fallback-creativity',
    mode: 'creativity',
    title: 'Impulso creativo',
    profile: { carrierHz: 146.83, overtoneHz: 293.66, movementHz: 0.12, filterHz: 1400, masterGain: 0.07 },
  },
  {
    id: 'fallback-relaxation',
    mode: 'relaxation',
    title: 'Marea serena',
    profile: { carrierHz: 98, overtoneHz: 196, movementHz: 0.07, filterHz: 700, masterGain: 0.06 },
  },
] as const satisfies readonly FallbackExperience[];

export function getFallbackExperience(mode: BrainSoundMode): FallbackExperience {
  const experience = fallbackCatalog.find((candidate) => candidate.mode === mode);
  if (experience === undefined) {
    throw new Error(`No existe experiencia de respaldo para ${mode}`);
  }
  return experience;
}
```

- [ ] **Step 5: Ejecutar pruebas y validación de catálogo**

Run:

```powershell
npm run test -- src/shared/domain/mode.test.ts src/catalog/fallback-catalog.test.ts
npm run validate:catalog
```

Expected: cuatro pruebas PASS y validación de catálogo con exit code `0`.

- [ ] **Step 6: Commit**

```powershell
git add src/shared/domain src/catalog
git commit -m "feat: define BrainSound modes and fallback catalog"
```

### Task 5: Construir el inicio visual y la selección de modo en un clic

**Files:**
- Create: `e2e/home.spec.ts`
- Create: `src/styles/tokens.css`
- Create: `src/styles/global.css`
- Create: `src/features/home/ModeCard.tsx`
- Create: `src/features/home/HomeScreen.tsx`
- Modify: `src/app/App.tsx`
- Modify: `src/main.tsx`

- [ ] **Step 1: Escribir la prueba E2E fallida del inicio**

Create `e2e/home.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('presenta los tres modos principales y permite seleccionarlos', async ({ page }) => {
  await page.goto('/');

  await expect(page.getByRole('heading', { name: 'Elige cómo quieres sentirte' })).toBeVisible();
  await expect(page.getByRole('button', { name: /Deep Focus/ })).toBeVisible();
  await expect(page.getByRole('button', { name: /Creatividad/ })).toBeVisible();
  await expect(page.getByRole('button', { name: /Relajación/ })).toBeVisible();

  await page.getByRole('button', { name: /Creatividad/ }).click();
  await expect(page.getByText('Preparando Impulso creativo')).toBeVisible();
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npx playwright test e2e/home.spec.ts --project=chromium
```

Expected: FAIL porque el encabezado y las tarjetas todavía no existen.

- [ ] **Step 3: Crear los tokens de “Energía cromática”**

Create `src/styles/tokens.css`:

```css
:root {
  color-scheme: dark;
  --color-canvas: #090b18;
  --color-surface: rgba(21, 25, 53, 0.82);
  --color-text: #f7f8ff;
  --color-muted: #afb5d6;
  --color-focus: #62e7ff;
  --color-creativity: #ff79c8;
  --color-relaxation: #8ef0a7;
  --color-border: rgba(255, 255, 255, 0.14);
  --radius-card: 1.5rem;
  --shadow-card: 0 1.5rem 4rem rgba(0, 0, 0, 0.28);
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}
```

Create `src/styles/global.css`:

```css
@import './tokens.css';

* { box-sizing: border-box; }
body {
  margin: 0;
  min-width: 20rem;
  min-height: 100vh;
  color: var(--color-text);
  background:
    radial-gradient(circle at 18% 12%, rgba(98, 231, 255, 0.18), transparent 32rem),
    radial-gradient(circle at 82% 18%, rgba(255, 121, 200, 0.15), transparent 30rem),
    var(--color-canvas);
}
button { font: inherit; }
button:focus-visible { outline: 0.2rem solid var(--color-focus); outline-offset: 0.2rem; }
.app-shell { width: min(72rem, calc(100% - 2rem)); margin: 0 auto; padding: 3rem 0 7rem; }
.eyebrow { color: var(--color-focus); font-weight: 700; letter-spacing: 0.08em; text-transform: uppercase; }
.mode-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 1rem; }
.mode-card {
  min-height: 15rem;
  padding: 1.5rem;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-card);
  color: var(--color-text);
  background: var(--color-surface);
  box-shadow: var(--shadow-card);
  cursor: pointer;
  text-align: left;
  transition: transform 180ms ease, border-color 180ms ease;
}
.mode-card:hover { transform: translateY(-0.25rem); border-color: currentColor; }
.mode-card[data-mode='deep-focus'] { color: var(--color-focus); }
.mode-card[data-mode='creativity'] { color: var(--color-creativity); }
.mode-card[data-mode='relaxation'] { color: var(--color-relaxation); }
.mode-card strong, .mode-card span { display: block; }
.mode-card strong { margin-bottom: 0.75rem; font-size: clamp(1.4rem, 3vw, 2rem); }
.mode-card span { color: var(--color-muted); line-height: 1.5; }
.selection-note { min-height: 1.5rem; color: var(--color-muted); }
@media (max-width: 46rem) { .mode-grid { grid-template-columns: 1fr; } }
@media (prefers-reduced-motion: reduce) { *, *::before, *::after { scroll-behavior: auto !important; transition: none !important; } }
```

- [ ] **Step 4: Crear los componentes del inicio**

Create `src/features/home/ModeCard.tsx`:

```tsx
import type { ModeDefinition } from '../../shared/domain/mode';

interface ModeCardProps {
  readonly mode: ModeDefinition;
  readonly onSelect: (mode: ModeDefinition) => void;
}

export function ModeCard({ mode, onSelect }: ModeCardProps) {
  return (
    <button className="mode-card" data-mode={mode.id} onClick={() => onSelect(mode)} type="button">
      <strong>{mode.label}</strong>
      <span>{mode.description}</span>
    </button>
  );
}
```

Create `src/features/home/HomeScreen.tsx`:

```tsx
import type { ModeDefinition } from '../../shared/domain/mode';
import { modeDefinitions } from '../../shared/domain/mode';
import { ModeCard } from './ModeCard';

interface HomeScreenProps {
  readonly selectionMessage: string;
  readonly onSelect: (mode: ModeDefinition) => void;
}

export function HomeScreen({ selectionMessage, onSelect }: HomeScreenProps) {
  return (
    <main className="app-shell">
      <p className="eyebrow">BrainSound</p>
      <h1>Elige cómo quieres sentirte</h1>
      <p>Audio funcional gratuito, privado y disponible incluso sin conexión.</p>
      <section aria-label="Modos de audio" className="mode-grid">
        {modeDefinitions.map((mode) => <ModeCard key={mode.id} mode={mode} onSelect={onSelect} />)}
      </section>
      <p aria-live="polite" className="selection-note">{selectionMessage}</p>
    </main>
  );
}
```

- [ ] **Step 5: Conectar la selección temporal en la aplicación**

Replace `src/app/App.tsx` with:

```tsx
import { useState } from 'react';
import { getFallbackExperience } from '../catalog/fallback-catalog';
import { HomeScreen } from '../features/home/HomeScreen';
import type { ModeDefinition } from '../shared/domain/mode';

export function App() {
  const [selectionMessage, setSelectionMessage] = useState('');

  function handleSelect(mode: ModeDefinition) {
    setSelectionMessage(`Preparando ${getFallbackExperience(mode.id).title}`);
  }

  return <HomeScreen selectionMessage={selectionMessage} onSelect={handleSelect} />;
}
```

Add the stylesheet import to `src/main.tsx` immediately after the React imports:

```ts
import './styles/global.css';
```

- [ ] **Step 6: Verificar la pantalla en ambos navegadores objetivo**

Run:

```powershell
npx playwright test e2e/home.spec.ts
npm run lint
npm run typecheck
```

Expected: dos ejecuciones E2E PASS, una en Chromium y otra en WebKit; lint y typecheck terminan con exit code `0`.

- [ ] **Step 7: Commit**

```powershell
git add e2e/home.spec.ts src/app/App.tsx src/main.tsx src/features/home src/styles
git commit -m "feat: add chromatic mode selection home"
```

### Task 6: Implementar el motor de síntesis Web Audio detrás de un puerto estable

**Files:**
- Create: `src/audio/audio-port.ts`
- Create: `src/audio/create-synth-plan.test.ts`
- Create: `src/audio/create-synth-plan.ts`
- Create: `src/audio/web-audio-synth.ts`

- [ ] **Step 1: Escribir primero las pruebas del plan de síntesis**

Create `src/audio/create-synth-plan.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { fallbackCatalog } from '../catalog/fallback-catalog';
import { createSynthPlan } from './create-synth-plan';

describe('createSynthPlan', () => {
  it('convierte un perfil del catálogo en un plan acotado', () => {
    const plan = createSynthPlan(fallbackCatalog[0]);

    expect(plan.oscillators).toEqual([
      { frequencyHz: 110, gain: 0.08 },
      { frequencyHz: 220, gain: 0.032 },
    ]);
    expect(plan.movementHz).toBe(0.18);
    expect(plan.filterHz).toBe(900);
  });

  it('rechaza una ganancia insegura', () => {
    const unsafe = {
      ...fallbackCatalog[0],
      profile: { ...fallbackCatalog[0].profile, masterGain: 0.2 },
    };

    expect(() => createSynthPlan(unsafe)).toThrow('Ganancia fuera del rango seguro');
  });
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npm run test -- src/audio/create-synth-plan.test.ts
```

Expected: FAIL porque `create-synth-plan.ts` todavía no existe.

- [ ] **Step 3: Definir el puerto y el plan puro de síntesis**

Create `src/audio/audio-port.ts`:

```ts
import type { FallbackExperience } from '../catalog/fallback-catalog';

export interface AudioPort {
  start(experience: FallbackExperience): Promise<void>;
  pause(): Promise<void>;
  resume(): Promise<void>;
  stop(): Promise<void>;
}
```

Create `src/audio/create-synth-plan.ts`:

```ts
import type { FallbackExperience } from '../catalog/fallback-catalog';

export interface SynthPlan {
  readonly oscillators: readonly { readonly frequencyHz: number; readonly gain: number }[];
  readonly movementHz: number;
  readonly filterHz: number;
}

export function createSynthPlan(experience: FallbackExperience): SynthPlan {
  const { carrierHz, overtoneHz, movementHz, filterHz, masterGain } = experience.profile;
  if (masterGain <= 0 || masterGain > 0.12) {
    throw new Error('Ganancia fuera del rango seguro');
  }

  return {
    oscillators: [
      { frequencyHz: carrierHz, gain: masterGain },
      { frequencyHz: overtoneHz, gain: masterGain * 0.4 },
    ],
    movementHz,
    filterHz,
  };
}
```

- [ ] **Step 4: Implementar el adaptador Web Audio**

Create `src/audio/web-audio-synth.ts`:

```ts
import type { FallbackExperience } from '../catalog/fallback-catalog';
import type { AudioPort } from './audio-port';
import { createSynthPlan } from './create-synth-plan';

export class WebAudioSynth implements AudioPort {
  private context: AudioContext | null = null;
  private sources: OscillatorNode[] = [];

  async start(experience: FallbackExperience): Promise<void> {
    await this.stop();
    const context = this.context ?? new AudioContext();
    this.context = context;
    await context.resume();

    const plan = createSynthPlan(experience);
    const master = context.createGain();
    const filter = context.createBiquadFilter();
    const movement = context.createOscillator();
    const movementDepth = context.createGain();

    master.gain.value = 0.72;
    filter.type = 'lowpass';
    filter.frequency.value = plan.filterHz;
    movement.frequency.value = plan.movementHz;
    movementDepth.gain.value = 0.08;
    movement.connect(movementDepth).connect(master.gain);
    filter.connect(master).connect(context.destination);

    const voices = plan.oscillators.map(({ frequencyHz, gain }) => {
      const oscillator = context.createOscillator();
      const voiceGain = context.createGain();
      oscillator.type = 'sine';
      oscillator.frequency.value = frequencyHz;
      voiceGain.gain.value = gain;
      oscillator.connect(voiceGain).connect(filter);
      oscillator.start();
      return oscillator;
    });

    movement.start();
    this.sources = [...voices, movement];
  }

  async pause(): Promise<void> { await this.context?.suspend(); }
  async resume(): Promise<void> { await this.context?.resume(); }

  async stop(): Promise<void> {
    for (const source of this.sources) {
      try { source.stop(); } catch { /* La fuente ya estaba detenida. */ }
      source.disconnect();
    }
    this.sources = [];
  }
}
```

- [ ] **Step 5: Ejecutar pruebas y comprobaciones estáticas**

Run:

```powershell
npm run test -- src/audio/create-synth-plan.test.ts
npm run lint
npm run typecheck
```

Expected: dos pruebas PASS; lint y typecheck con exit code `0`.

- [ ] **Step 6: Commit**

```powershell
git add src/audio
git commit -m "feat: add safe Web Audio synthesis engine"
```

### Task 7: Crear el controlador de sesión y su adaptador React

**Files:**
- Create: `src/features/player/session-controller.test.ts`
- Create: `src/features/player/session-controller.ts`
- Create: `src/features/player/use-session-controller.ts`

- [ ] **Step 1: Escribir la prueba fallida del ciclo de sesión**

Create `src/features/player/session-controller.test.ts`:

```ts
import { describe, expect, it, vi } from 'vitest';
import type { AudioPort } from '../../audio/audio-port';
import { fallbackCatalog } from '../../catalog/fallback-catalog';
import { SessionController } from './session-controller';

function createAudioStub(): AudioPort {
  return {
    start: vi.fn().mockResolvedValue(undefined),
    pause: vi.fn().mockResolvedValue(undefined),
    resume: vi.fn().mockResolvedValue(undefined),
    stop: vi.fn().mockResolvedValue(undefined),
  };
}

describe('SessionController', () => {
  it('recorre inicio, pausa, reanudación y detención', async () => {
    const audio = createAudioStub();
    const controller = new SessionController(audio);
    const experience = fallbackCatalog[0];

    await controller.start(experience);
    expect(controller.getSnapshot()).toEqual({ status: 'playing', experience });
    await controller.pause();
    expect(controller.getSnapshot().status).toBe('paused');
    await controller.resume();
    expect(controller.getSnapshot().status).toBe('playing');
    await controller.stop();
    expect(controller.getSnapshot()).toEqual({ status: 'idle', experience: null });
  });

  it('notifica cada transición a los suscriptores', async () => {
    const controller = new SessionController(createAudioStub());
    const listener = vi.fn();
    const unsubscribe = controller.subscribe(listener);

    await controller.start(fallbackCatalog[1]);
    await controller.stop();
    unsubscribe();

    expect(listener).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npm run test -- src/features/player/session-controller.test.ts
```

Expected: FAIL porque el controlador todavía no existe.

- [ ] **Step 3: Implementar el controlador sin dependencias de React**

Create `src/features/player/session-controller.ts`:

```ts
import type { AudioPort } from '../../audio/audio-port';
import type { FallbackExperience } from '../../catalog/fallback-catalog';

export type SessionStatus = 'idle' | 'playing' | 'paused';

export interface SessionState {
  readonly status: SessionStatus;
  readonly experience: FallbackExperience | null;
}

type Listener = () => void;

export class SessionController {
  private state: SessionState = { status: 'idle', experience: null };
  private readonly listeners = new Set<Listener>();

  constructor(private readonly audio: AudioPort) {}

  readonly getSnapshot = (): SessionState => this.state;
  readonly subscribe = (listener: Listener): (() => void) => {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  };

  async start(experience: FallbackExperience): Promise<void> {
    await this.audio.start(experience);
    this.setState({ status: 'playing', experience });
  }

  async pause(): Promise<void> {
    if (this.state.status !== 'playing') return;
    await this.audio.pause();
    this.setState({ ...this.state, status: 'paused' });
  }

  async resume(): Promise<void> {
    if (this.state.status !== 'paused') return;
    await this.audio.resume();
    this.setState({ ...this.state, status: 'playing' });
  }

  async stop(): Promise<void> {
    if (this.state.status === 'idle') return;
    await this.audio.stop();
    this.setState({ status: 'idle', experience: null });
  }

  private setState(state: SessionState): void {
    this.state = state;
    this.listeners.forEach((listener) => listener());
  }
}
```

- [ ] **Step 4: Crear el adaptador mínimo para React**

Create `src/features/player/use-session-controller.ts`:

```ts
import { useSyncExternalStore } from 'react';
import type { FallbackExperience } from '../../catalog/fallback-catalog';
import type { SessionController } from './session-controller';

export function useSessionController(controller: SessionController) {
  const state = useSyncExternalStore(controller.subscribe, controller.getSnapshot);
  return {
    state,
    start: (experience: FallbackExperience) => controller.start(experience),
    pause: () => controller.pause(),
    resume: () => controller.resume(),
    stop: () => controller.stop(),
  };
}
```

- [ ] **Step 5: Verificar controlador y tipos**

Run:

```powershell
npm run test -- src/features/player/session-controller.test.ts
npm run typecheck
```

Expected: dos pruebas PASS y typecheck con exit code `0`.

- [ ] **Step 6: Commit**

```powershell
git add src/features/player
git commit -m "feat: add framework-independent session controller"
```

### Task 8: Añadir un reloj monotónico sin deriva acumulativa

**Files:**
- Create: `src/features/timer/elapsed-timer.test.ts`
- Create: `src/features/timer/elapsed-timer.ts`
- Create: `src/features/timer/use-session-clock.ts`
- Modify: `src/features/player/session-controller.ts`
- Modify: `src/features/player/session-controller.test.ts`

- [ ] **Step 1: Escribir las pruebas fallidas del reloj**

Create `src/features/timer/elapsed-timer.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { ElapsedTimer } from './elapsed-timer';

describe('ElapsedTimer', () => {
  it('deriva el tiempo desde un reloj monotónico y conserva las pausas', () => {
    let now = 1_000;
    const timer = new ElapsedTimer({ now: () => now });

    timer.start();
    now = 4_400;
    expect(timer.elapsedSeconds()).toBe(3);
    timer.pause();
    now = 20_000;
    expect(timer.elapsedSeconds()).toBe(3);
    timer.resume();
    now = 22_600;
    expect(timer.elapsedSeconds()).toBe(6);
  });

  it('vuelve a cero al detenerse', () => {
    let now = 0;
    const timer = new ElapsedTimer({ now: () => now });
    timer.start();
    now = 9_000;
    timer.stop();
    expect(timer.elapsedSeconds()).toBe(0);
  });

  it('no acumula deriva después de una hora', () => {
    let now = 0;
    const timer = new ElapsedTimer({ now: () => now });
    timer.start();
    now = 3_600_000;
    expect(timer.elapsedSeconds()).toBe(3_600);
  });
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npm run test -- src/features/timer/elapsed-timer.test.ts
```

Expected: FAIL porque `elapsed-timer.ts` todavía no existe.

- [ ] **Step 3: Implementar el reloj monotónico**

Create `src/features/timer/elapsed-timer.ts`:

```ts
export interface Clock { now(): number; }

export class ElapsedTimer {
  private accumulatedMs = 0;
  private startedAt: number | null = null;

  constructor(private readonly clock: Clock) {}

  start(): void { this.accumulatedMs = 0; this.startedAt = this.clock.now(); }
  pause(): void {
    if (this.startedAt === null) return;
    this.accumulatedMs += this.clock.now() - this.startedAt;
    this.startedAt = null;
  }
  resume(): void { if (this.startedAt === null) this.startedAt = this.clock.now(); }
  stop(): void { this.accumulatedMs = 0; this.startedAt = null; }
  elapsedSeconds(): number {
    const activeMs = this.startedAt === null ? 0 : this.clock.now() - this.startedAt;
    return Math.round((this.accumulatedMs + activeMs) / 1_000);
  }
}
```

- [ ] **Step 4: Integrar el reloj al controlador**

Update `SessionController` so its constructor receives an `ElapsedTimer`, calls `start`, `pause`, `resume` and `stop` immediately after the corresponding successful audio operation, and exposes:

```ts
readonly getElapsedSeconds = (): number => this.timer.elapsedSeconds();
```

The constructor and affected statements must be exactly:

```ts
constructor(
  private readonly audio: AudioPort,
  private readonly timer: ElapsedTimer,
) {}

// Después de await this.audio.start(experience):
this.timer.start();
// Después de await this.audio.pause():
this.timer.pause();
// Después de await this.audio.resume():
this.timer.resume();
// Después de await this.audio.stop():
this.timer.stop();
```

Add this import:

```ts
import type { ElapsedTimer } from '../timer/elapsed-timer';
```

Replace `src/features/player/session-controller.test.ts` with the integration-aware test:

```ts
import { describe, expect, it, vi } from 'vitest';
import type { AudioPort } from '../../audio/audio-port';
import { fallbackCatalog } from '../../catalog/fallback-catalog';
import { ElapsedTimer } from '../timer/elapsed-timer';
import { SessionController } from './session-controller';

function createAudioStub(): AudioPort {
  return {
    start: vi.fn().mockResolvedValue(undefined),
    pause: vi.fn().mockResolvedValue(undefined),
    resume: vi.fn().mockResolvedValue(undefined),
    stop: vi.fn().mockResolvedValue(undefined),
  };
}

describe('SessionController', () => {
  it('recorre inicio, pausa, reanudación y detención', async () => {
    const controller = new SessionController(
      createAudioStub(),
      new ElapsedTimer({ now: () => 0 }),
    );
    const experience = fallbackCatalog[0];

    await controller.start(experience);
    expect(controller.getSnapshot()).toEqual({ status: 'playing', experience });
    await controller.pause();
    expect(controller.getSnapshot().status).toBe('paused');
    await controller.resume();
    expect(controller.getSnapshot().status).toBe('playing');
    await controller.stop();
    expect(controller.getSnapshot()).toEqual({ status: 'idle', experience: null });
  });

  it('notifica cada transición a los suscriptores', async () => {
    const controller = new SessionController(
      createAudioStub(),
      new ElapsedTimer({ now: () => 0 }),
    );
    const listener = vi.fn();
    const unsubscribe = controller.subscribe(listener);
    await controller.start(fallbackCatalog[1]);
    await controller.stop();
    unsubscribe();
    expect(listener).toHaveBeenCalledTimes(2);
  });

  it('coordina el tiempo con las pausas de audio', async () => {
    let now = 0;
    const controller = new SessionController(
      createAudioStub(),
      new ElapsedTimer({ now: () => now }),
    );
    await controller.start(fallbackCatalog[2]);
    now = 1_400;
    expect(controller.getElapsedSeconds()).toBe(1);
    await controller.pause();
    now = 9_000;
    expect(controller.getElapsedSeconds()).toBe(1);
  });
});
```

- [ ] **Step 5: Crear el refresco visual desacoplado del cálculo**

Create `src/features/timer/use-session-clock.ts`:

```ts
import { useEffect, useState } from 'react';
import type { SessionController, SessionStatus } from '../player/session-controller';

export function useSessionClock(controller: SessionController, status: SessionStatus): number {
  const [elapsed, setElapsed] = useState(0);

  useEffect(() => {
    setElapsed(controller.getElapsedSeconds());
    if (status !== 'playing') return undefined;
    const interval = window.setInterval(() => setElapsed(controller.getElapsedSeconds()), 250);
    return () => window.clearInterval(interval);
  }, [controller, status]);

  return elapsed;
}
```

- [ ] **Step 6: Ejecutar toda la unidad afectada**

Run:

```powershell
npm run test -- src/features/timer/elapsed-timer.test.ts src/features/player/session-controller.test.ts
npm run typecheck
```

Expected: seis pruebas PASS y typecheck con exit code `0`.

- [ ] **Step 7: Commit**

```powershell
git add src/features/timer src/features/player/session-controller.ts src/features/player/session-controller.test.ts
git commit -m "feat: add monotonic session timing"
```

### Task 9: Completar la sesión vertical desde inicio hasta detener

**Files:**
- Create: `src/features/player/PlayerPanel.tsx`
- Modify: `src/app/App.tsx`
- Modify: `src/styles/global.css`
- Modify: `e2e/home.spec.ts`
- Create: `e2e/session.spec.ts`

- [ ] **Step 1: Escribir la prueba E2E fallida de la sesión completa**

Create `e2e/session.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('inicia, pausa, reanuda y detiene una sesión', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('button', { name: /Deep Focus/ }).click();

  await expect(page.getByRole('heading', { name: 'Pulso profundo en curso' })).toBeVisible();
  await expect(page.getByTestId('elapsed-time')).toHaveText('00:00');
  await page.getByRole('button', { name: 'Pausar' }).click();
  await expect(page.getByText('Sesión en pausa')).toBeVisible();
  await page.getByRole('button', { name: 'Continuar' }).click();
  await expect(page.getByText('Sesión activa')).toBeVisible();
  await page.getByRole('button', { name: 'Detener' }).click();
  await expect(page.getByRole('heading', { name: 'Pulso profundo en curso' })).not.toBeVisible();
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npx playwright test e2e/session.spec.ts --project=chromium
```

Expected: FAIL porque el reproductor aún no está conectado.

- [ ] **Step 3: Crear el reproductor compacto**

Create `src/features/player/PlayerPanel.tsx`:

```tsx
import type { SessionState } from './session-controller';

interface PlayerPanelProps {
  readonly state: SessionState;
  readonly elapsedSeconds: number;
  readonly onPause: () => Promise<void>;
  readonly onResume: () => Promise<void>;
  readonly onStop: () => Promise<void>;
}

function formatElapsed(totalSeconds: number): string {
  const minutes = Math.floor(totalSeconds / 60).toString().padStart(2, '0');
  const seconds = (totalSeconds % 60).toString().padStart(2, '0');
  return `${minutes}:${seconds}`;
}

export function PlayerPanel({ state, elapsedSeconds, onPause, onResume, onStop }: PlayerPanelProps) {
  if (state.experience === null) return null;
  const paused = state.status === 'paused';

  return (
    <aside aria-label="Reproductor" className="player-panel">
      <div>
        <p>{paused ? 'Sesión en pausa' : 'Sesión activa'}</p>
        <h2>{state.experience.title} en curso</h2>
      </div>
      <output data-testid="elapsed-time">{formatElapsed(elapsedSeconds)}</output>
      <div className="player-actions">
        {paused
          ? <button onClick={() => void onResume()} type="button">Continuar</button>
          : <button onClick={() => void onPause()} type="button">Pausar</button>}
        <button onClick={() => void onStop()} type="button">Detener</button>
      </div>
    </aside>
  );
}
```

- [ ] **Step 4: Conectar audio, controlador, reloj e interfaz**

Replace `src/app/App.tsx` with:

```tsx
import { useEffect, useState } from 'react';
import { WebAudioSynth } from '../audio/web-audio-synth';
import { getFallbackExperience } from '../catalog/fallback-catalog';
import { HomeScreen } from '../features/home/HomeScreen';
import { PlayerPanel } from '../features/player/PlayerPanel';
import { SessionController } from '../features/player/session-controller';
import { useSessionController } from '../features/player/use-session-controller';
import { ElapsedTimer } from '../features/timer/elapsed-timer';
import { useSessionClock } from '../features/timer/use-session-clock';
import type { ModeDefinition } from '../shared/domain/mode';

export function App() {
  const [controller] = useState(() => new SessionController(
    new WebAudioSynth(),
    new ElapsedTimer({ now: () => performance.now() }),
  ));
  const session = useSessionController(controller);
  const elapsedSeconds = useSessionClock(controller, session.state.status);

  useEffect(() => () => { void controller.stop(); }, [controller]);

  function handleSelect(mode: ModeDefinition) {
    void session.start(getFallbackExperience(mode.id));
  }

  return (
    <>
      <HomeScreen selectionMessage="" onSelect={handleSelect} />
      <PlayerPanel
        elapsedSeconds={elapsedSeconds}
        onPause={session.pause}
        onResume={session.resume}
        onStop={session.stop}
        state={session.state}
      />
    </>
  );
}
```

- [ ] **Step 5: Añadir los estilos del reproductor**

Append to `src/styles/global.css`:

```css
.player-panel {
  position: fixed;
  right: 1rem;
  bottom: 1rem;
  left: 1rem;
  display: grid;
  grid-template-columns: 1fr auto auto;
  align-items: center;
  gap: 1rem;
  width: min(62rem, calc(100% - 2rem));
  margin: 0 auto;
  padding: 1rem 1.25rem;
  border: 1px solid var(--color-border);
  border-radius: 1.25rem;
  background: rgba(13, 16, 35, 0.94);
  box-shadow: var(--shadow-card);
  backdrop-filter: blur(1rem);
}
.player-panel p, .player-panel h2 { margin: 0; }
.player-panel p { color: var(--color-muted); }
.player-panel output { font-variant-numeric: tabular-nums; font-weight: 700; }
.player-actions { display: flex; gap: 0.5rem; }
.player-actions button {
  padding: 0.7rem 1rem;
  border: 1px solid var(--color-border);
  border-radius: 999px;
  color: var(--color-text);
  background: transparent;
  cursor: pointer;
}
@media (max-width: 38rem) {
  .player-panel { grid-template-columns: 1fr auto; }
  .player-actions { grid-column: 1 / -1; }
}
```

- [ ] **Step 6: Actualizar la expectativa transitoria del inicio**

In `e2e/home.spec.ts`, replace:

```ts
await expect(page.getByText('Preparando Impulso creativo')).toBeVisible();
```

with:

```ts
await expect(page.getByRole('heading', { name: 'Impulso creativo en curso' })).toBeVisible();
```

- [ ] **Step 7: Verificar el recorrido en Chromium y WebKit**

Run:

```powershell
npx playwright test e2e/home.spec.ts e2e/session.spec.ts
npm run test
npm run lint
npm run typecheck
```

Expected: cuatro ejecuciones E2E PASS, todas las pruebas unitarias PASS y comprobaciones estáticas con exit code `0`.

- [ ] **Step 8: Commit**

```powershell
git add src/app/App.tsx src/features/player/PlayerPanel.tsx src/styles/global.css e2e
git commit -m "feat: deliver one-click functional audio session"
```

### Task 10: Añadir identidad instalable PWA

**Files:**
- Create: `public/manifest.webmanifest`
- Create: `public/icons/brainsound.svg`
- Modify: `index.html`
- Create: `e2e/pwa.spec.ts`

- [ ] **Step 1: Escribir la prueba fallida del manifiesto**

Create `e2e/pwa.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('expone un manifiesto instalable y metadatos de aplicación', async ({ page }) => {
  await page.goto('/');
  const manifestHref = await page.locator('link[rel="manifest"]').getAttribute('href');
  expect(manifestHref).toBe('/manifest.webmanifest');

  const response = await page.request.get('/manifest.webmanifest');
  expect(response.ok()).toBe(true);
  const manifest: unknown = await response.json();
  expect(manifest).toMatchObject({ name: 'BrainSound', display: 'standalone', start_url: '/' });
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npx playwright test e2e/pwa.spec.ts --project=chromium
```

Expected: FAIL porque no hay enlace de manifiesto.

- [ ] **Step 3: Crear manifiesto e icono sin dependencias externas**

Create `public/manifest.webmanifest`:

```json
{
  "name": "BrainSound",
  "short_name": "BrainSound",
  "description": "Audio funcional gratuito para enfoque, creatividad y relajación.",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#090b18",
  "theme_color": "#090b18",
  "icons": [
    { "src": "/icons/brainsound.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any maskable" }
  ]
}
```

Create `public/icons/brainsound.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
  <rect width="512" height="512" rx="112" fill="#090b18"/>
  <path d="M96 280c48-112 96 112 144 0s96 112 176 0" fill="none" stroke="#62e7ff" stroke-linecap="round" stroke-width="42"/>
  <circle cx="360" cy="158" r="54" fill="#ff79c8"/>
</svg>
```

- [ ] **Step 4: Enlazar metadatos en el documento**

Replace the existing `theme-color` value with `#090b18`, then add the remaining metadata inside `<head>` in `index.html`:

```html
<meta name="description" content="Audio funcional gratuito para enfoque, creatividad y relajación." />
<link rel="icon" href="/icons/brainsound.svg" type="image/svg+xml" />
<link rel="manifest" href="/manifest.webmanifest" />
```

- [ ] **Step 5: Verificar manifiesto y compilación**

Run:

```powershell
npx playwright test e2e/pwa.spec.ts
npm run build
```

Expected: dos ejecuciones E2E PASS y `dist/manifest.webmanifest` presente tras el build.

- [ ] **Step 6: Commit**

```powershell
git add public index.html e2e/pwa.spec.ts
git commit -m "feat: add installable BrainSound manifest"
```

### Task 11: Hacer que la vertical funcione sin conexión

**Files:**
- Create: `public/service-worker.js`
- Create: `src/pwa/register-service-worker.ts`
- Modify: `src/main.tsx`
- Create: `e2e/offline.spec.ts`

- [ ] **Step 1: Escribir la prueba fallida offline**

Create `e2e/offline.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('recarga la aplicación y conserva la sesión sintética sin red', async ({ context, page }) => {
  await page.goto('/');
  await expect.poll(() => page.evaluate(() => navigator.serviceWorker.controller !== null)).toBe(true);

  await context.setOffline(true);
  await page.reload();
  await expect(page.getByRole('heading', { name: 'Elige cómo quieres sentirte' })).toBeVisible();
  await page.getByRole('button', { name: /Relajación/ }).click();
  await expect(page.getByRole('heading', { name: 'Marea serena en curso' })).toBeVisible();
});
```

- [ ] **Step 2: Ejecutar la prueba para comprobar que falla**

Run:

```powershell
npx playwright test e2e/offline.spec.ts --project=chromium
```

Expected: FAIL esperando un Service Worker controlador.

- [ ] **Step 3: Implementar caché local de la aplicación**

Create `public/service-worker.js`:

```js
const CACHE_NAME = 'brainsound-i1-v1';
const APP_SHELL = ['/', '/index.html', '/manifest.webmanifest', '/icons/brainsound.svg'];

async function cacheApplication() {
  const cache = await caches.open(CACHE_NAME);
  await cache.addAll(APP_SHELL);
  const response = await fetch('/');
  const html = await response.text();
  const assetPaths = [...html.matchAll(/(?:src|href)="(\/assets\/[^\"]+)"/g)]
    .map((match) => match[1]);
  await cache.addAll(assetPaths);
}

self.addEventListener('install', (event) => {
  event.waitUntil(cacheApplication());
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(keys.filter((key) => key !== CACHE_NAME).map((key) => caches.delete(key))))
      .then(() => self.clients.claim()),
  );
});

self.addEventListener('fetch', (event) => {
  const requestUrl = new URL(event.request.url);
  if (event.request.method !== 'GET' || requestUrl.origin !== self.location.origin) return;

  event.respondWith(
    caches.match(event.request).then((cached) => cached ?? fetch(event.request).then((response) => {
      if (response.ok) {
        const copy = response.clone();
        void caches.open(CACHE_NAME).then((cache) => cache.put(event.request, copy));
      }
      return response;
    })),
  );
});
```

- [ ] **Step 4: Registrar el Service Worker solo en producción**

Create `src/pwa/register-service-worker.ts`:

```ts
export function registerServiceWorker(): void {
  if (!('serviceWorker' in navigator) || !import.meta.env.PROD) return;
  window.addEventListener('load', () => {
    void navigator.serviceWorker.register('/service-worker.js');
  });
}
```

Add to `src/main.tsx`:

```ts
import { registerServiceWorker } from './pwa/register-service-worker';

registerServiceWorker();
```

- [ ] **Step 5: Ejecutar el E2E contra el build de producción ya configurado en Task 3**

Run:

```powershell
npx playwright test e2e/offline.spec.ts
```

Expected: PASS en Chromium y WebKit tras instalar el Service Worker y desconectar el contexto.

- [ ] **Step 6: Commit**

```powershell
git add public/service-worker.js src/pwa src/main.tsx e2e/offline.spec.ts
git commit -m "feat: make the foundation session work offline"
```

### Task 12: Cerrar accesibilidad, movimiento reducido y privacidad de red

**Files:**
- Create: `e2e/quality.spec.ts`
- Modify: `src/features/home/HomeScreen.tsx`
- Modify: `src/styles/global.css`

- [ ] **Step 1: Escribir las pruebas de calidad visibles**

Create `e2e/quality.spec.ts`:

```ts
import { expect, test } from '@playwright/test';

test('permite iniciar un modo usando solo teclado', async ({ page }) => {
  await page.goto('/');
  await page.keyboard.press('Tab');
  await expect(page.getByRole('button', { name: /Deep Focus/ })).toBeFocused();
  await page.keyboard.press('Enter');
  await expect(page.getByRole('heading', { name: 'Pulso profundo en curso' })).toBeVisible();
});

test('respeta la preferencia de movimiento reducido', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  const duration = await page.getByRole('button', { name: /Deep Focus/ })
    .evaluate((element) => getComputedStyle(element).transitionDuration);
  expect(duration).toBe('0s');
});

test('no solicita recursos de terceros', async ({ page }) => {
  const externalOrigins = new Set<string>();
  page.on('request', (request) => {
    const url = new URL(request.url());
    if (url.origin !== 'http://127.0.0.1:4173') externalOrigins.add(url.origin);
  });

  await page.goto('/');
  await page.getByRole('button', { name: /Creatividad/ }).click();
  expect([...externalOrigins]).toEqual([]);
});
```

- [ ] **Step 2: Ejecutar las pruebas para registrar el estado inicial**

Run:

```powershell
npx playwright test e2e/quality.spec.ts --project=chromium
```

Expected: la prueba de teclado puede pasar; la de movimiento reducido debe validar `0s`; cualquier solicitud externa hace fallar la tercera. No continuar hasta que el resultado sea reproducible.

- [ ] **Step 3: Añadir el posicionamiento responsable visible**

In `HomeScreen`, add after the introductory paragraph:

```tsx
<p className="responsible-note">
  Diseñado para acompañar tu rutina; no es un tratamiento médico ni sustituye orientación profesional.
</p>
```

Append to `src/styles/global.css`:

```css
.responsible-note { max-width: 46rem; margin-bottom: 2rem; color: var(--color-muted); font-size: 0.9rem; }
```

- [ ] **Step 4: Ejecutar toda la matriz E2E**

Run:

```powershell
npx playwright test
```

Expected: todas las pruebas PASS en Chromium y WebKit, sin solicitudes externas.

- [ ] **Step 5: Commit**

```powershell
git add e2e/quality.spec.ts src/features/home/HomeScreen.tsx src/styles/global.css
git commit -m "test: enforce accessible private foundation experience"
```

### Task 13: Documentar, medir y verificar la Iteración 1

**Files:**
- Create: `README.md`
- Modify: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Modify: `docs/plans/2026-08-07-brainsound-i1-foundation.md`

- [ ] **Step 1: Documentar el alcance real y los comandos**

Create `README.md` with this content:

```md
# BrainSound

PWA gratuita y local de audio funcional para Deep Focus, Creatividad y Relajación.

## Estado

La Iteración 1 entrega una vertical instalable y offline con una experiencia sintética segura por modo, inicio en un clic, pausa, reanudación, detención y reloj monotónico. El catálogo completo, audio híbrido, personalización, persistencia y respaldo pertenecen a las siguientes iteraciones del roadmap SDD.

## Requisitos

- Node.js 24 LTS
- npm incluido con Node.js

## Uso local

```powershell
npm ci
npm run dev
```

Abrir `http://127.0.0.1:5173`.

## Verificación

```powershell
npm run verify
npm run test:e2e
```

## Privacidad y alcance responsable

BrainSound no requiere cuentas, backend, telemetría ni servicios pagos. La experiencia de esta iteración se genera en el dispositivo. Es una herramienta de bienestar y productividad, no un tratamiento médico.

## Especificación y planes

- Diseño aprobado: `docs/specs/2026-08-07-brainsound-mvp-design.md`
- Roadmap del MVP: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Plan de Iteración 1: `docs/plans/2026-08-07-brainsound-i1-foundation.md`
```

- [ ] **Step 2: Comprobar instalación reproducible desde el archivo de bloqueo**

Run:

```powershell
npm ci
```

Expected: dependencias recreadas exactamente desde `package-lock.json` con exit code `0`.

- [ ] **Step 3: Ejecutar la verificación reproducible completa**

Run:

```powershell
npm run verify
npm run test:e2e
npm run build
```

Expected:

- lint con exit code `0`;
- typecheck con exit code `0`;
- pruebas unitarias con cobertura global mínima de 80%;
- todas las pruebas E2E PASS en Chromium y WebKit;
- build de producción exitoso en `dist/`.

- [ ] **Step 4: Medir los criterios propios de I1**

Run:

```powershell
npx playwright test e2e/session.spec.ts e2e/offline.spec.ts e2e/quality.spec.ts
Get-ChildItem -LiteralPath dist -Recurse | Measure-Object -Property Length -Sum
```

Expected: sesiones y offline PASS en ambos navegadores. El tamaño total de `dist/` debe ser muy inferior al límite del MVP de 300 MB; registrar el valor exacto en la sección `Resultados de ejecución` de este plan.

- [ ] **Step 5: Revisar el árbol de trabajo antes del cierre**

Run:

```powershell
git status --short
git diff --check
```

Expected: ningún error de espacios; solo archivos intencionales de documentación pendientes. No agregar `AGENTS.md` ni archivos ajenos a BrainSound.

- [ ] **Step 6: Actualizar el estado SDD con evidencia**

In the roadmap, change I1 from `Planificada` to `Completada` only if every command above passes. Collect the execution identity with:

```powershell
Get-Date -Format 'yyyy-MM-dd HH:mm:ss K'
git rev-parse HEAD
```

Append a `## Resultados de ejecución` section to this plan with six bullets: fecha, commit verificado, resultado unitario/cobertura, resultado Chromium/WebKit, tamaño exacto de `dist/` and desviaciones aprobadas. Populate every bullet exclusively with the observed outputs; never invent values. If any gate fails, keep I1 as `En ejecución` and record the blocker instead of declaring completion.

- [ ] **Step 7: Commit documental de cierre**

```powershell
git add README.md docs/plans/2026-08-07-brainsound-mvp-roadmap.md docs/plans/2026-08-07-brainsound-i1-foundation.md
git commit -m "docs: close BrainSound foundation iteration"
```

## Gate de aprobación antes de ejecutar

Este documento fue redactado antes de formalizar la SPEC particular de I1 y queda suspendido. No iniciar código hasta que el usuario apruebe explícitamente:

1. `docs/specs/2026-08-07-brainsound-i1-foundation-spec.md`;
2. la revisión de este plan contra esa SPEC;
3. la descomposición del MVP en cinco iteraciones del roadmap;
4. que el catálogo de 15 experiencias, intensidades y audio híbrido se incorporen en I2 e I3 sin ampliar I1.

Cualquier cambio funcional o arquitectónico posterior actualiza primero la SPEC o este plan, registra la decisión y vuelve a pasar por aprobación SDD.
