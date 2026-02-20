# 3D Viewer — IFC Direct Rendering

The platform renders IFC files directly in the browser using `@thatopen/components` v3.
No backend GLB conversion. The backend only runs checks.

```
Upload IFC → R2 → Frontend loads IFC directly via @thatopen/components
                   HF Space only runs checks, frontend polls for results
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

**web-ifc is Emscripten-compiled WASM. It CANNOT be processed by JavaScript bundlers.**
Two complementary patterns are used:

### Pattern 1: IIFE + Shim (for the JS API)

The bare `import { IfcAPI } from 'web-ifc'` is intercepted by a Vite alias:

```ts
// vite.config.ts — regex alias intercepts ONLY bare "web-ifc" imports
resolve: {
  alias: [
    { find: /^web-ifc$/, replacement: path.resolve(__dirname, "src/lib/web-ifc-shim.ts") },
  ],
},
```

`index.html` loads the IIFE before the app bundle:
```html
<script src="/web-ifc-api-iife.js"></script>
```

`web-ifc-shim.ts` re-exports from the global:
```ts
const W = (globalThis as any).WebIFC;
export default W;
export const IfcAPI = W.IfcAPI;
// ... all named exports
```

**CRITICAL:** Use regex `/^web-ifc$/` — NOT a string `"web-ifc"`. A string alias also matches
subpath imports like `web-ifc/web-ifc.wasm?url`, breaking WASM path resolution.

### Pattern 2: `?url` imports (for WASM binaries)

BIMViewer uses Vite `?url` imports for WASM file resolution:
```ts
import webIfcWasmUrl from "web-ifc/web-ifc.wasm?url";
import webIfcMtWasmUrl from "web-ifc/web-ifc-mt.wasm?url";

function resolveWasmUrl(fileName: string) {
  const normalized = fileName.split("?")[0].toLowerCase();
  return new URL(normalized.includes("mt") ? webIfcMtWasmUrl : webIfcWasmUrl, window.location.origin).toString();
}
```

This is passed to `ifcLoader.setup({ customLocateFileHandler: resolveWasmUrl })`.
The regex alias doesn't match these subpath imports, so they resolve normally.

### Multi-threading guard

CF Pages doesn't set COOP/COEP headers needed for SharedArrayBuffer:
```ts
if (typeof globalThis !== "undefined" && !globalThis.crossOriginIsolated) {
  Object.defineProperty(globalThis, "crossOriginIsolated", { value: false, writable: false });
}
```
This forces single-threaded WASM (uses `web-ifc.wasm`, not `web-ifc-mt.wasm`).

### Static Files in `public/`

Copy from `node_modules/web-ifc/` after install:

| File | Purpose |
|------|---------|
| `web-ifc-api-iife.js` | Pre-built IIFE (JS API) |
| `web-ifc.wasm` | Single-threaded WASM binary |
| `web-ifc-mt.wasm` | Multi-threaded WASM (needs COOP/COEP) |
| `fragments-worker.mjs` | FragmentsManager Web Worker |

## Viewer Architecture

### Zustand State (viewerSlice)

```ts
ifcUrl: string | null           // URL to fetch IFC from (set by route)
viewerVisible: boolean          // whether viewer panel is shown
colorMap: Record<string, string>       // GlobalId → hex (base check status colors)
highlightColorMap: Record<string, string> // GlobalId → hex (category filter overlay)
selectedIds: Set<string>        // GlobalIds currently selected (clicked)
hiddenIds: Set<string>          // GlobalIds hidden from view
isReady: boolean                // true when ThatOpen engine initialized
```

**Dual colorMap system:**
- `colorMap` is set by `useViewer` (check status → color for all elements)
- `highlightColorMap` is set by `useCategoryColors` (category filter → overlay)
- BIMViewer merges: `{ ...colorMap, ...highlightColorMap }` — highlight takes priority

### BIMViewer.tsx

Renders ThatOpen `Components` engine in a `<div>`. Key lifecycle:

1. **ResizeObserver** — watches container dimensions, updates renderer + camera
2. **Init** (runs ONCE) — creates `Components`, `SimpleScene`, `SimpleRenderer`, `OrthoPerspectiveCamera`,
   sets up `FragmentsManager.init()`, `IfcLoader.setup()` with custom WASM locator, click raycasting
3. **Load** — when `ifcUrl` changes, fetches IFC via `fetch()`, validates not HTML, runs `ifcLoader.load()`
4. **GUID map** — builds `GlobalId → expressID` map via webIfc or model properties
5. **Color** — `applyColors()` merges both colorMaps, uses `fragments.highlight()` (v3 API)
6. **rAF camera coalescing** — camera control updates are coalesced to one GPU update per frame
   via `requestAnimationFrame` to prevent excessive `fragments.core.update()` calls
7. **Cleanup** — `components.dispose()` on unmount, cancels animation frames

### useViewer.ts — Check Results → Color Map

Maps `elementResults` to `colorMap` with **5 status colors** and **fail-priority**:

```ts
const STATUS_HEX: Record<string, string> = {
  fail: "#e62020",      // red, full opacity
  warning: "#f59e0b",   // amber
  pass: "#22c55e",      // green
  blocked: "#6b7280",   // gray
  log: "#9ca8c9",       // light gray
};
```

If the same element has multiple results, `fail` takes priority over other statuses.

**Performance:** Uses `getState()` with equality check before writing — prevents
re-render cascades when colorMap hasn't actually changed.

### useCategoryColors.ts — Category Filter → Highlight Map

When a category is selected in CategoryCards:
- Elements with **fail** status in that category → highlighted with the category color
- All other elements → gray (`#d0d3da`)
- When no category selected → clears `highlightColorMap`

