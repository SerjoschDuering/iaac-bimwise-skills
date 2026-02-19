# 3D Viewer — IFC Direct Rendering

The platform renders IFC files directly in the browser using `@thatopen/components` v3.
No backend GLB conversion. No S3 for 3D files. The backend only runs checks.

```
Upload IFC → R2 → Frontend loads IFC directly via @thatopen/components
                   HF Space only runs checks, POSTs results back
```

## Stack

| Package | Version | Role |
|---------|---------|------|
| `@thatopen/components` | ^3.3.0 | IFC viewer engine (scene, camera, loader) |
| `@thatopen/components-front` | ^3.3.0 | Frontend extras (highlighter, etc.) |
| `@thatopen/fragments` | ^3.3.0 | Fragment-based model management |
| `web-ifc` | ^0.0.75 | Emscripten WASM — parses IFC binary format |
| `three` | ^0.175.0 | 3D rendering (required by ThatOpen) |

## WASM Loading Pattern (CRITICAL)

**web-ifc is an Emscripten-compiled WASM module. It CANNOT be processed by any
JavaScript bundler (Vite, esbuild, Rollup, Webpack). All of them mangle the
Emscripten glue code and break WASM instantiation.**

The error you'll see if bundled:
```
WebAssembly.instantiate(): Import #0 "a": module is not an object or function
```

### Why Bundlers Break It

Vite's WASM helper returns `WebAssembly.Exports` but Emscripten's `instantiateWasm`
callback expects a `WebAssembly.Module` — the mismatch crashes initialization.
esbuild has no native WASM loader at all. Neither bundler can process Emscripten glue.

### The Fix: Load Outside the Bundle

Three files work together:

**1. `index.html` — IIFE script tag (loads before the app bundle)**
```html
<script src="/web-ifc-api-iife.js"></script>
<script type="module" src="/src/main.tsx"></script>
```

**2. `src/lib/web-ifc-shim.ts` — re-exports from the global**
```ts
const W = (globalThis as any).WebIFC;
export default W;
export const IfcAPI = W.IfcAPI, LogLevel = W.LogLevel;
// ... all 1176 named exports (auto-generated)
```

**3. `vite.config.ts` — alias redirects ONLY the bare import to the shim**
```ts
resolve: {
  alias: [
    { find: /^web-ifc$/, replacement: path.resolve(__dirname, "src/lib/web-ifc-shim.ts") },
  ],
},
optimizeDeps: {
  exclude: ["@thatopen/components", "@thatopen/components-front", "@thatopen/fragments"],
},
```

**CRITICAL:** Use regex `/^web-ifc$/` — NOT a string key `"web-ifc"`. A string alias
also matches subpath imports like `web-ifc/web-ifc.wasm?url`, breaking WASM path resolution.

When `@thatopen/components` does `import { IfcAPI } from 'web-ifc'`, Vite resolves it
to the shim, which reads `globalThis.WebIFC.IfcAPI` — populated by the IIFE script tag.
The Emscripten WASM glue runs in its own script scope, untouched by any bundler.

### Static Files in `public/`

These must be copied from `node_modules/web-ifc/` after install:

| File | Size | Purpose |
|------|------|---------|
| `web-ifc-api-iife.js` | ~6 MB | Pre-built IIFE (built by ThatOpen from TS source) |
| `web-ifc.wasm` | ~1.3 MB | Single-threaded WASM binary |
| `web-ifc-mt.wasm` | ~1.3 MB | Multi-threaded WASM (needs COOP/COEP headers) |
| `fragments-worker.mjs` | ~1.3 MB | FragmentsManager Web Worker |

### Generating the Shim

The shim must re-export all 1176 named symbols from `web-ifc`. From `frontend/`:

```bash
node -e "const fs=require('fs'),c=fs.readFileSync('node_modules/web-ifc/web-ifc-api.js','utf8'),m=c.match(/export\s*\{([^}]+)\}/s),e=m[1].split(',').map(s=>s.trim()).filter(Boolean).map(r=>{const p=r.split(/\s+as\s+/);return p[1]?.trim()??r});let s='// AUTO-GENERATED\nconst W=(globalThis as any).WebIFC;\nexport default W;\n';for(let i=0;i<e.length;i+=6){const b=e.slice(i,i+6);s+='export const '+b.map(n=>n+'=W.'+n).join(',')+';\\n'}fs.writeFileSync('src/lib/web-ifc-shim.ts',s);console.log(e.length+' exports')"
```

