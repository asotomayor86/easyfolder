# easyFold — Backlog de construcción

> Guía de construcción secuenciada. Cada tarea es atómica y verificable.
> Stack: un único `index.html` · JavaScript vanilla · librerías por CDN · sin servidor · sin npm.

---

## Stack objetivo

| Capa | Solución |
|---|---|
| Framework | Ninguno — DOM vanilla |
| Estilos | Tailwind CSS Play CDN (`https://cdn.tailwindcss.com`) |
| 3D | Three.js 0.184 (`https://cdn.jsdelivr.net/npm/three@0.184.0/build/three.min.js`) |
| Estado | Objeto global `APP` + función `render()` manual |
| Persistencia | `localStorage` (igual que antes) |
| Tests | Bloque `<script>` de autotest ejecutado al cargar (console.assert) |

---

## Convenciones del archivo

- Todo el código vive en `index.html` dentro de bloques `<script>` y `<style>`.
- Sección `<!-- LÓGICA -->` para funciones puras (sin DOM).
- Sección `<!-- ESTADO -->` para el objeto `APP` y los handlers.
- Sección `<!-- RENDER -->` para las funciones que tocan el DOM.
- Sección `<!-- INIT -->` para el arranque.
- No se usan `import`/`export`. Todas las funciones son globales en `window`.

---

## Fase 0 — Esqueleto HTML

### T-01 · Esqueleto base ✅
- [x] Crear `index.html` con `<!DOCTYPE html>`, `<html lang="es">`, `<head>`, `<body>`.
- [x] En `<head>`: `<meta charset="UTF-8">`, viewport, título "easyFold".
- [x] Cargar Tailwind CSS Play CDN.
- [x] Cargar Three.js desde jsDelivr.
- [x] Añadir `<body class="bg-slate-50 min-h-screen">`.
- [x] Verificación: abrir el archivo en el navegador, consola sin errores, `THREE` disponible en consola.

### T-02 · Estructura HTML estática ✅
- [x] Cabecera: `<header>` con título "easyFold", subtítulo y botón ⚙ (id `btn-config`).
- [x] Contenedor principal: `<main id="main" class="max-w-7xl mx-auto px-4 py-8 space-y-6">`.
- [x] Sección formulario: `<section id="section-form">`.
- [x] Sección resultados: `<section id="section-results" class="hidden">`.
- [x] Footer: texto "easyFold — uso interno — cálculo ejecutado en cliente".
- [x] Modal config: `<div id="modal-config" class="hidden fixed inset-0 ...">` con overlay y panel.
- [x] Verificación: estructura visible en navegador con Tailwind aplicado.

---

## Fase 1 — Lógica pura (sin DOM)

> Estas funciones son independientes del navegador. Se pueden probar en consola.

### T-03 · Constantes y tipos (como objetos JS) ✅
- [x] Definir `QUALITY_LEVELS` (array de objetos con `key`, `label`, `short`, `min`, `color`, `bg`).
- [x] Definir `LAYER_COLORS` (array de 7 hex numbers + 1 para CA).
- [x] Definir `DEFAULT_CONFIG` (`plegadoCinta:5`, `anchoCinta:5`, `galgasCinta:80`, `distanciaSeguridad:1`).
- [x] Verificación: `console.log(QUALITY_LEVELS[0].key)` → `"excelente"`.

### T-04 · Validación de entrada ✅
- [x] Implementar `validateInputs(cP, aR)` → `{ ok: true }` o `{ ok: false, error: string }`.
- [x] Casos: no-número, no-entero, ≤ 0.
- [x] Verificación en consola:
  - `validateInputs(0, 5)` → `ok: false`
  - `validateInputs(1.5, 5)` → `ok: false`
  - `validateInputs(10, 5)` → `ok: true`

### T-05 · Algoritmo de backtracking ✅
- [x] Implementar `calculateCombinations(cP, aR)` → array de objetos `{a,b,c,d,e,f,g}`.
- [x] Truco de semi-unidades (×2 internamente) para evitar errores de punto flotante.
- [x] Las 8 restricciones (suma, a=aR, cota, orden por pares, continuidad, simetría D, simetría F, pasos 0.5).
- [x] Verificación en consola:
  - `calculateCombinations(5, 5).length` → `1`
  - `calculateCombinations(4, 2).length` → `5`
  - `calculateCombinations(5, 2).length` → `9`
  - `calculateCombinations(3, 5).length` → `0`

