# BrainSound MVP — Especificación de diseño

- **Versión:** 0.1.0
- **Fecha:** 2026-08-07
- **Estado:** diseño conversacional aprobado; documento pendiente de revisión del usuario
- **Método:** Spec-Driven Development (SPEC → PLAN → TASKS → IMPLEMENT)

## 1. Objetivo

BrainSound será una aplicación web instalable y gratuita para uso personal que ofrece audio funcional para productividad y bienestar. Debe permitir iniciar en segundos sesiones de Deep Focus, Creatividad o Relajación, personalizar la experiencia y utilizar íntegramente el catálogo sin conexión después de la instalación inicial.

El MVP busca paridad funcional del núcleo de uso personal de Brain.fm, no copiar su marca, interfaz, catálogo, código, tecnología patentada ni afirmaciones científicas. BrainSound se presentará como una herramienta de apoyo; no realizará afirmaciones médicas, terapéuticas o de eficacia clínica.

### Usuario objetivo

- Un único usuario por instalación.
- Uso principal en computadores Windows y macOS.
- Navegadores objetivo: Edge, Chrome y Safari actuales.
- Sin cuenta, suscripción, sincronización remota ni administración multiusuario.

### Resultado esperado

Una PWA estática, visualmente expresiva y sin distracciones, capaz de descargar, validar y reproducir un catálogo híbrido de audio con personalización y datos completamente locales.

## 2. Decisiones y supuestos explícitos

- BrainSound es el nombre aprobado para el MVP.
- La interfaz inicial será en español.
- El MVP no tendrá backend ni dependerá de APIs pagas.
- El alojamiento únicamente servirá archivos estáticos mediante HTTPS.
- El catálogo completo tendrá un objetivo máximo de 300 MB.
- Los datos de uso permanecerán en el dispositivo y podrán exportarse e importarse.
- No habrá telemetría, publicidad, rastreadores ni envío de hábitos de escucha.
- La reproducción debe funcionar con audífonos o altavoces; la aplicación podrá recomendar audífonos, pero no exigirlos.
- Las capacidades no soportadas por un navegador tendrán degradación explícita y segura.

## 3. Alcance funcional

### Incluido en el MVP

1. Calibración inicial de sensibilidad sonora, preferencia rítmica, estilos y duración habitual.
2. Tres modos: Deep Focus, Creatividad y Relajación.
3. Cinco experiencias por modo; quince experiencias base en total.
4. Tres intensidades por experiencia: suave, media y profunda.
5. Identidad sonora equilibrada: ambient, electrónica suave, lo-fi instrumental, acústica minimalista y naturaleza.
6. Inicio de sesión con un clic usando la última configuración útil del modo.
7. Reproductor continuo con volumen, pausa, cambio de experiencia, favorito e intensidad.
8. Temporizadores de sesión libre, cuenta regresiva e intervalos configurables.
9. Exploración y búsqueda por modo, género, ambiente e intensidad.
10. Favoritos, recientes, historial, minutos acumulados y rachas locales.
11. Personalización local mediante reglas transparentes, favoritos, descartes, escucha reciente y valoración opcional.
12. Descarga transaccional del catálogo completo para uso sin conexión.
13. Exportación e importación versionada de preferencias, favoritos e historial.
14. Instalación como PWA en Windows y macOS.
15. Navegación por teclado, lector de pantalla, contraste adecuado y movimiento reducido.
16. Información verificable de procedencia, licencia y atribución de cada activo.

### Excluido del MVP

- Sueño, siestas, meditación y contenido guiado por voz.
- Cuentas, pagos, perfiles compartidos o sincronización entre dispositivos.
- Funciones sociales, publicación de rachas o comparación con otros usuarios.
- Aplicaciones nativas móviles o de escritorio.
- Modo o afirmaciones clínicas para TDAH.
- Integración con dispositivos biométricos.
- Generación musical mediante servicios externos.
- Paridad cuantitativa con miles de pistas.

## 4. Experiencia de usuario

### Primer uso

1. La persona conoce el alcance de BrainSound y acepta iniciar la calibración.
2. Responde preguntas no médicas sobre sensibilidad, pulso, estilos y duración.
3. La aplicación verifica compatibilidad y espacio disponible.
4. Informa el tamaño total y solicita iniciar la descarga del catálogo.
5. Descarga, verifica y activa todos los activos.
6. Presenta los tres modos y permite comenzar una sesión.

