# BrainSound I5 — SPEC de instalación transaccional, respaldo y entrega MVP

- **SPEC maestra:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Iteración:** I5 de 5
- **Estado:** aprobada explícitamente por el usuario el 2026-08-07; planificación autorizada
- **Dependencias:** I1–I4 completadas con sus compuertas verdes

## 1. Objetivo

Cerrar el MVP de BrainSound con instalación y actualización recuperables del catálogo completo, funcionamiento offline real, exportación/importación local, manejo de cuota, accesibilidad y compatibilidad en Windows/macOS con Edge, Chrome y Safari. I5 prepara un build estático publicable, pero no lo despliega sin autorización separada.

## 2. Supuestos explícitos

- Los activos y manifiestos provienen del catálogo aprobado en I2.
- El motor I3 y el esquema de datos I4 son estables antes de crear migraciones o respaldos.
- La instalación inicial puede necesitar red; después de completarse, las funciones principales no la necesitan.
- El almacenamiento disponible varía por navegador; se consulta y prevalida cuando la API lo permite.
- Cache Storage contiene assets de aplicación/catálogo; IndexedDB contiene estado de instalación y datos personales.
- No se usa alojamiento, CDN, telemetría o backend durante la ejecución local de I5.

## 3. Alcance funcional

### Incluido

1. Estimación de cuota y tamaño antes de descargar.
2. Descarga con progreso por bytes/activos y reanudación después de interrupción.
3. Verificación de tamaño y SHA-256 por activo.
4. Staging separado de la versión activa.
5. Activación atómica del nuevo catálogo y rollback ante fallo.
6. Actualización solo entre sesiones, nunca durante reproducción.
7. Estado visible de instalación, versión y recuperación.
8. Exportación e importación JSON versionada de preferencias, favoritos e historial.
9. Validación completa antes de modificar datos locales durante una importación.
10. Auditoría offline y de solicitudes externas.
11. Accesibilidad automatizada y manual; teclado y movimiento reducido.
12. Verificación Edge/Chrome en Windows y Chrome/Safari en macOS.
13. Documentación de instalación, privacidad, respaldo, recuperación y límites.
14. Build estático preparado para alojamiento gratuito, sin despliegue.

### Excluido

- Publicación efectiva, dominio, analítica, cuentas o sincronización.
- Actualizaciones en segundo plano durante una sesión.
- Importación desde formatos de terceros.
- Recuperación de un respaldo cifrado o protegido por contraseña.
- Tienda de aplicaciones o empaquetado nativo.

## 4. Máquina de estados de instalación

```ts
export type InstallState =
  | { readonly kind: 'not-installed' }
  | { readonly kind: 'checking'; readonly targetVersion: string }
  | { readonly kind: 'downloading'; readonly targetVersion: string; readonly completedBytes: number; readonly totalBytes: number }
  | { readonly kind: 'verifying'; readonly targetVersion: string; readonly completedAssets: number; readonly totalAssets: number }
  | { readonly kind: 'ready-to-activate'; readonly targetVersion: string }
  | { readonly kind: 'installed'; readonly activeVersion: string }
  | { readonly kind: 'failed'; readonly activeVersion: string | null; readonly recoverable: boolean; readonly reason: string };
```

Reglas:

- Solo `ready-to-activate` puede pasar a `installed` con la nueva versión.
- La versión activa no cambia durante descarga o verificación.
- El staging identifica versión y activos confirmados para reanudar.
- Cancelar conserva la versión activa y elimina solo staging de la versión objetivo.
- Una sesión activa bloquea activación; la actualización queda pendiente.
- Después de activación exitosa se elimina la versión anterior únicamente cuando una prueba de lectura del catálogo activo pasa.

## 5. Transacción de instalación

```text
leer índice remoto aprobado
  → estimar cuota y solicitar confirmación
  → crear staging(version)
  → descargar cada activo faltante
  → verificar bytes + SHA-256
  → verificar manifiestos completos
  → marcar ready-to-activate
  → esperar ausencia de sesión
  → cambiar puntero activeVersion
  → prueba de lectura/reproducción de respaldo
  → limpiar versión anterior
```

La red solo accede al origen estático configurado para BrainSound. Toda URL se deriva del índice aprobado; no se siguen URLs de activos arbitrarias desde metadatos de atribución.

## 6. Respaldo e importación

```ts
export interface BrainSoundBackupV1 {
  readonly format: 'brainsound-backup';
  readonly version: 1;
  readonly exportedAt: string;
  readonly appVersion: string;
  readonly data: {
    readonly preferences: UserPreferences;
    readonly favoriteExperienceIds: readonly string[];
    readonly sessions: readonly SessionRecord[];
  };
}
```

Reglas:

- Exportar produce UTF-8 JSON y no incluye audio, caché, diagnósticos ni secretos.
- La importación recibe `unknown`, valida formato, versión, límites, IDs y fechas antes de escribir.
- Máximo de importación: 10 MB y 100.000 sesiones.
- Duplicados de sesión por ID se reemplazan solo si el registro importado es exactamente igual; un conflicto diferente cancela toda la importación.
- Favoritos ausentes del catálogo se conservan como referencias archivadas, sin recomendarlos.
- La escritura usa staging IndexedDB y activa todos los datos o ninguno.
- Antes de importar se ofrece descargar un respaldo actual; rechazarlo no bloquea la importación después de confirmación explícita.

## 7. Arquitectura y estructura