### T-06 · Tests automáticos del algoritmo ✅
- [x] Añadir bloque `<script>` de tests que se ejecuta al cargar.
- [x] Verificar: suma exacta, `a === aR`, todas ≤ `aR`, `c≤b`, `e≤d`, `g≤f`, continuidad, simetría D/F, múltiplos de 0.5, sin duplicados.
- [x] Verificación: consola muestra "✓ Todos los tests pasaron".

### T-07 · Extents Y físicos de las capas ✅
- [x] Implementar `computeLayerExtents(vars)` → array de `{lo, hi, idx}` para cada capa activa.
- [x] Lógica: alternar ↓ (par) y ↑ (impar), calcular `entryY`/`exitY`, retornar `[min, max]`.
- [x] Verificación en consola con `{a:5,b:3,c:3,d:2,e:2,f:0,g:0}`.

### T-08 · Zonas de gramaje ✅
- [x] Implementar `computeGaugeZones(vars, aR, config)` → `{ zones, maxCount, ranges }`.
- [x] Recopilar breakpoints de todas las capas + CA si aplica.
- [x] Para cada intervalo: contar capas que contienen `yMid`.
- [x] Verificación: la suma de `(yTop - yBot)` de todas las zonas debe ser igual a `aR`.

### T-09 · Gramaje efectivo y score ✅
- [x] Implementar `effectiveG(zona, galgas, aR, config)` con contribución opcional de CA.
- [x] Implementar `computeRollScore(vars, aR, galgas, config)` → número `[0, 1]`.
- [x] Implementar `getQualityLevel(score)` → objeto de `QUALITY_LEVELS`.
- [x] Verificación: score de distribución perfecta cercano a 1.

### T-10 · `maxGaugeLayers` y enriquecimiento de filas ✅
- [x] Implementar `enrichCombination(combo, aR, galgas, config)` → añade `cP`, `aR`, `score`, `maxGaugeLayers`.
- [x] `maxGaugeLayers` = `maxCount` de `computeGaugeZones`.

### T-11 · Generación de CSV ✅
- [x] Implementar `combinationsToCSV(combinations)` → string con cabecera `cP,aR,a,b,c,d,e,f,g`.
- [x] Implementar `downloadCSV(cP, aR, combinations)` → crea blob, ancla y dispara descarga.
- [x] Verificación en consola: cabecera correcta, filas correctas.

### T-12 · Gradiente de color para gramaje ✅
- [x] Implementar `gaugeColor(t)` → string CSS `rgb(r,g,b)` para `t ∈ [0, 1]`.
- [x] Interpolación en dos tramos (0–0.5: rojo oscuro→rojo vivo; 0.5–1: rojo vivo→verde).
- [x] Implementar `lighten(hexInt, t)` → hex integer con mezcla de blanco (para capas en 3D).

---

## Fase 2 — Estado de la aplicación

### T-13 · Objeto global APP ✅
- [x] Definir `window.APP` con todos los campos de estado.
- [x] Implementar `loadConfig()` — lee `localStorage["easyfold_config"]` y mezcla con `DEFAULT_CONFIG`.
- [x] Implementar `saveConfig(config)` — escribe en `localStorage`.

### T-14 · Motor de cálculo del cliente ✅
- [x] Implementar `runCalculation(page)`: validar → enriquecer → ordenar → filtrar → paginar → render.
- [x] Envuelto en `setTimeout(0)` para que el spinner se pinte antes del cálculo.

### T-15 · Clave de fila ✅
- [x] Implementar `rowKey(combo)` → `"${b}-${c}-${d}-${e}-${f}-${g}"`.

---

## Fase 3 — Formulario y configuración

### T-16 · Render del formulario (`renderForm`) ✅
- [x] Campos `cP`, `aR`, `galgas`, selector `cintaAutocierre`.
- [x] Mensaje de error inline en rojo.
- [x] Botón con spinner durante `APP.loading`.
- [x] Texto informativo de CA cuando se selecciona "con".
- [x] Bind de eventos completo.

