# Registro de SPECs de BrainSound

Este directorio es la fuente normativa del desarrollo. Un roadmap organiza; una SPEC autoriza el alcance; un plan aprobado autoriza tareas; ninguna de esas aprobaciones autoriza despliegue.

## Jerarquía

1. `2026-08-07-brainsound-mvp-design.md` — alcance y arquitectura del MVP completo; aprobada.
2. SPEC de iteración — concreta contratos y criterios sin ampliar la SPEC maestra.
3. Plan de iteración — solo se escribe después de aprobar su SPEC.
4. Ejecución — solo comienza después de aprobar el plan.

## Matriz de control

| Iteración | SPEC | Estado SPEC | Plan | Estado plan |
|---|---|---|---|---|
| I1 | `2026-08-07-brainsound-i1-foundation-spec.md` | Propuesta | `../plans/2026-08-07-brainsound-i1-foundation.md` | Suspendido hasta aprobación y revisión |
| I2 | `2026-08-07-brainsound-i2-catalog-spec.md` | Propuesta | `../plans/2026-08-07-brainsound-i2-catalog.md` | No creado |
| I3 | `2026-08-07-brainsound-i3-audio-engine-spec.md` | Propuesta | `../plans/2026-08-07-brainsound-i3-audio-engine.md` | No creado |
| I4 | `2026-08-07-brainsound-i4-personal-experience-spec.md` | Propuesta | `../plans/2026-08-07-brainsound-i4-personal-experience.md` | No creado |
| I5 | `2026-08-07-brainsound-i5-offline-release-spec.md` | Propuesta | `../plans/2026-08-07-brainsound-i5-offline-release.md` | No creado |

## Regla de aprobación

La aprobación debe ser explícita y quedar registrada en el documento afectado. Una aprobación colectiva puede cubrir las cinco SPECs si el usuario las identifica como conjunto. Cualquier observación mantiene la SPEC correspondiente en propuesta y bloquea su plan.

## Regla de cambio

- Alcance o arquitectura del MVP: actualizar y aprobar la SPEC maestra.
- Comportamiento o contrato de una iteración: actualizar y aprobar su SPEC.
- Secuencia técnica dentro de alcance aprobado: actualizar y aprobar su plan.
- Código, activos, dependencias o configuración ejecutable: solo después de ambas aprobaciones.
- Publicación o despliegue: autorización separada, incluso con I5 completada.