**When upgrading `web-ifc`:** re-run this script AND copy new IIFE + WASM to `public/`.

## Viewer Architecture

### Zustand State (viewerSlice)

The viewer is driven by a Zustand slice. Any component can read/write viewer state.

```ts
// Key state
ifcUrl: string | null       // URL to fetch IFC from (set by route loader)
colorMap: Record<string, string>  // GlobalId → hex color ("#e62020" = fail)
selectedIds: Set<string>    // GlobalIds currently selected
hiddenIds: Set<string>      // GlobalIds hidden from view
isReady: boolean            // true when ThatOpen engine initialized
```

### BIMViewer.tsx

Renders a ThatOpen `Components` engine in a `<div>`. Key lifecycle:

1. **Init** — creates `Components`, `SimpleScene`, `SimpleRenderer`, `OrthoPerspectiveCamera`
2. **Load** — when `ifcUrl` changes, fetches IFC, runs `IfcLoader.load()`
3. **GUID map** — builds `GlobalId → expressID` map via `ifcLoader.webIfc.GetLine()`
4. **Color** — applies `colorMap` via `FragmentsManager.highlight()` (v3 API)
5. **Cleanup** — `components.dispose()` on unmount

### useViewer.ts

Thin bridge: watches `elementResults` from checksSlice, computes `colorMap`:
- `fail` → `#e62020` (red, full opacity)
- `pass` → `#d0d3da` (grey, semi-transparent)

### Data Flow

```
Upload IFC → R2 → route loader sets ifcUrl
                   → BIMViewer fetches IFC, renders in ThatOpen
                   → builds GUID map

Run checks → HF Space processes → results stored in D1
          → checksSlice.elementResults populated
          → useViewer computes colorMap
          → BIMViewer applies colors via FragmentsManager.highlight()
```

## Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| "Import #0 'a' module is not an object" | Bundler processed web-ifc | Use IIFE + shim pattern above |
| "pd is not a constructor" | Shim exports `W.IfcAPI2` instead of `W.IfcAPI` | IIFE applies aliases — always use the exported name |
| "You need to initialize fragments first" | `FragmentsManager.init()` not called | Call `fragments.init("/fragments-worker.mjs")` before loading |
| 8-char hex warns + no-op | `#d0d3da33` (with alpha) | Use 6-char hex: `#d0d3da` |
| colorMap computed before model loads | Race condition | Call `applyColors()` both after load AND on colorMap change |
| WASM fails in Cloudflare context | `@cloudflare/vite-plugin` applies `workerd` condition | The alias + shim bypasses this entirely |

## Best Practices

### DO

| Practice | Why |
|----------|-----|
| `fragments.core.disposeModel(modelId)` to remove models | `scene.remove(obj)` only removes from render — fragment worker leaks memory |
| `ifcLoader.setup()` once in the init effect | Re-running setup on every load re-initializes web-ifc unnecessarily |
| Wire camera for LOD: `controls.addEventListener("update", () => fragments.core.update())` | Without this, tiles/LOD freeze after initial load |
| `model.useCamera(world.camera.three)` after every load | Enables per-model LOD and frustum culling |
| Use `world.camera.controls.fitToSphere(sphere, true)` | Camera-controls wraps Three — all camera ops go through `controls` |
| 6-char hex only: `#ff0000` | 8-char (`#ff000080`) warns and leaves color unchanged — THREE.Color has no alpha |
| Alpha via material: `material.transparent = true; material.opacity = 0.5` | THREE.Color ignores alpha channel entirely |
| `fragments.core.update(true)` after visibility changes | `setVisible()` queues in worker — forced update renders immediately |

### DON'T

| Anti-Pattern | What Happens |
|--------------|--------------|
| `scene.remove(model.object)` without `disposeModel()` | Fragment worker memory leaks on every model swap |
| `world.camera.three.position.set(...)` | No effect — camera-controls overrides. Use `controls.setPosition()` |
| `new THREE.Vector3()` inside animation loops | GC pressure → frame stutters. Declare once, reuse `.set()` |
| `Box3.setFromObject(fragmentRoot)` for camera framing | Unreliable on InstancedMesh. Use `controls.fitToSphere()` |
| Load raw IFC on every page visit (production) | CPU/memory-heavy. Pre-convert to `.frag` binary (10x faster) |
| Enable COOP/COEP for MT WASM | Breaks OAuth popups, analytics. Not worth it for course-scale models |
| `scene.clear()` to "clean up" | Removes from graph, does NOT free GPU memory |