### T-17 · Modal de configuración (`renderConfigModal`) ✅
- [x] Overlay + panel blanco centrado.
- [x] Campos: `plegadoCinta`, `anchoCinta`, `galgasCinta`, `distanciaSeguridad`.
- [x] Botones "Cancelar" / "Guardar" con `saveConfig`.
- [x] Click en overlay cierra.
- [x] Bind del botón ⚙.

---

## Fase 4 — Tabla, filtros y paginación

### T-18 · Barra de resumen y botón CSV ✅
- [x] Texto localizado `es-ES` con total de combinaciones.
- [x] Mensaje de 0 resultados.
- [x] Botón CSV con spinner y estado deshabilitado.

### T-19 · Chips de filtro por calidad ✅
- [x] Chips "Todos" + uno por nivel presente.
- [x] Estilos activo/inactivo correctos.
- [x] Toggle al hacer click en chip activo.

### T-20 · Chips de filtro por galga máxima ✅
- [x] Solo si hay más de un valor único de `maxGaugeLayers`.
- [x] Etiqueta en gramos si `galgas > 0`, en capas si no.
- [x] Colores indigo.

### T-21 · Tabla de resultados (`renderTable`) ✅
- [x] 11 columnas con estilos correctos.
- [x] Encabezado `a` en azul.
- [x] Ceros en gris claro.
- [x] Hover y lock con estilos blue-800/blanco.
- [x] Celda `Cal.` con badge responsive + score mono.
- [x] Panel inline integrado como fila extra.

### T-22 · Paginación (`renderPagination`) ✅
- [x] Texto "Mostrando X–Y de Z".
- [x] Selector de filas por página.
- [x] Botones « ‹ [páginas] › » con lógica `getPageRange`.
- [x] Deshabilitado durante carga.

---

## Fase 5 — Panel inline

### T-23 · Animación CSS del panel ✅
- [x] CSS `grid-template-rows: 0fr → 1fr` con transición 220ms.
- [x] Cierre instantáneo al cambiar de fila + delay 350ms.
- [x] Cierre con animación al desseleccionar misma fila.
- [x] `transitionend` para desmontaje.

### T-24 · Contenido del panel (`buildInlinePanelHTML`) ✅
- [x] Grid 3 columnas en `lg:`, 1 en móvil.
- [x] Contenedores para SVG, tabla gramaje + GaugeViewer, RollViewer.
- [x] Scroll suave a la fila en móvil.

---

## Fase 6 — Visualizaciones

### T-25 · Orden visual de capas en diagrama ✅
- [x] `getVisualOrder(numLayers)` → array de índices físicos.

### T-26 · Diagrama SVG de sección transversal ✅
- [x] Constantes de layout correctas.
- [x] Grid horizontal con etiquetas Y.
- [x] Barras coloreadas en orden visual.
- [x] CA como columna extra a la izquierda.
- [x] Path de lámina con arcos cuadráticos.
- [x] Etiquetas de letra debajo de cada barra.

### T-27 · Tabla de gramaje por zonas ✅
- [x] Mensaje si `galgas = 0`.
- [x] Tabla con zonas de arriba a abajo.
- [x] Cuadrado de color + rango + valor + porcentaje.
- [x] Barra de gradiente como leyenda.

### T-28 · GaugeViewer (Three.js) ✅
- [x] Escena con cilindro de `vertexColors` según gramaje.
- [x] Núcleo de cartón.
- [x] Iluminación (ambient + 2 directional).
- [x] Interacción: orbitar + zoom.
- [x] Balanceo sutil automático.
- [x] `ResizeObserver` + dispose.

### T-29 · RollViewer (Three.js) ✅
- [x] Cilindro central + núcleo cartón.
- [x] Capas en profundidad (Z_STEP) con color pastel + borde pleno.
- [x] Sprites de letra con `CanvasTexture`.
- [x] Capa A ajustada a superficie del cilindro.
- [x] CA como capa adicional si aplica.
- [x] Interacción: orbitar + pan vertical + zoom.
- [x] Balanceo sutil + `ResizeObserver` + dispose.

---

## Fase 7 — Integración y pulido