```text
src/pwa/install/install-state.ts          Máquina de estados
src/pwa/install/install-controller.ts     Orquestación
src/pwa/install/quota-port.ts             Estimación de espacio
src/pwa/install/download-port.ts          Descarga/reanudación
src/pwa/install/catalog-cache.ts          Staging y versión activa
src/pwa/install/integrity.ts              SHA-256 y tamaño
src/storage/backup/backup-schema.ts       Validación versionada
src/storage/backup/export-backup.ts       Serialización
src/storage/backup/import-backup.ts       Transacción de datos
src/features/settings/installation/**     Estado y acciones
src/features/settings/backup/**           Exportar/importar
tests/pwa/**                              Interrupciones y rollback
tests/storage/backup/**                   Compatibilidad y atomicidad
e2e/offline-mvp.spec.ts                   MVP sin red
e2e/backup.spec.ts                        Round trip
docs/user/**                              Guías finales
```

Los controladores dependen de puertos de red, caché, cuota y almacenamiento para ser deterministas en pruebas.

## 8. Base técnica y comandos

I5 conserva el stack aprobado. Usa `crypto.subtle.digest('SHA-256', ...)`, Cache Storage, IndexedDB y StorageManager nativos.

```powershell
npm ci
npm run test -- src/pwa src/storage/backup tests/pwa tests/storage/backup
npm run test:coverage
npm run test:e2e -- e2e/offline-mvp.spec.ts e2e/backup.spec.ts
npm run audit:network
npm run audit:accessibility
npm run validate:catalog
npm run build
npm run verify
```

Los comandos de auditoría devuelven exit code distinto de cero ante solicitudes externas o errores críticos de accesibilidad.

## 9. Estilo de código

- Estados como uniones discriminadas; transiciones en función pura antes de efectos.
- Descarga y caché detrás de puertos; ninguna llamada de red desde componentes.
- Importación `unknown` y sin conversiones forzadas.
- Operaciones destructivas limitadas a staging o versiones identificadas explícitamente.

```ts
export interface InstallPorts {
  readonly quota: QuotaPort;
  readonly download: DownloadPort;
  readonly cache: CatalogCachePort;
  readonly integrity: IntegrityPort;
}

export interface ImportResult {
  readonly importedSessions: number;
  readonly importedFavorites: number;
  readonly warnings: readonly string[];
}
```

## 10. Estrategia de pruebas

- **Unitarias:** transiciones, progreso, cuota, validación de respaldo y conflictos.
- **Integración:** interrupción en cada fase, reanudación, hash fallido, rollback y doble ejecución.
- **Backup:** exportar/importar reproduce exactamente preferencias, favoritos e historial.
- **E2E offline:** instalar, desconectar, recorrer funciones principales y reproducir las 15 experiencias.
- **Red:** cero solicitudes externas durante sesión offline; solo origen configurado durante instalación.
- **Accesibilidad:** auditoría automatizada sin críticos y recorrido manual de teclado/lector.
- **Compatibilidad:** Edge/Chrome en Windows; Chrome/Safari en macOS, con matriz firmada.
- **Cobertura:** 90% en instalación, integridad, importación y rollback; global 80%.

## 11. Manejo de errores

- Cuota insuficiente: impedir descarga antes de modificar staging y mostrar bytes requeridos/disponibles.
- Red interrumpida: conservar progreso confirmado y ofrecer reanudar/cancelar.
- Hash o tamaño incorrecto: eliminar solo el asset inválido de staging y bloquear activación.
- Activación fallida: restaurar puntero anterior y conservar versión funcional.
- Respaldo inválido: mostrar todos los errores y no modificar la base.
- Navegador sin estimación de cuota: advertir incertidumbre y permitir continuar con confirmación.
- Service Worker incompatible: conservar acceso online y explicar que offline completo no está disponible.

## 12. Criterios de aceptación

1. La instalación muestra tamaño, progreso y versión objetivo.
2. Una interrupción puede reanudarse sin repetir activos ya verificados.
3. Un hash fallido o cuota insuficiente nunca reemplaza la versión activa.
4. La activación ocurre entre sesiones y es atómica.
5. Una actualización fallida conserva la versión funcional anterior.
6. Las 15 experiencias funcionan sin red después de instalación validada.
7. El catálogo instalado no supera 300 MB.
8. Exportar e importar reproduce preferencias, favoritos e historial exactamente.
9. Una importación inválida no realiza escrituras parciales.
10. No existen solicitudes externas durante uso offline.
11. Funciones principales operan con teclado y auditoría automática sin críticos.
12. La matriz Edge/Chrome/Safari cumple los recorridos definidos.
13. Los trece criterios de éxito de la SPEC maestra tienen evidencia enlazada.
14. `npm run verify` finaliza correctamente.
15. El build queda preparado, pero no desplegado.

## 13. Límites de trabajo

### Siempre hacer

- Mantener versión activa hasta verificar completamente la nueva.
- Validar un respaldo completo antes de escribir.
- Registrar evidencia de navegador, red, accesibilidad y tamaño.

### Preguntar primero

- Cambiar formato de backup, esquema persistente, origen de descarga o presupuesto.
- Agregar dependencias, cifrado, telemetría o un proveedor de hosting.
- Publicar, desplegar, crear dominio o enviar datos.

### Nunca hacer

- Eliminar una versión funcional antes de activar y probar la nueva.
- Importar parcialmente un respaldo inválido.
- Desplegar por inferencia o considerar preparación como autorización.

## 14. Preguntas abiertas

El proveedor de alojamiento gratuito no se selecciona en esta SPEC porque desplegar requiere una autorización separada. Esta decisión no bloquea el plan local ni el build estático.

## 15. Compuerta SDD

La aprobación explícita quedó registrada el 2026-08-07 y autoriza escribir el plan I5. Ejecutarlo requiere aprobación separada del plan e I1–I4 completadas; publicar requiere una autorización adicional posterior a la verificación.
