# BrainSound I2 — SPEC de catálogo, licencias y canal de activos

- **SPEC maestra:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Iteración:** I2 de 5
- **Estado:** aprobada explícitamente por el usuario el 2026-08-07; planificación autorizada
- **Dependencia:** I1 completada y sus contratos de modo disponibles

## 1. Objetivo

Definir y validar el catálogo versionado de BrainSound con quince experiencias —cinco para Deep Focus, cinco para Creatividad y cinco para Relajación— y un canal reproducible que impida aceptar activos sin licencia, procedencia, atribución, integridad o tamaño comprobados.

I2 entrega navegación y búsqueda del catálogo basada en metadatos. No reproduce todavía la mezcla híbrida normal; la audición continúa usando la degradación sintética de I1 hasta I3.

## 2. Supuestos explícitos

- Solo se admiten activos que permitan redistribución y modificación en el contexto del proyecto.
- Ningún audio de Brain.fm se descarga, analiza, transforma o incorpora.
- La selección exacta de archivos es una tarea de ejecución: cada alta requiere evidencia verificable y no puede aprobarse por semejanza o memoria.
- El catálogo completo no puede superar 300 MB; I2 reservará presupuesto por activo y rechazará excesos.
- Los quince IDs son estables: `deep-focus-01` a `deep-focus-05`, `creativity-01` a `creativity-05` y `relaxation-01` a `relaxation-05`.
- Los títulos públicos pueden cambiar antes de aprobar un manifiesto, pero el ID no cambia.

## 3. Alcance funcional

### Incluido

1. Contrato de catálogo y esquema versionado.
2. Quince manifiestos de experiencia, cada uno con tres intensidades soportadas.
3. Registro de activos con URL de origen, autor, licencia, atribución, fecha de verificación, tamaño y SHA-256.
4. Validador ejecutable que falla ante metadatos incompletos, duplicados, rutas inseguras, hashes inválidos o presupuesto excedido.
5. Reporte determinista de licencias y presupuesto.
6. Lectura local del catálogo desde la aplicación.
7. Pantalla Explorar con filtro por modo, búsqueda por título/tags y resultado vacío accesible.
8. Pruebas con fixtures propios o sintéticos; ningún fixture propietario.

### Excluido

- Decodificación, mezcla o reproducción de capas reales.
- Descarga reanudable, activación atómica y rollback.
- Favoritos, historial, recomendación, valoraciones y progreso.
- Edición del catálogo desde la interfaz.
- Fuentes o APIs de audio que requieran claves, pagos o conexión en tiempo de uso.

## 4. Contrato de catálogo

```ts
export type BrainSoundMode = 'deep-focus' | 'creativity' | 'relaxation';
export type Intensity = 'soft' | 'medium' | 'deep';
export type LayerKind = 'music' | 'ambient' | 'nature';

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

export interface CatalogExperience {
  readonly schemaVersion: 1;
  readonly id: string;
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

export interface CatalogIndex {
  readonly schemaVersion: 1;
  readonly catalogVersion: string;
  readonly generatedAt: string;
  readonly experiences: readonly string[];
}
```

Reglas obligatorias:

- `catalogVersion` usa fecha y revisión: `YYYY.MM.DD.N`.
- Todos los textos visibles están en español; IDs, tags y rutas usan kebab-case ASCII.
- Cada experiencia declara al menos una capa y exactamente un perfil para `soft`, `medium` y `deep`.
- `intensities` contiene exactamente los mismos tres valores declarados por `intensityProfiles`, sin otro valor ni duplicados.
- Cada referencia `assetId` de un perfil existe dentro de `assets` de la misma experiencia.
- `masterGain` está entre `0` y `0.9`, `gain` entre `0` y `1`, `playbackRate` entre `0.95` y `1.05`, y `crossfadeSeconds` entre `8` y `12`.
- `sha256` contiene 64 caracteres hexadecimales en minúscula.
- `verifiedOn` usa `YYYY-MM-DD` y no puede ser futuro respecto de la ejecución del validador.
- `sourceUrl`, `licenseUrl` y `attribution` no pueden estar vacíos.
- Rutas son relativas a `public/catalog/`, sin `..`, letras de unidad ni URL remota.
- La suma declarada y real de activos únicos debe ser como máximo 314.572.800 bytes.
- Un activo compartido se cuenta una sola vez en el presupuesto.

## 5. Flujo de incorporación

```text
fuente candidata
  → verificar licencia y autor
  → descargar a zona de revisión local
  → normalizar formato permitido
  → calcular bytes y SHA-256
  → crear/actualizar manifiesto
  → ejecutar validador
  → generar reporte
  → revisión humana
  → aprobar activo en public/catalog
```

El validador nunca descarga activos. Trabaja exclusivamente con archivos locales y manifiestos. Una URL válida no sustituye la revisión de la licencia.