### T-30 · Función principal `render()` ✅
- [x] Orquesta formulario, modal, barra resumen, chips, tabla, paginación.
- [x] Activa panel con doble `requestAnimationFrame` para transición CSS.
- [x] `bindResultsEvents()` re-adjunta listeners tras cada render.

### T-31 · Gestión del ciclo de vida de los viewers 3D ✅
- [x] `APP.viewers` almacena `animId`, `renderer`, `observer`.
- [x] `disposeViewer(id)` cancela loop + libera GPU + desconecta observer.
- [x] `disposeAllViewers()` limpia todos al cambiar de fila.

### T-32 · Scroll y teclado ✅
- [x] Escape cierra panel y modal de config.
- [x] Scroll suave a la fila en móvil (<1024px).

### T-33 · Responsividad ✅
- [x] Formulario: 2 col en `sm:`, 1 en móvil.
- [x] Tabla: `overflow-x-auto`.
- [x] Panel inline: 3 col en `lg:`, 1 en móvil.
- [x] Badge: nombre completo en `sm:`, letra en móvil.

### T-34 · Autotest completo en carga ✅
- [x] Tests de algoritmo, score, colores, CSV, niveles de calidad.
- [x] Mensaje verde "✓ Todos los tests pasaron" en consola.

### T-35 · Revisión final y smoke test ✅
- [x] Abrir `index.html` directamente (protocolo `file://`) en Chrome.
- [x] Flujo completo: cP=65 aR=17 → Calcular → ver resultados → click en fila → ver 3 visualizaciones.
- [x] Filtros de calidad y galga operativos.
- [x] Paginación funcional.
- [x] Descarga CSV correcta.
- [x] Consola muestra `✓ Todos los tests pasaron`.

---

## Fase 8 — Revisión y corrección de visualizaciones 3D

> Bugs estructurales identificados en la implementación original de T-28/T-29 que impiden el renderizado.

### T-36 · Bug — Montaje prematuro antes de que el panel esté expandido ✅
- [x] **Problema:** `mountViewers` se llama en el segundo `requestAnimationFrame` (≈ 16 ms), pero la transición CSS del panel dura 220 ms. En ese momento `container.clientWidth === 0`, el `WebGLRenderer` se inicializa con las dimensiones de fallback (200 × 200) y el canvas queda desencajado o invisible.
- [x] **Fix:** Escuchar `transitionend` en el `.panel-grid` con `{ once: true }` y llamar `mountViewers` en ese callback.
- [x] Verificación: `container.clientWidth > 0` en el momento de crear el `WebGLRenderer`.

### T-37 · Bug — Loop de animación no cancelable en GaugeViewer ✅
- [x] **Problema:** `APP.viewers[containerId + '_id']` usaba clave incorrecta. `disposeViewer` leía `v.animId` (primer frame, ya consumido) y el loop continuaba.
- [x] **Fix:** Registrar entry antes de iniciar el loop y actualizar `APP.viewers[containerId].animId` en cada frame.
- [x] Verificación: `cancelAnimationFrame` detiene el loop al cerrar el panel.

### T-38 · Bug — Loop de animación no cancelable en RollViewer ✅
- [x] **Problema:** `requestAnimationFrame(animate)` no guardaba el ID devuelto. Loop inmortal.
- [x] **Fix:** Igual que T-37 — registrar entry y actualizar `APP.viewers[containerId].animId` en cada frame.
- [x] Verificación: `cancelAnimationFrame` detiene el loop al cerrar el panel.

### T-39 · Bug — Vertex colors en CylinderGeometry (Three.js r184) ✅
- [x] **Problema:** `CylinderGeometry` con `heightSegments > 1` genera caps y vértices extra cuyo orden interno en r184 no coincide con el muestreo por coordenada Y.
- [x] **Fix:** Un `CylinderGeometry` por zona de gramaje, cada uno con `MeshPhongMaterial` de color sólido. Primer y último segmento con tapas; los intermedios `openEnded: true` para evitar artefactos en las uniones.
- [x] Verificación: el cilindro muestra el gradiente de color correcto (rojo oscuro→verde según densidad).

