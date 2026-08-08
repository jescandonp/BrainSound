# BrainSound I5 Offline Transaction and MVP Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Completar el MVP con instalación/actualización transaccional del catálogo, rollback, respaldo JSON atómico, offline integral, accesibilidad y evidencia multiplataforma, dejando un build estático sin desplegar.

**Architecture:** `InstallController` ejecuta una máquina de estados pura sobre puertos de cuota, descarga, integridad y caché; un puntero de versión activa impide reemplazos parciales. Exportación/importación usa un esquema versionado y staging IndexedDB. Las auditorías de red, accesibilidad y navegador producen evidencia local y no telemetría.

**Tech Stack:** TypeScript, React, Cache Storage, IndexedDB/idb, StorageManager, Web Crypto SHA-256, Service Worker, Vitest, Playwright y herramientas de accesibilidad aprobadas como dependencia de desarrollo.

---

**SPEC:** `docs/specs/2026-08-07-brainsound-i5-offline-release-spec.md` — aprobada el 2026-08-07.

**Estado:** aprobado por el usuario el 2026-08-07; ejecución condicionada al cierre satisfactorio de I4.

**Dependencia de ejecución:** I1–I4 completadas. Rama/worktree `feat/i5-offline-release`.

## Mapa de archivos

```text
src/pwa/install/install-state.ts             Máquina pura
src/pwa/install/quota-port.ts                Estimación
src/pwa/install/integrity.ts                 SHA-256
src/pwa/install/download-port.ts             Descarga/reanudación
src/pwa/install/catalog-cache.ts             Staging/activeVersion
src/pwa/install/install-controller.ts        Orquestación
src/storage/backup/backup-schema.ts          unknown → V1
src/storage/backup/export-backup.ts          JSON UTF-8
src/storage/backup/import-backup.ts          Staging/commit
src/features/settings/installation/**        UI instalación
src/features/settings/backup/**              UI respaldo
scripts/audit-network.ts                     Red permitida
scripts/audit-accessibility.ts               Críticos
e2e/offline-mvp.spec.ts                      15 experiencias offline
e2e/backup.spec.ts                           Round trip/atomicidad
docs/qa/browser-matrix.md                    Edge/Chrome/Safari
docs/user/**                                 Guías finales
reports/mvp/**                               Trece criterios
```

### Task 1: Implementar máquina de estados de instalación con TDD

**Files:**
- Create: `src/pwa/install/install-state.test.ts`
- Create: `src/pwa/install/install-state.ts`

- [ ] **Step 1: RED de transiciones válidas e inválidas**

Assert `not-installed→checking→downloading→verifying→ready-to-activate→installed`; reject direct downloading→installed; failed preserves activeVersion; progress cannot decrease or exceed total.

- [ ] **Step 2: Implementar unión y reducer**

```ts
export type InstallState = {readonly kind:'not-installed'} | {readonly kind:'checking';readonly targetVersion:string} | {readonly kind:'downloading';readonly targetVersion:string;readonly completedBytes:number;readonly totalBytes:number} | {readonly kind:'verifying';readonly targetVersion:string;readonly completedAssets:number;readonly totalAssets:number} | {readonly kind:'ready-to-activate';readonly targetVersion:string} | {readonly kind:'installed';readonly activeVersion:string} | {readonly kind:'failed';readonly activeVersion:string|null;readonly recoverable:boolean;readonly reason:string};
export type InstallEvent = {readonly type:'CHECK';readonly targetVersion:string}|{readonly type:'DOWNLOAD';readonly completedBytes:number;readonly totalBytes:number}|{readonly type:'VERIFY';readonly completedAssets:number;readonly totalAssets:number}|{readonly type:'READY'}|{readonly type:'ACTIVATE'}|{readonly type:'FAIL';readonly recoverable:boolean;readonly reason:string};
export function reduceInstall(state:InstallState,event:InstallEvent):InstallState;
```