Uses `getState()` with equality check, proper cleanup on unmount.

### viewerActions.ts — Programmatic Operations

Extracted helper functions for viewer operations:
```ts
guidsToModelIdMap(components, guids)     // GlobalIds → ModelIdMap
resetHighlights(components, map?)        // clear all/specific highlights
highlightGuids(components, guids, hex, opacity?)  // apply color to GUIDs
hideGuids(components, guids)             // hide elements
showGuids(components, guids)             // show hidden elements
isolateGuids(components, guids)          // hide everything except these
```

### viewerDiagnostics.ts — Phase Tracking

Tracks viewer lifecycle phases for debugging:
`idle → init-engine → init-loader → fetch-ifc → process-ifc → render-model → apply-colors → ready`

Error classification: `wasm | ifc-fetch | ifc-payload | fragments | render | unknown`
Enable debug mode: `localStorage.setItem("viewerDebug", "1")`

### ElementTooltip.tsx — Click → Element Info

Shows check results for the clicked element as an overlay in the viewer:
- Reads `selectedIds` from store
- Finds matching `elementResults` by `element_id`
- Displays status dot, actual/required values, comment
- Close button clears selection

## Data Flow

```
Upload IFC → R2 → route sets ifcUrl
                   → BIMViewer fetches IFC, renders in ThatOpen
                   → builds GlobalId → expressID map

Run checks → HF Space processes → CF Worker stores in D1
          → poller updates checksSlice (batched setState)
          → useViewer computes colorMap (5 statuses, fail priority)
          → BIMViewer applyColors() merges colorMap + highlightColorMap
          → fragments.highlight() applies colors per GUID

Click category → useCategoryColors sets highlightColorMap
              → BIMViewer reacts, re-applies merged colors

Click element → raycasting → getGuidsByLocalIds → selectElements()
             → ElementTooltip shows results for that GUID
```

## Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| "Import #0 'a' module is not an object" | Bundler processed web-ifc | Use IIFE + shim pattern |
| "You need to initialize fragments first" | `FragmentsManager.init()` not called | Call before loading |
| colorMap computed before model loads | Race condition | `applyColors()` after load AND on colorMap change |
| Frame stutters on camera move | `fragments.core.update()` called too often | Use rAF coalescing |
| Re-render cascade on poll | poller writes to store 3 times | Batch into single `setState()` |
| Category colors flicker | useViewer + useCategoryColors ping-pong | `getState()` with equality check |

## Best Practices

### DO
- `fragments.core.disposeModel(modelId)` to remove models (not `scene.remove`)
- `ifcLoader.setup()` once in init effect
- Camera updates via `controls.addEventListener("update", ...)` with rAF coalescing
- `model.useCamera(world.camera.three)` after every load (enables LOD)
- `world.camera.controls.fitToSphere(sphere, true)` for camera framing
- 6-char hex only: `#ff0000` (THREE.Color has no alpha)
- `fragments.core.update(true)` after visibility changes

### DON'T
- `scene.remove(model.object)` without `disposeModel()` (memory leak)
- `world.camera.three.position.set(...)` (camera-controls overrides — use `controls.setPosition()`)
- `new THREE.Vector3()` inside animation loops (GC pressure)
- Enable COOP/COEP for MT WASM (breaks OAuth popups)
- `scene.clear()` to "clean up" (doesn't free GPU memory)
- Write to store inside `applyColors` (re-render loop)