### Uso habitual

1. La pantalla de inicio muestra Deep Focus, Creatividad y Relajación.
2. Un clic inicia el modo con la última combinación útil.
3. El reproductor compacto permite ajustar intensidad, temporizador y experiencia.
4. Al terminar, registra duración y ofrece una valoración opcional.
5. El selector local utiliza la interacción para ordenar futuras recomendaciones.

### Navegación

- **Inicio:** modos, reanudación rápida y sesión actual.
- **Explorar:** búsqueda, filtros y las quince experiencias.
- **Favoritos:** favoritos y reproducciones recientes.
- **Progreso:** sesiones, minutos y rachas sin presión social.
- **Ajustes:** audio, temporizadores, almacenamiento, accesibilidad, respaldo y licencias.

### Identidad visual

Dirección aprobada: **Energía cromática**. Cada modo tendrá un color dominante, tipografía expresiva, tarjetas amplias y un reproductor compacto. El color nunca será la única señal de estado o navegación.

## 5. Arquitectura

BrainSound será una aplicación cliente estática. El alojamiento distribuye la PWA y el catálogo; toda la lógica de sesión y persistencia ocurre en el navegador.

```text
Alojamiento estático HTTPS
          │
          ▼
PWA + Service Worker ──────► Cache Storage (app, manifiesto y audio)
          │
          ├───────────────► IndexedDB (perfil, sesiones y ajustes)
          │
          ▼
Orquestador de sesión
          │
          ▼
Motor Web Audio ───────────► Salida de audio
```

### Componentes

- **App Shell:** navegación, diseño, estados globales y manejo de actualización.
- **Onboarding:** calibración y consentimiento para la descarga.
- **Catalog:** manifiestos, filtros, búsqueda, licencias e integridad.
- **Session Orchestrator:** selección, plan de sesión y coordinación con temporizadores.
- **Audio Engine:** carga, mezcla, intensidad, fundidos y salida segura.
- **Timers:** libre, regresivo e intervalos; pausa sincronizada con audio.
- **Personalization:** puntuación local y explicable; sin modelo remoto.
- **Offline Installer:** prevalidación, descarga, reanudación, verificación y activación.
- **Local Storage:** esquema versionado y migraciones de IndexedDB.
- **Backup:** exportación, validación, resumen e importación recuperable.
- **Progress:** historial, tiempo acumulado y rachas.

## 6. Motor de audio y catálogo

### Flujo de sesión

1. Recibir modo, temporizador, calibración, favoritos, descartes y recientes.
2. Puntuar experiencias compatibles y reducir repetición inmediata.
3. Crear un plan con experiencia, capas, intensidad, duración y siguiente alternativa.
4. Precargar activos antes de reproducir.
5. Mezclar capas de música, ambiente y naturaleza mediante Web Audio.
6. Encadenar cambios con fundidos de 8 a 12 segundos.
7. Registrar el resultado localmente al finalizar.

### Intensidad

- **Suave:** menor densidad y profundidad de modulación.
- **Media:** equilibrio predeterminado entre música, pulso y ambiente.
- **Profunda:** mayor presencia rítmica y contraste controlado.

Los cambios de intensidad serán progresivos. El motor incorporará control de ganancia y limitación para evitar clipping. La intensidad se describirá como preferencia sonora, no como tratamiento o medición neurológica.

### Contrato de experiencia

Cada experiencia declarará:

- ID estable y versión.
- Modo, géneros y ambientes.
- Lista de activos y funciones de cada capa.
- Puntos válidos de bucle y duración estimada.
- Perfiles de intensidad.
- Tamaño y suma de integridad por activo.
- Procedencia, licencia y atribución.

Solo se incluirán activos cuya licencia permita expresamente redistribución, modificación y almacenamiento offline para este proyecto.

## 7. Persistencia, offline y actualizaciones

### Instalación transaccional

1. Prevalidar navegador, red y espacio estimado.
2. Crear una caché temporal separada de la versión activa.
3. Descargar con progreso total y por activo.
4. Conservar el progreso necesario para reanudar.
5. Verificar manifiesto, tamaño e integridad.
6. Activar la nueva versión de forma atómica.
7. Liberar la versión anterior únicamente después de la activación exitosa.

Una descarga parcial nunca se anunciará como instalada. Una actualización no interrumpirá una sesión activa.

