# easyFold — Architecture Reference

> Documento de referencia técnica para futuros contextos de IA o desarrolladores.
> Describe el stack, la estructura interna del archivo y las decisiones de implementación relevantes.

---

## Stack

| Capa | Solución | Notas |
|---|---|---|
| Distribución | Un único `index.html` | Sin servidor, sin npm, abre con `file://` |
| Estilos | Tailwind CSS Play CDN | `https://cdn.tailwindcss.com` |
| 3D | Three.js **r128** desde cdnjs | `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` — **no usar versiones > r150**: a partir de r160+ el `build/three.min.js` es ESM y no expone `window.THREE` globalmente |
| Lógica | JavaScript vanilla (ES5/ES6 sin módulos) | Sin `import`/`export`; todas las funciones son globales en `window` |
| Estado | Objeto global `APP` + `render()` manual | Sin framework reactivo |
| Persistencia | `localStorage` con clave `"easyfold_config"` | Solo configuración del usuario |
| Tests | IIFE `runTests()` ejecutada al cargar | `console.assert` + mensaje verde en consola |

---

## Estructura del archivo `index.html`

El archivo está organizado en bloques `<script>` con comentarios de sección:

```
<head>
  CDNs (Tailwind, Three.js)
  <style>  → animación CSS del panel inline + .tabular

<body>
  <header>         → título, subtítulo, botón ⚙
  <main>
    #section-form    → renderizado por renderForm()
    #section-results → renderizado por render()
  <footer>
  #modal-config    → renderizado por renderConfigModal()

<!-- LÓGICA -->    funciones puras, sin DOM
<!-- ESTADO -->    objeto APP, motor de cálculo, dispose de viewers
<!-- RENDER -->    funciones que generan y modifican el DOM
<!-- VISUALIZACIONES 3D -->  mountGaugeViewer, mountRollViewer
<!-- TESTS -->     IIFE runTests()
<!-- INIT -->      DOMContentLoaded: loadConfig(), render(), listeners globales
```

---

## Dominio

**easyFold** resuelve: dado un rollo de lámina plástica plegado sobre sí mismo, encontrar todas las tuplas `(a,b,c,d,e,f,g)` válidas para unos parámetros de entrada `cP` (cantidad a plegar) y `aR` (altura de rollo).

### Restricciones del problema
1. `a + b + c + d + e + f + g = cP`
2. `a = aR` (fija)
3. Todas las variables `≤ aR`
4. `c ≤ b`, `e ≤ d`, `g ≤ f` (orden por pares)
5. Continuidad: si alguna variable en `b..f` es 0, todas las siguientes también son 0
6. Simetría D: si `d > 0` → `c = b`
7. Simetría F: si `f > 0` → `e = d`
8. Todos los valores son múltiplos de 0.5

---

## Funciones clave — LÓGICA

| Función | Firma | Descripción |
|---|---|---|
| `validateInputs` | `(cP, aR) → {ok, error?}` | Valida enteros positivos |
| `calculateCombinations` | `(cP, aR) → Combo[]` | Backtracking con semi-unidades (×2) para evitar errores de float |
| `computeLayerExtents` | `(vars) → [{lo,hi,idx}]` | Extensión Y física de cada capa activa |
| `computeGaugeZones` | `(vars, aR, config) → {zones, maxCount, ranges}` | Divide `[0,aR]` en zonas de gramaje constante |
| `effectiveG` | `(zona, galgas, aR, config) → number` | Gramaje en una zona, incluye contribución de CA |
| `computeRollScore` | `(vars, aR, galgas, config) → [0,1]` | Uniformidad (85%) + solidez central (15%) |
| `getQualityLevel` | `(score) → QualityLevel` | Mapea score a nivel (Excelente/Muy bueno/Bueno/Regular/Deficiente) |
| `enrichCombination` | `(combo, aR, galgas, config) → CombinationRow` | Añade `cP`, `aR`, `score`, `maxGaugeLayers` |
| `combinationsToCSV` | `(rows) → string` | Genera CSV con cabecera `cP,aR,a,b,c,d,e,f,g` |
| `downloadCSV` | `(cP, aR, rows)` | Crea blob y dispara descarga |
| `gaugeColor` | `(t) → "rgb(...)"` | Gradiente rojo-oscuro→verde para t∈[0,1] |
| `lighten` | `(hexInt, t) → hexInt` | Mezcla un color con blanco (para capas 3D) |
| `getVisualOrder` | `(numLayers) → int[]` | Orden izquierda→derecha para el diagrama SVG |
| `getPageRange` | `(page, totalPages) → (int\|"…")[]` | Lógica de elipsis para la paginación |
| `rowKey` | `(combo) → string` | Clave única: `"b-c-d-e-f-g"` (a omitida, siempre = aR) |

### Truco de semi-unidades en el backtracking
El algoritmo multiplica todos los valores por 2 internamente (`aR2 = aR×2`, `R2 = (cP−aR)×2`) para trabajar con enteros y evitar errores de punto flotante al comparar `remaining === 0`. Los valores se dividen por 2 al insertar en el array de resultados.