Implement exhaustive `switch` and throw ``new Error(`Transición inválida: ${state.kind}/${event.type}`)``; validate progress in each case.

- [ ] **Step 3: GREEN y commit**

```powershell
npm run test -- src/pwa/install/install-state.test.ts
git add src/pwa/install/install-state.ts src/pwa/install/install-state.test.ts
git commit -m "feat: define transactional install state machine"
```

### Task 2: Prevalidar cuota y verificar SHA-256

**Files:**
- Create: `src/pwa/install/quota-port.test.ts`
- Create: `src/pwa/install/quota-port.ts`
- Create: `src/pwa/install/integrity.test.ts`
- Create: `src/pwa/install/integrity.ts`

- [ ] **Step 1: RED de cuota conocida/desconocida**

Test available `quota-usage` greater/equal required passes; less fails with exact bytes; absent `navigator.storage.estimate` returns `unknown` requiring confirmation, not success.

- [ ] **Step 2: Implementar puerto**

```ts
export type QuotaCheck = {readonly kind:'enough';readonly availableBytes:number}|{readonly kind:'insufficient';readonly availableBytes:number;readonly requiredBytes:number}|{readonly kind:'unknown';readonly requiredBytes:number};
export interface QuotaPort { check(requiredBytes:number):Promise<QuotaCheck>; }
```

Browser implementation calls `navigator.storage.estimate()` and never treats undefined values as zero.

- [ ] **Step 3: RED/GREEN de integridad**

Test bytes `[1,2,3]` against known SHA-256 and wrong size/hash. Implement:

```ts
export async function verifyAsset(bytes:ArrayBuffer, expectedBytes:number, expectedSha256:string):Promise<readonly string[]> { const issues:string[]=[]; if(bytes.byteLength!==expectedBytes)issues.push('bytes'); const digest=await crypto.subtle.digest('SHA-256',bytes); const hex=[...new Uint8Array(digest)].map((value)=>value.toString(16).padStart(2,'0')).join(''); if(hex!==expectedSha256)issues.push('sha256'); return issues; }
```

- [ ] **Step 4: Commit**

```powershell
npm run test -- src/pwa/install/quota-port.test.ts src/pwa/install/integrity.test.ts
git add src/pwa/install/quota-port* src/pwa/install/integrity*
git commit -m "feat: preflight quota and catalog integrity"
```

### Task 3: Crear staging de caché con puntero activo

**Files:**
- Create: `src/pwa/install/catalog-cache.test.ts`
- Create: `src/pwa/install/catalog-cache.ts`

- [ ] **Step 1: RED de aislamiento y rollback**

Using injected CacheStorage/metadata port, assert writes target ``brainsound-catalog-staging-${version}``; `activeVersion` unchanged until activate; cancel deletes only target staging; failed read test restores prior pointer; successful read deletes old only afterward.

- [ ] **Step 2: Implementar API exacta**

```ts
export interface CatalogCachePort { begin(version:string):Promise<void>; has(version:string,path:string):Promise<boolean>; put(version:string,path:string,response:Response):Promise<void>; activate(version:string,readProbe:()=>Promise<void>):Promise<void>; cancel(version:string):Promise<void>; activeVersion():Promise<string|null>; }
```

Store active pointer in IndexedDB `meta` key `activeCatalogVersion`; never infer active cache from cache names.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/pwa/install/catalog-cache.test.ts
git add src/pwa/install/catalog-cache.ts src/pwa/install/catalog-cache.test.ts
git commit -m "feat: stage and atomically activate catalogs"
```

### Task 4: Descargar con reanudación por activo verificado

**Files:**
- Create: `src/pwa/install/download-port.test.ts`
- Create: `src/pwa/install/download-port.ts`

- [ ] **Step 1: RED de reanudación**

Test manifest paths `[a,b,c]`, staging already has verified `a`, network fails on `c`; first run downloads only b/c and leaves a/b; second run downloads only c. External origin is rejected before fetch.

- [ ] **Step 2: Implementar origen fijo y progreso**

```ts
export interface DownloadProgress { readonly completedBytes:number; readonly totalBytes:number; readonly completedAssets:number; readonly totalAssets:number; }
export interface DownloadPort { download(version:string,assets:readonly CatalogAsset[],onProgress:(progress:DownloadProgress)=>void,signal:AbortSignal):Promise<void>; }
```

Resolve each path via `new URL('/catalog/assets/'+asset.path, configuredOrigin)` and assert `url.origin===configuredOrigin`. Put response only after `verifyAsset` returns no issues.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/pwa/install/download-port.test.ts
git add src/pwa/install/download-port.ts src/pwa/install/download-port.test.ts
git commit -m "feat: resume verified catalog downloads"
```