### Datos locales

IndexedDB almacenará:

- Perfil de calibración.
- Preferencias y última configuración por modo.
- Favoritos, descartes y recientes.
- Sesiones, duración, valoración y rachas.
- Estado de instalación y versión de catálogo.

### Respaldo

El archivo JSON tendrá versión de esquema y no incluirá audio. La importación se validará completamente antes de modificar datos. Se mostrará un resumen, se pedirá confirmación y se generará una copia recuperable antes del reemplazo.

## 8. Manejo de errores

- **Espacio insuficiente:** mostrar requerido y disponible; conservar versión activa.
- **Activo corrupto:** reintentar solo ese activo; no activar catálogo incompleto.
- **Audio bloqueado por el navegador:** solicitar una interacción explícita para comenzar.
- **Función Web Audio no soportada:** usar reproducción básica si existe una alternativa segura.
- **Respaldo inválido:** rechazar sin modificar datos locales.
- **Migración fallida:** conservar el esquema anterior y ofrecer recuperación.
- **Primera visita sin red:** explicar que la instalación inicial requiere conexión.
- **Actualización disponible:** aplicarla únicamente entre sesiones o por acción del usuario.

## 9. Base técnica y versiones

Versiones verificadas el 2026-08-07; se fijarán exactamente en el archivo de bloqueo:

- Node.js 24.18.1 LTS.
- TypeScript 7.0.2.
- React y React DOM 19.2.8.
- Vite 8.1.5.
- Vitest 4.1.10.
- Playwright Test 1.61.1.
- `idb` 8.0.3.
- Web Audio API, Service Worker, Cache Storage e IndexedDB nativos.

No se agregará una dependencia para una capacidad que pueda implementarse con una API web estable sin introducir complejidad desproporcionada.

## 10. Comandos previstos

```powershell
npm ci
npm run dev
npm run build
npm run preview
npm run lint
npm run typecheck
npm run test
npm run test:coverage
npm run test:e2e
npm run validate:catalog
npm run verify
```

`npm run verify` ejecutará lint, tipos, pruebas, cobertura, compilación y validación de catálogo. Las pruebas E2E podrán ejecutarse por separado para mantener ciclos locales rápidos.

## 11. Estructura del proyecto

```text
docs/specs/       Especificaciones SDD aprobadas
docs/plans/       Planes de implementación aprobados
public/catalog/   Activos y manifiestos del catálogo
src/app/          Shell, rutas y composición
src/features/     Onboarding, player, explore, progress y settings
src/audio/        Motor Web Audio y planificación de sesión
src/catalog/      Contratos, búsqueda, licencias y validación
src/storage/      IndexedDB, migraciones y respaldo
src/pwa/          Service Worker, caché e instalación
src/shared/       Tipos, utilidades y componentes reutilizables
src/styles/       Tokens y estilos globales
tests/            Pruebas de integración compartidas
e2e/              Pruebas Playwright
scripts/          Validación de catálogo y tareas de proyecto
```

Cada módulo tendrá una responsabilidad clara y expondrá contratos tipados. La interfaz no accederá directamente a IndexedDB, Cache Storage o nodos Web Audio.

## 12. Estilo de código

- TypeScript estricto; evitar `any` y conversiones inseguras.
- Componentes React en PascalCase; funciones y variables en camelCase.
- Archivos de dominio en kebab-case.
- Exportaciones nombradas; sin exportación predeterminada salvo exigencia del framework.
- Lógica de selección, temporización y validación como funciones puras cuando sea posible.
- Efectos y APIs del navegador detrás de adaptadores inyectables.

Ejemplo de contrato y función pura:

```ts
export type BrainSoundMode = 'deep-focus' | 'creativity' | 'relaxation';
export type Intensity = 'soft' | 'medium' | 'deep';

export interface SessionRequest {
  mode: BrainSoundMode;
  intensity: Intensity;
  durationSeconds: number | null;
}

export function isTimedSession(request: SessionRequest): boolean {
  return request.durationSeconds !== null;
}
```

## 13. Estrategia de pruebas

### Unitarias

- Selector y reglas de personalización.
- Temporizadores y cálculo de tiempo transcurrido.
- Manifiestos, integridad, respaldo y migraciones.
- Estados de descarga y actualización.

### Integración

- Orquestador con adaptador de audio.
- IndexedDB y migraciones.
- Instalación, reanudación y rollback de caché.
- Importación y exportación completa.