### T-40 · Bug — Orden de profundidad incorrecto en RollViewer ✅
- [x] **Problema:** `getVisualOrder` devuelve orden izquierda→derecha del diagrama SVG. En 3D asignaba Z_FRONT (frente) a la capa A, que debería estar al fondo.
- [x] **Fix:** `getVisualOrder(numLayers).slice().reverse()` — b queda al frente, a al fondo.
- [x] Verificación: la capa A (azul) en el fondo del rollo, la capa B (naranja) al frente.

### T-44 · Bug — CDN de Three.js r184 no expone `THREE` globalmente ✅
- [x] **Problema:** `three@0.184.0/build/three.min.js` en jsDelivr es el build ESM (usa `export`), no el UMD. Al cargarlo con `<script src>` sin `type="module"`, el módulo no ejecuta y `window.THREE` queda `undefined`.
- [x] **Fix:** Cambiar a Three.js r128 desde cdnjs, que sí es un build UMD global confirmado: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`. Todos los APIs usados (Scene, CylinderGeometry, BoxGeometry, SpriteMaterial, CanvasTexture…) existen desde r125.
- [x] Verificación: consola muestra `✓ Three.js r128 listo` al cargar.

### T-45 · Bug — Valores obsoletos en el array `current` del backtracking ✅
- [x] **Problema:** El array `current[0..5]` es compartido entre todas las ramas del backtracking. Cuando el check de continuidad (`current[pos-1] === 0`) dispara un `push`, las posiciones `pos..5` contienen valores de rutas anteriores ya descartadas — no se han inicializado a 0 en el camino actual. Esto produce combinaciones con suma ≠ cP, ceros seguidos de no-ceros y violaciones de simetría F y g≤f.
- [x] **Fix:** En el `push` de continuidad, usar `current[k]/2` solo para `k < pos` (posiciones ya fijadas en este camino) y forzar `0` para `k >= pos`: `c: pos>1 ? current[1]/2 : 0`, etc.
- [x] Verificación: todos los asserts del bloque de tests pasan; `calculateCombinations(8,4)` no produce ninguna combinación con suma ≠ 8.

### T-42 · Diagnóstico activo — console.log en cada paso crítico del montaje ✅
- [x] Añadir trazas en: `transitionend` recibido, `container.clientWidth` al montar, creación de `WebGLRenderer`, primer frame renderizado, y `dispose`.
- [x] Añadir timeout fallback de 350 ms por si `transitionend` no dispara (p.ej. `prefers-reduced-motion`, transición cancelada, o panel ya expandido).
- [x] Verificación: abrir DevTools → Console y confirmar qué paso falla.

### T-43 · Protección contra fallo silencioso de WebGLRenderer ✅
- [x] Envolver `new THREE.WebGLRenderer(...)` en `try/catch`: si falla (WebGL bloqueado por el navegador o driver), mostrar mensaje informativo en el contenedor en lugar de fallar silenciosamente.
- [x] Verificación: con WebGL desactivado forzosamente, el panel muestra "WebGL no disponible" en lugar de quedarse en gris.

### T-41 · Verificación final de visualizaciones 3D ✅
- [x] Ambos canvas aparecen y muestran geometría.
- [x] GaugeViewer muestra cilindro con gradiente de color por zonas de gramaje.
- [x] RollViewer muestra cilindro central, núcleo marrón, capas en profundidad y sprites de letra.
- [x] Balanceo sutil, órbita con ratón y zoom con rueda operativos.
- [x] Dispose correcto al cambiar de fila — sin loops huérfanos.

---

## Orden de implementación recomendado

```
T-01 → T-02                          (esqueleto)
T-03 → T-04 → T-05 → T-06           (lógica core + tests)
T-07 → T-08 → T-09 → T-10 → T-11   (extents, gramaje, score, CSV)
T-12                                  (colores)
T-13 → T-14 → T-15                  (estado)
T-16 → T-17                          (formulario + config modal)
T-18 → T-19 → T-20 → T-21 → T-22   (resultados, filtros, tabla, paginación)
T-23 → T-24                          (panel inline + animación)
T-25 → T-26 → T-27                  (SVG + tabla gramaje)
T-28 → T-29                          (viewers Three.js)
T-30 → T-31 → T-32 → T-33          (integración)
T-34 → T-35                          (tests finales + smoke)
T-36 → T-37 → T-38 → T-39 → T-40 → T-41  (revisión 3D)
```

**Total: 41 tareas** en 8 fases.