## 6. Arquitectura y estructura

```text
public/catalog/index.json                 Índice versionado
public/catalog/experiences/*.json         Quince manifiestos
public/catalog/assets/**                  Activos aprobados
src/catalog/catalog-types.ts              Contratos
src/catalog/catalog-schema.ts             Validación estructural
src/catalog/catalog-loader.ts             Lectura del índice/manifiestos
src/catalog/catalog-search.ts             Búsqueda y filtros puros
src/features/explore/**                   Interfaz de catálogo
scripts/validate-catalog.ts               Compuerta ejecutable
scripts/report-catalog.ts                 Licencias y presupuesto
tests/catalog/fixtures/**                  Fixtures propios válidos/inválidos
reports/catalog/**                         Reportes generados y versionados
```

La aplicación consume un `CatalogRepository`; los componentes no hacen `fetch` directo ni interpretan JSON sin validar.

## 7. Base técnica y comandos

I2 conserva el stack de I1. No agrega una librería de esquema sin aprobación; la validación se implementa con TypeScript estricto y funciones puras mientras resulte mantenible.

```powershell
npm ci
npm run validate:catalog
npm run report:catalog
npm run test -- src/catalog tests/catalog
npm run test:e2e -- e2e/catalog.spec.ts
npm run verify
```

`validate:catalog` devuelve exit code distinto de cero ante cualquier incumplimiento. `report:catalog` no modifica manifiestos ni activos.

## 8. Estilo de código

- Lectura, validación, búsqueda y presentación son unidades separadas.
- Los validadores devuelven todos los errores ordenados por ruta y campo; no se detienen en el primero.
- La búsqueda normaliza mayúsculas y diacríticos, pero no modifica el catálogo.
- No se usa `any`; JSON entra como `unknown`.

```ts
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
```

## 9. Estrategia de pruebas

- **Unitarias:** formato de campos, unicidad, rutas, fechas, SHA-256, presupuesto y búsqueda.
- **Integración:** índice más quince manifiestos; bytes/hash contra archivos locales; reporte determinista.
- **E2E:** mostrar 15 experiencias, filtrar 5 por modo, buscar por texto/tags y representar cero resultados.
- **Negativas:** licencia vacía, hash alterado, activo faltante, ruta externa, ID duplicado y presupuesto superior al límite.
- **Cobertura:** mínimo 90% en validación, carga y búsqueda; mínimo global 80%.

## 10. Manejo de errores

- Un manifiesto inválido no entra al `CatalogRepository` activo.
- Un activo faltante o con hash diferente bloquea la validación completa.
- El reporte identifica archivo, campo y código; no oculta errores posteriores.
- La UI ante catálogo inválido conserva el catálogo sintético de I1 y muestra un estado recuperable.
- Una búsqueda sin coincidencias no se considera error.

## 11. Criterios de aceptación

1. Existen exactamente 15 IDs, cinco por modo y sin duplicados.
2. Cada experiencia declara tres perfiles de intensidad completos y todas sus referencias resuelven a assets propios.
3. Cada activo tiene licencia, procedencia, atribución, bytes y SHA-256 verificados.
4. El tamaño único total no supera 300 MB.
5. El validador falla de forma determinista para cada fixture inválido definido.
6. El reporte enumera todos los activos, licencias, atribuciones y presupuesto total.
7. Explorar lista el catálogo y filtra cinco experiencias por modo.
8. La búsqueda funciona por título y tags, sin distinguir mayúsculas o diacríticos.
9. La UI no intenta reproducir capas reales antes de I3.
10. `npm run verify` finaliza correctamente.

## 12. Límites de trabajo

### Siempre hacer

- Conservar evidencia de licencia y procedencia por activo.
- Calcular integridad sobre el archivo que realmente se versiona.
- Revisar manualmente el reporte antes de aprobar un activo.

### Preguntar primero

- Aceptar una licencia no contemplada o con restricciones ambiguas.
- Cambiar IDs estables, esquema, formatos de audio o presupuesto.
- Agregar dependencias de producción o una fuente que requiera autenticación.

### Nunca hacer

- Incorporar activos sin permiso claro de redistribución y modificación.
- Copiar o usar audio de Brain.fm.
- Hacer que el validador descargue, reemplace o elimine activos.

## 13. Preguntas abiertas

No hay decisiones de producto pendientes para planificar I2. La identidad de cada activo permanece pendiente de verificación durante la ejecución y no se considera aprobada por aparecer como candidata.

## 14. Compuerta SDD

La aprobación explícita quedó registrada el 2026-08-07 y autoriza escribir el plan I2. Ejecutarlo requiere aprobación separada del plan; incorporar cada activo exige además evidencia de licencia dentro del repositorio.
