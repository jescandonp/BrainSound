# BrainSound MVP — Hoja de ruta de implementación

- **SPEC de origen:** `docs/specs/2026-08-07-brainsound-mvp-design.md`
- **Fecha:** 2026-08-07
- **Estado:** propuesta para aprobación
- **Tipo:** roadmap de trazabilidad; no es un plan ejecutable
- **Regla SDD:** cada iteración requiere primero una SPEC aprobada y después un plan detallado aprobado antes de modificar código.

## Motivo de la descomposición

El MVP reúne cinco subsistemas con riesgos distintos: fundación PWA, contrato de catálogo, motor híbrido de audio, experiencia y personalización, y operación offline con calidad de entrega. Implementarlos como un solo bloque impediría validar software funcional entre etapas. Esta hoja de ruta establece cinco incrementos; cada uno termina con una aplicación ejecutable y verificable.

## Secuencia

```text
I1 Fundación vertical
        ↓
I2 Catálogo y trazabilidad
        ↓
I3 Motor híbrido de audio
        ↓
I4 Experiencia personal completa
        ↓
I5 Offline transaccional y entrega MVP
```

## I1 — Fundación PWA y sesión vertical

**Estado:** Planificada.

**SPEC de iteración:** `docs/specs/2026-08-07-brainsound-i1-foundation-spec.md` — propuesta, pendiente de aprobación.

**Objetivo:** producir una PWA instalable con identidad Energía cromática, tres modos, audio sintético de respaldo, reproductor compacto, temporizador libre y recarga offline.

**Entrega verificable:** una persona abre BrainSound, selecciona cualquiera de los tres modos, escucha un paisaje sintético local, pausa o reanuda, observa el tiempo transcurrido y recarga la aplicación sin red.

**Plan detallado:** `docs/plans/2026-08-07-brainsound-i1-foundation.md` — suspendido hasta aprobación de la SPEC I1 y revisión de trazabilidad.

**Compuerta de salida:** lint, tipos, unitarias, cobertura, compilación, catálogo de respaldo y E2E Chromium/WebKit en verde.

## I2 — Catálogo, licencias y canal de activos

**Estado:** Planificada.

**SPEC de iteración:** `docs/specs/2026-08-07-brainsound-i2-catalog-spec.md` — propuesta, pendiente de aprobación.

**Plan detallado previsto:** `docs/plans/2026-08-07-brainsound-i2-catalog.md`; no se escribe hasta aprobar la SPEC I2.

**Objetivo:** establecer el contrato definitivo del catálogo y la entrada segura de activos redistribuibles.

**Entrega verificable:** BrainSound lista las quince experiencias desde manifiestos versionados; el proceso de validación rechaza activos sin licencia, atribución, tamaño o integridad y produce un reporte trazable.

**Incluye:** esquema de manifiesto, cinco experiencias por modo, registro de procedencia, sumas de integridad, presupuesto de 300 MB, búsqueda básica y pruebas de validación.

**Compuerta de salida:** todos los manifiestos válidos, reporte de licencias completo, catálogo dentro del presupuesto y navegación de las quince experiencias.

## I3 — Motor híbrido, intensidad y continuidad

**Estado:** Planificada.

**SPEC de iteración:** `docs/specs/2026-08-07-brainsound-i3-audio-engine-spec.md` — propuesta, pendiente de aprobación.

**Plan detallado previsto:** `docs/plans/2026-08-07-brainsound-i3-audio-engine.md`; no se escribe hasta aprobar la SPEC I3.

**Objetivo:** reemplazar el uso normal del sintetizador de respaldo por mezcla híbrida de música, ambiente y naturaleza.

**Entrega verificable:** cada experiencia reproduce capas locales, cambia entre intensidades suave/media/profunda y transiciona en 8–12 segundos sin silencio inesperado; el sintetizador de I1 permanece como degradación segura.

**Incluye:** grafo Web Audio, precarga, plan de sesión, control de ganancia, limitación, fundidos, cambio de experiencia y pruebas de audio con adaptadores deterministas.

**Compuerta de salida:** inicio máximo de 1,5 segundos con activos instalados, ausencia de clipping en fixtures y transiciones verificadas.

## I4 — Experiencia, temporizadores y personalización local

**Estado:** Planificada.

**SPEC de iteración:** `docs/specs/2026-08-07-brainsound-i4-personal-experience-spec.md` — propuesta, pendiente de aprobación.

**Plan detallado previsto:** `docs/plans/2026-08-07-brainsound-i4-personal-experience.md`; no se escribe hasta aprobar la SPEC I4.

**Objetivo:** completar la experiencia funcional de uso personal.

**Entrega verificable:** calibración inicial, inicio con un clic, explorar/buscar, favoritos, recientes, cuenta regresiva, intervalos, valoración, selector local explicable, progreso y rachas.

**Incluye:** IndexedDB versionado, navegación Inicio/Explorar/Favoritos/Progreso/Ajustes, temporizadores completos y migraciones probadas.

**Compuerta de salida:** recorridos E2E de primer uso y uso habitual, desviación del temporizador menor a un segundo por hora y persistencia correcta tras recarga.

## I5 — Instalación transaccional, respaldo y entrega MVP

**Estado:** Planificada.

**SPEC de iteración:** `docs/specs/2026-08-07-brainsound-i5-offline-release-spec.md` — propuesta, pendiente de aprobación.

**Plan detallado previsto:** `docs/plans/2026-08-07-brainsound-i5-offline-release.md`; no se escribe hasta aprobar la SPEC I5.

**Objetivo:** cerrar la instalación completa del catálogo y la calidad multiplataforma.

**Entrega verificable:** descarga con progreso y reanudación, verificación por activo, activación atómica, rollback, exportación/importación recuperable y funcionamiento offline completo.

**Incluye:** actualización entre sesiones, manejo de cuota, respaldo versionado, auditoría de red, accesibilidad, Edge/Chrome/Safari, documentación de uso y preparación del alojamiento estático.

**Compuerta de salida:** las trece condiciones de éxito de la SPEC aprobadas con evidencia; despliegue únicamente después de una autorización separada.

## Dependencias y límites

- I2 depende de los contratos y comandos de I1.
- I3 depende de los manifiestos validados de I2.
- I4 consume el motor estable de I3 y no accede directamente a Web Audio.
- I5 consume almacenamiento y catálogo estables; no redefine sus contratos sin actualizar la SPEC.
- El audio propietario de Brain.fm nunca se usa como fixture, referencia descargable o activo.
- Cuentas, backend, sincronización, telemetría, funciones sociales y contenido de sueño permanecen fuera del MVP.

## Política de cambios

Un cambio dentro de una iteración actualiza primero su SPEC si afecta comportamiento o contratos, y después su plan. Un cambio de alcance, arquitectura, navegadores, catálogo, afirmaciones o criterios de éxito actualiza primero la SPEC maestra. Ningún plan previsto se escribe antes de aprobar su SPEC de iteración.