### E2E

- Calibración y descarga inicial.
- Inicio con un clic en los tres modos.
- Temporizadores, favoritos, búsqueda y progreso.
- Recarga y reproducción sin conexión.
- Navegación completa por teclado.

### Manual y especializada

- Calidad sonora y transiciones con audífonos y altavoces.
- Safari real en macOS.
- Lector de pantalla y movimiento reducido.
- Inspección de red durante sesión offline.

### Cobertura

- Cobertura global mínima: 80%.
- Lógica crítica de selección, temporización, catálogo, almacenamiento y respaldo: 90%.

## 14. Criterios de éxito

1. Las quince experiencias pueden reproducirse sin conexión después de una instalación validada.
2. Una sesión instalada comienza en máximo 1,5 segundos desde la acción del usuario en los equipos de prueba soportados.
3. No existen silencios inesperados perceptibles durante fundidos o cambios de experiencia.
4. El temporizador se desvía menos de un segundo por hora respecto al reloj monotónico usado como referencia.
5. Pausar detiene de forma coherente audio y temporizador.
6. Una descarga o actualización fallida conserva la versión funcional anterior.
7. Exportar e importar reproduce preferencias, favoritos e historial exactamente.
8. Una sesión offline no produce solicitudes externas.
9. Todas las funciones principales operan con teclado.
10. Las pruebas automatizadas de accesibilidad no reportan errores críticos.
11. El catálogo completo no supera 300 MB.
12. Cada activo aprobado posee licencia, procedencia, atribución e integridad verificadas.
13. `npm run verify` finaliza correctamente antes de cada entrega.

## 15. Límites de trabajo

### Siempre hacer

- Actualizar la especificación antes de cambiar alcance o arquitectura.
- Escribir pruebas antes o junto con la lógica implementada.
- Validar entradas, respaldos y manifiestos.
- Ejecutar `npm run verify` antes de afirmar que una iteración está completa.
- Mantener trazabilidad de licencias e integridad del catálogo.
- Preservar accesibilidad y degradación segura.

### Preguntar primero

- Agregar dependencias de producción.
- Cambiar el esquema persistente o el formato de respaldo.
- Superar el objetivo de 300 MB.
- Agregar backend, telemetría, cuentas o sincronización.
- Modificar navegadores objetivo o criterios de éxito.
- Cambiar el alcance de modos, catálogo o afirmaciones del producto.
- Publicar o desplegar fuera del entorno acordado.

### Nunca hacer

- Copiar audio, marca, interfaz, código o contenido propietario de Brain.fm.
- Presentar el producto como tratamiento médico o tecnología clínicamente validada.
- Incluir activos sin licencia de redistribución y modificación verificable.
- Registrar secretos o información privada en el repositorio.
- Eliminar pruebas fallidas para obtener una verificación verde.
- Transmitir hábitos de escucha sin una futura especificación y aprobación explícitas.

## 16. Riesgos y mitigaciones

- **Cuota de almacenamiento variable:** prevalidar, mostrar tamaño y conservar rollback.
- **Diferencias de Web Audio:** adaptadores, pruebas WebKit y reproducción básica.
- **Calidad o monotonía:** capas híbridas, variación controlada y auditoría sonora.
- **Licencias incompatibles:** manifiesto obligatorio y compuerta automática de catálogo.
- **Descarga inicial pesada:** progreso, reanudación e integridad por activo.
- **Expectativas científicas:** lenguaje responsable y ausencia de afirmaciones clínicas.

## 17. Preguntas abiertas

No existen preguntas de producto que bloqueen el plan. La selección exacta de los activos de audio será una tarea de implementación gobernada por los requisitos de licencia, calidad, tamaño e integridad definidos aquí.

## 18. Fuentes de referencia

- Funcionalidades públicas de referencia: <https://www.brain.fm/>.
- Fundamento declarado por el producto de referencia: <https://www.brain.fm/science>.
- Versiones de paquetes: páginas oficiales de React, Vite, Vitest, Playwright, TypeScript e `idb` en npm.
- Ciclo de soporte de Node.js: <https://nodejs.org/en/about/previous-releases>.

## 19. Compuerta SDD

Este documento debe ser revisado y aprobado explícitamente por el usuario antes de crear el plan de implementación. La aprobación del diseño conversacional no sustituye la revisión de esta especificación escrita.