### Task 5: Orquestar instalación y activación entre sesiones

**Files:**
- Create: `src/pwa/install/install-controller.test.ts`
- Create: `src/pwa/install/install-controller.ts`

- [ ] **Step 1: RED de flujo, cancelación y sesión activa**

Fake all ports. Assert insufficient quota calls no download; unknown quota needs `confirmUnknownQuota`; active player stops at ready-to-activate and exposes pending; failed activation returns failed with previous active version.

- [ ] **Step 2: Implementar controlador observable**

```ts
export class InstallController { readonly subscribe:(listener:()=>void)=>()=>void; readonly getSnapshot:()=>InstallState; checkAndInstall(index:CatalogIndex,assets:readonly CatalogAsset[],options:{readonly confirmUnknownQuota:boolean}):Promise<void>; activateWhenIdle(isSessionActive:()=>boolean,readProbe:()=>Promise<void>):Promise<void>; cancel():Promise<void>; }
```

Every awaited side effect advances reducer only after success. `cancel` aborts current fetch and deletes only target staging.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/pwa/install/install-controller.test.ts
npm run typecheck
git add src/pwa/install/install-controller.ts src/pwa/install/install-controller.test.ts
git commit -m "feat: orchestrate recoverable catalog installation"
```

### Task 6: Construir UI de instalación y recuperación

**Files:**
- Create: `src/features/settings/installation/InstallationPanel.tsx`
- Modify: `src/features/settings/SettingsScreen.tsx`
- Create: `e2e/install-recovery.spec.ts`

- [ ] **Step 1: E2E RED**

Route assets with controlled sizes/failure. Assert size/version before confirmation, byte progress, interruption, `Reanudar`, prior version still active, successful activation after stopping session, and cancel text.

- [ ] **Step 2: Implementar panel por unión discriminada**

Render every `InstallState.kind` exhaustively with text and native `<progress>`. Unknown quota requires checkbox confirmation. Buttons call controller methods; no fetch from component.

- [ ] **Step 3: Verificar y commit**

```powershell
npx playwright test e2e/install-recovery.spec.ts
npm run typecheck
git add src/features/settings e2e/install-recovery.spec.ts
git commit -m "feat: present recoverable catalog installation"
```

### Task 7: Validar y exportar Backup V1

**Files:**
- Create: `src/storage/backup/backup-schema.test.ts`
- Create: `src/storage/backup/backup-schema.ts`
- Create: `src/storage/backup/export-backup.test.ts`
- Create: `src/storage/backup/export-backup.ts`

- [ ] **Step 1: RED de formato/límites**

Assert wrong format/version/date, >100000 sessions, invalid rating/ID and >10MB input report all issues. Valid V1 returns typed value.

- [ ] **Step 2: Implementar parser unknown**

Export `parseBackup(input:unknown): ValidationResult<BrainSoundBackupV1>` and `parseBackupText(text:string)`; check UTF-8 string byte length with `TextEncoder`; reuse preference/session guards, no casts before validation.

- [ ] **Step 3: Export deterministic JSON**

```ts
export async function exportBackup(repositories:BackupRepositories,now:string,appVersion:string):Promise<string> { const backup:BrainSoundBackupV1={format:'brainsound-backup',version:1,exportedAt:now,appVersion,data:{preferences:await repositories.preferences.get(),favoriteExperienceIds:(await repositories.favorites.list()).map(({experienceId})=>experienceId).sort(),sessions:[...(await repositories.sessions.list())].sort((a,b)=>a.id.localeCompare(b.id))}}; return JSON.stringify(backup,null,2)+'\n'; }
```

- [ ] **Step 4: Verificar y commit**

```powershell
npm run test -- src/storage/backup
git add src/storage/backup
git commit -m "feat: validate and export BrainSound backups"
```

### Task 8: Importar mediante staging y resolver duplicados

**Files:**
- Create: `src/storage/backup/import-backup.test.ts`
- Create: `src/storage/backup/import-backup.ts`

- [ ] **Step 1: RED de atomicidad**

Test invalid backup writes zero; same session ID/equal record is idempotent; same ID/different record cancels all; simulated transaction abort preserves preferences/favorites/sessions.

- [ ] **Step 2: Implementar una transacción única**

```ts
export interface ImportResult { readonly importedSessions:number; readonly importedFavorites:number; readonly warnings:readonly string[]; }
export async function importBackup(db:BrainSoundDatabase,input:unknown):Promise<ImportResult>;
```

Parse before opening transaction. Use one readwrite transaction over preferences/favorites/sessions; pre-read conflicts, abort on any difference, then write all and await `tx.done`.

- [ ] **Step 3: Verificar y commit**

```powershell
npm run test -- src/storage/backup/import-backup.test.ts
git add src/storage/backup/import-backup.ts src/storage/backup/import-backup.test.ts
git commit -m "feat: import backups atomically"
```

### Task 9: Crear UI de respaldo y E2E round trip

**Files:**
- Create: `src/features/settings/backup/BackupPanel.tsx`
- Modify: `src/features/settings/SettingsScreen.tsx`
- Create: `e2e/backup.spec.ts`

- [ ] **Step 1: E2E RED**

Seed preferences/favorite/session; export download; clear DB through test fixture; import file; assert exact restored values. Import malformed/conflicting backup and assert original state unchanged. Confirm current-backup offer appears before import.

- [ ] **Step 2: Implementar UI local**

Use Blob/ObjectURL for export and revoke after click. File input accepts `.json`, reads text, parses before confirmation, lists every validation issue. Never logs backup content.

- [ ] **Step 3: Verificar y commit**

```powershell
npx playwright test e2e/backup.spec.ts
npm run test -- src/storage/backup
git add src/features/settings e2e/backup.spec.ts
git commit -m "feat: export and restore local BrainSound data"
```

### Task 10: Cerrar offline completo y auditoría de red

**Files:**
- Modify: `public/service-worker.js`
- Create: `e2e/offline-mvp.spec.ts`
- Create: `scripts/audit-network.ts`
- Modify: `package.json`

- [ ] **Step 1: E2E RED de 15 experiencias offline**

Install catalog online, set context offline, reload, visit five areas, start each of 15 IDs, switch intensity, favorite, record session and export backup. Assert no external request and no failed same-origin asset request.

- [ ] **Step 2: Actualizar Service Worker**

Keep app shell cache separate from catalog version caches. Fetch handler resolves `/catalog/assets/*` through activeVersion cache, never network when installed/offline; update app shell with install/activate semantics preserving prior cache if new install fails.

- [ ] **Step 3: Auditoría automatizada**

Create `audit-network.ts` that runs Playwright Chromium against production preview, collects requests, fails if origin differs from configured local origin during offline journey, and prints sorted violations. Add script `audit:network`.

- [ ] **Step 4: Verify/commit**

```powershell
npx playwright test e2e/offline-mvp.spec.ts
npm run audit:network
git add public/service-worker.js e2e/offline-mvp.spec.ts scripts/audit-network.ts package.json package-lock.json
git commit -m "feat: complete installed offline experience"
```

### Task 11: Auditar accesibilidad y matriz real de navegadores

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`
- Create: `scripts/audit-accessibility.ts`
- Create: `docs/qa/browser-matrix.md`

- [ ] **Step 1: Agregar auditor de desarrollo aprobado**

Install the verified development-only package:

```powershell
npm install --save-dev @axe-core/playwright@4.12.1
```

Expected: package lock pins `4.12.1` and license review records MPL-2.0; no production dependency is added.

- [ ] **Step 2: Implementar comando**

Create `scripts/audit-accessibility.ts` using `AxeBuilder({ page }).analyze()` for routes `/`, `/?view=explore`, `/?view=favorites`, `/?view=progress`, `/?view=settings`; filter `impact === 'critical'`, print route/rule/targets, set `process.exitCode=1` when any exist. Add `"audit:accessibility": "node scripts/audit-accessibility.ts"`.

- [ ] **Step 3: Crear matriz sin resultados inventados**

Create rows for Windows Edge/Chrome and macOS Chrome/Safari; columns install, offline, audio, timers, backup, keyboard, reduced motion, result, version, tester, date. Populate only from actual device runs; any blank/FAIL blocks I5.

- [ ] **Step 4: Verify/commit**

```powershell
npm run audit:accessibility
npm run test:e2e
git add package.json package-lock.json scripts/audit-accessibility.ts docs/qa/browser-matrix.md
git commit -m "test: audit accessibility and supported browsers"
```

### Task 12: Documentar, evidenciar y preparar build sin desplegar

**Files:**
- Create: `docs/user/installation.md`
- Create: `docs/user/privacy-and-backup.md`
- Create: `docs/user/recovery.md`
- Create: `reports/mvp/success-criteria.md`
- Modify: `README.md`
- Modify: `docs/plans/2026-08-07-brainsound-mvp-roadmap.md`
- Modify: `docs/plans/2026-08-07-brainsound-i5-offline-release.md`

- [ ] **Step 1: Escribir guías verificables**

Document install/uninstall, 300MB target, offline limits, local data, export/import, interruption/resume, quota/hash failure and rollback. No hosting URL because deployment is not authorized.

- [ ] **Step 2: Ejecutar todas las compuertas**

```powershell
npm ci
npm run validate:catalog
npm run audit:network
npm run audit:accessibility
npm run test:coverage
npm run verify
npm run test:e2e
npm run build
git diff --check
```

Expected: every command `0`, critical modules ≥90%, global ≥80%, 15 experiences offline, build in `dist/` and no deployment.

- [ ] **Step 3: Crear matriz de trece criterios**

In `reports/mvp/success-criteria.md`, copy each criterion from master SPEC and link exact automated output/manual row/size/hash. State PASS only with evidence; blank evidence is FAIL.

- [ ] **Step 4: Cerrar estados con evidencia**

Append exact commit, versions, tests, coverage, bytes, backup roundtrip, network/accessibility and browser matrix results. Set I5/MVP completed only if all thirteen criteria PASS.

- [ ] **Step 5: Commit documental final**

```powershell
git add README.md docs/user docs/qa/browser-matrix.md reports/mvp docs/plans
git commit -m "docs: prepare verified BrainSound MVP release"
```

Do not run any deploy, publish, domain or hosting command.

## Trazabilidad SPEC → tareas

| Criterio I5 | Tareas |
|---|---|
| Estado/cuota/integridad | 1, 2 |
| Staging/reanudación/rollback | 3, 4, 5 |
| UI de instalación | 6 |
| Backup validado/atómico | 7, 8, 9 |
| 15 experiencias offline/red | 10 |
| Accesibilidad/navegadores | 11 |
| Trece criterios/build sin deploy | 12 |

## Gate de aprobación antes de ejecutar

Gate de plan superado el 2026-08-07. I5 permanece bloqueada hasta cerrar I1–I4. La preparación del build no autoriza despliegue; cualquier publicación exige una solicitud posterior separada.