## Feature Wiring Patterns

All features connect through the **viewerSlice** in Zustand. The pattern is always:
`UI event → store action → useEffect in BIMViewer reacts → ThatOpen API call`

### Click → Selection → Properties Panel

```
Canvas click → fragments.raycast() → getGuidsByLocalIds()
            → selectElements([guid])          // writes to store
            → selectedIds in Zustand           // any component can read
            → PropertiesPanel reads selectedIds, calls getItemsData()
```

`selectedIds` is a `Set<string>` of GlobalIds. Wire a `useEffect` to highlight selected
elements (blue overlay) and fetch properties for the sidebar.

### Check Results → Color Coding

```
HF Space POSTs results → checksSlice.elementResults populated
                       → useViewer() computes colorMap (fail=#e62020, pass=#d0d3da)
                       → BIMViewer applyColors() via FragmentsManager.highlight()
```

### Tooltip on Hover (add to BIMViewer)

Wire `pointermove` → `highlighter.highlight("hover")` → read hovered element
→ show positioned `<div>`. **Must manually call highlight on pointermove** — hover
is NOT auto-triggered. Clean up tooltip DOM element in `useEffect` return.

### Hide/Show Elements

```ts
hideElements(wallGuids)  → hiddenIds in Zustand
                         → useEffect calls model.setVisible(ids, false)
                         → fragments.core.update(true)
showAll()                → clears hiddenIds → restores all visibility
```

`hiddenIds` is NOT wired in BIMViewer yet — add a `useEffect` if needed.

## ThatOpen API Quick Reference

All interaction uses `@thatopen/components` + `@thatopen/components-front`.
Key classes: `OBCF.Highlighter`, `OBC.Raycasters`, `OBC.FragmentsManager`.

### Highlighter Setup

Init `Raycasters` before `Highlighter`. Call `highlighter.setup({ world })`.
Events: `highlighter.events.select.onHighlight/onClear`, `highlighter.events.hover.*`.
Callback receives `modelIdMap: Record<string, number[]>` (modelId → localIds).

### Getting Element Data from localIds

`model.getItemsData([localId])` — returns `[{ GlobalId, Name, type, ... }]`.
For property sets: pass `{ relations: { IsDefinedBy: { attributes: true, relations: true } } }`.
PSets nested under `item.IsDefinedBy[].HasProperties[].NominalValue.value`.

### Programmatic Highlight

`fragments.guidsToModelIdMap(guids)` → `ModelIdMap` (`Record<string, number[]>`).
`highlighter.highlightByID("select", modelIdMap, true)` to highlight.
`highlighter.clear("select")` to clear.
**Highlight styles use `MaterialDefinition`** — `color` must be `THREE.Color`, NOT raw `{ r, g, b }`.

### Key Gotchas

- **Hover is NOT automatic** — must call `highlighter.highlight("hover")` in your own `pointermove` listener
- **`getItemsData` takes an array** — `[localId]` not `localId`
- **`localId` vs `GlobalId`** — localId = engine int, GlobalId = IFC GUID string. Convert via `getItemsData()` or `guidsToModelIdMap()`
- **PSets need explicit request** — without `relations: { IsDefinedBy }` you only get Name/GlobalId/type
- **Clean up DOM** — tooltips appended to `document.body` leak on unmount. Remove in `useEffect` return
- **Don't write to store inside `applyColors`** — `useStore.getState()` reads are safe (snapshot), but `setColorMap()` inside would loop
- **`selectedIds`/`hiddenIds` not fully wired** — BIMViewer reacts to `ifcUrl` and `colorMap`. Wire hide/show via `useEffect` if needed

## Extending the Viewer

Students/captains build on the viewerSlice API. All methods via `useStore.getState()`:

| Method | Effect |
|--------|--------|
| `selectElements(["guid1", ...])` | Sets `selectedIds` → highlight blue in viewer |
| `setColorMap({ "guid": "#ff9900" })` | Custom color coding (heatmap, markup) |
| `hideElements(wallGuids)` | Adds to `hiddenIds` → wire `useEffect` for `setVisible()` |
| `showAll()` | Clears `hiddenIds` → restores all visibility |