### Bug crítico resuelto (T-45)
El array `current[0..5]` es compartido entre todas las ramas. Al hacer `push` desde el check de continuidad (`current[pos-1] === 0`), las posiciones `>= pos` contienen basura de rutas anteriores. **Fix:** usar `pos > k ? current[k]/2 : 0` al construir el objeto resultado.

---

## Estado global — `APP`

```js
APP = {
  form:   { cP, aR, galgas, cintaAutocierre: 'sin'|'con' },
  config: { plegadoCinta, anchoCinta, galgasCinta, distanciaSeguridad },
  configOpen: bool,

  results: {
    totalCount, page, pageSize, totalPages,
    combinations: CombinationRow[],   // página actual
    availableGaugeLayers: number[],
    cP, aR, galgas, config,
    allEnriched: CombinationRow[],     // todas (para CSV)
  } | null,

  qualityFilter:    string | null,   // key de QUALITY_LEVELS
  gaugeLayerFilter: number | null,

  hoveredKey:        string | null,
  lockedKey:         string | null,
  lockedCombination: CombinationRow | null,
  instantClose:      bool,

  loading:    bool,
  csvLoading: bool,
  error:      string | null,
  pageSize:   number,               // default 50

  viewers: {                        // keyed por containerId
    [id]: { animId, renderer, observer }
  },
}
```

---

## Ciclo de render

```
usuario interactúa
      ↓
handler actualiza APP
      ↓
render() → innerHTML de #section-form y #section-results
      ↓
bindResultsEvents() → re-adjunta listeners
      ↓ (si lockedKey)
rAF×2 → gridEl.open → transitionend (220ms)
      ↓
mountViewers() → mountGaugeViewer() + mountRollViewer()
```

`render()` es destructivo: reemplaza `innerHTML` completo en cada llamada. No hay diff/reconciliación. Los viewers 3D se montan siempre después del `transitionend` para garantizar que `container.clientWidth > 0`.

---

## Visualizaciones 3D

### Ciclo de vida de un viewer
1. `render()` genera los contenedores `#gauge-viewer-KEY` y `#roll-viewer-KEY` en el HTML del panel.
2. Tras `transitionend` del panel, `mountViewers()` crea la escena Three.js y arranca el loop `requestAnimationFrame`.
3. El ID de cada frame se actualiza en `APP.viewers[id].animId` en cada iteración (no solo en el primero).
4. Al cerrar el panel, `disposeViewer(id)` cancela el loop, llama `renderer.dispose()` y desconecta el `ResizeObserver`.

### GaugeViewer — cilindro coloreado por gramaje
- Un `CylinderGeometry` por zona de gramaje (no un único cilindro con `vertexColors`): evita dependencia del orden interno de vértices en Three.js r128.
- Color de cada segmento: `gaugeColor(effectiveG(zona) / gMax)`.
- Primer y último segmento con tapas (`openEnded: false`); intermedios sin tapas para evitar z-fighting.

### RollViewer — capas en profundidad
- Orden de profundidad: `getVisualOrder(numLayers).slice().reverse()` — la última capa del orden visual (b) queda al frente (mayor Z), la primera (a) queda al fondo.
- Cada capa: `BoxGeometry` con color pastel (`lighten(hex, 0.65)`) + borde superior en color pleno + sprite de letra con `CanvasTexture`.
- Capa A ajusta su X para tocar la superficie del cilindro: `xOff = cylX + sqrt(cylR² − z²)`.

### Parámetros de cámara
| Viewer | FOV | Radio inicial | CAM_Y | Target |
|---|---|---|---|---|
| GaugeViewer | 38° | 8 | 3.5 | (0, 1.4, 0) |
| RollViewer  | 38° | 11 | 4.0 | (1.0, 1.4, 0) |

---

## Constantes importantes

```js
QUALITY_LEVELS  // [{key, label, short, min, color, bg}] — 5 niveles
LAYER_COLORS    // [0x3b82f6…] — 7 capas + CA (rose-500)
DEFAULT_CONFIG  // {plegadoCinta:5, anchoCinta:5, galgasCinta:80, distanciaSeguridad:1}

// En los viewers:
STRIP_H = 2.8   // altura del cilindro en unidades Three.js
Y_SCALE = 2.8 / aR
Z_STEP  = 0.22  // separación entre capas en eje Z (RollViewer)
```

---

## Notas para futuros cambios

- **Añadir una capa h:** ampliar `LAYER_COLORS`, `getVisualOrder`, `computeLayerExtents`, el backtracking (pos 0..6 → 0..7), y todos los `push` del algoritmo.
- **Cambiar Three.js:** usar solo versiones ≤ r150 con `build/three.min.js` UMD, o migrar a `type="module"` con importmap y refactorizar para eliminar los globales.
- **Añadir persistencia de resultados:** `APP.results` se puede serializar a `sessionStorage` para sobrevivir recargas.
- **Rendimiento:** `calculateCombinations` es síncrono. Para cP grandes (>100) puede bloquear el hilo. Migrar a `Worker` envolviendo la función en un blob URL.
- **Tests:** el bloque `runTests()` usa `console.assert`. Para ampliar, añadir casos al final de la IIFE antes del `console.log` de confirmación.
