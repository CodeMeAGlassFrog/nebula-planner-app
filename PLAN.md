# Nebula Planner: Project Plan

A space-themed infinite canvas for projects, plans, ideas, and images. Detail is revealed by zooming, the way you move from galaxy to planet to surface.

Status: planning. Last updated: 27 Oct 2026.

---

## 1. Vision

One canvas where every project is a star, every idea is a moon around it, and images are glowing windows. You pan and zoom through it. Zoomed out you see the shape of everything you are working on. Zoomed in you see the full detail of one thing.

The space theme is behaviour, not just paint. If it were only a skin, an Obsidian Canvas CSS snippet would do the job. The thing worth building is **semantic zoom**: the amount of detail shown depends on how far in you are.

## 2. Goals and non-goals

**Goals**

- Pan and zoom canvas with draggable nodes and connections
- Three node types: project, idea, image
- Semantic zoom (three detail levels)
- Paste or drag-drop images straight onto the canvas
- Local-first persistence, with JSON export and import as backup
- A distinct space look that stays smooth at a few hundred nodes

**Non-goals (for v1)**

- Accounts, backend, multi-user sync, real-time collaboration
- Mobile-first layout
- Rich text editing beyond plain text and basic markdown
- True orbital animation (animated positions fight with dragging)

## 3. Metaphor mapping

| Concept | Space object | Visual treatment |
|---|---|---|
| Project | Star or planet | Glowing radial gradient, size scales with number of children |
| Idea | Moon or satellite | Small body near its project, gentle float animation |
| Image | Window or nebula plate | Soft glow border, rounded corners |
| Connection | Constellation line | Thin, faint, low opacity |
| Group of projects | Galaxy region | Label that appears only when far zoomed out |

## 4. Tech stack

| Choice | Why |
|---|---|
| Vite + React + TypeScript | Fast dev loop, good VS Code support, types help with the node data model |
| `@xyflow/react` (React Flow) | Handles pan, zoom, dragging, edges, custom node types, minimap. Saves building the hardest canvas mechanics |
| Zustand | Small global store, easy selectors, works well with React Flow |
| `idb-keyval` | Minimal IndexedDB wrapper. Images as blobs will outgrow localStorage quickly |
| Plain CSS with variables | Theming in one place, no extra dependency |

Check each library's docs for the current version and Node requirements before installing (links in section 14).

## 5. Data model (starting point)

```ts
type NodeKind = 'project' | 'idea' | 'image';

type NebulaNodeData = {
  kind: NodeKind;
  title: string;
  body?: string;          // plain text or markdown
  imageId?: string;       // key into IndexedDB blob store
  collapsed?: boolean;    // project only: hide child moons
  pinned?: boolean;       // force full detail regardless of zoom
  tags?: string[];
  color?: string;         // accent override
};

type MapFile = {
  version: 1;
  nodes: Node<NebulaNodeData>[];
  edges: Edge[];
  viewport: { x: number; y: number; zoom: number };
};
```

Notes:

- Keep a `version` field from day one so you can migrate saved data later.
- Images live in IndexedDB keyed by `imageId`. The node only stores the key.
- Moons are React Flow child nodes of a project (`parentId`). Child positions are relative to the parent, and dragging the project moves its moons.

## 6. Folder layout

```
Nebula Planner App/
  PLAN.md
  index.html
  package.json
  src/
    main.tsx
    App.tsx
    nodes/
      ProjectNode.tsx
      IdeaNode.tsx
      ImageNode.tsx
      index.ts            // exports nodeTypes
    components/
      Starfield.tsx       // background canvas
      Toolbar.tsx
      SearchPalette.tsx
    store/
      useMapStore.ts
    storage/
      db.ts               // idb-keyval wrappers
      exportImport.ts
    hooks/
      useZoomLevel.ts
    styles/
      tokens.css          // colour, type, spacing variables
      nodes.css
```

## 7. Semantic zoom spec

Three detail levels driven by the viewport zoom value.

| Level | Zoom range | Project | Idea | Image |
|---|---|---|---|---|
| Far | below 0.4 | Glowing star + name | Hidden | Hidden |
| Mid | 0.4 to 0.9 | Star + name + child count | Small moon, title only | Small thumbnail |
| Near | above 0.9 | Full card with body text | Full text | Full image + caption |

Implementation sketch:

```ts
// hooks/useZoomLevel.ts
import { useStore } from '@xyflow/react';

export type ZoomLevel = 'far' | 'mid' | 'near';

export function useZoomLevel(): ZoomLevel {
  // Selector returns a string, so components only re-render when the level
  // changes, not on every zoom tick.
  return useStore((s) => {
    const z = s.transform[2];
    return z < 0.4 ? 'far' : z < 0.9 ? 'mid' : 'near';
  });
}
```

Rules:

- Cross-fade between levels with opacity transitions (150 to 250 ms). Avoid hard swaps.
- Add a small hysteresis band around thresholds so detail does not flicker when zoom sits right on a boundary.
- `pinned` nodes ignore the zoom level and always render full detail.
- Hidden moons should use the node `hidden` flag or `display: none` after the fade, so they cost nothing.

## 8. Visual design system

**Palette (CSS variables in `tokens.css`)**

```css
:root {
  --space-bg: #05070f;
  --space-surface: #0c1020;
  --accent-violet: #8b5cf6;
  --accent-cyan: #22d3ee;
  --text-primary: #f3efe6;   /* warm white */
  --text-muted: #8a90a6;
  --line-faint: rgba(255, 255, 255, 0.12);
}
```

**Type**

- Space Grotesk for body and UI
- Orbitron only for large project titles, if at all. It tires quickly at small sizes

**Starfield background**

- A single `<canvas>` behind React Flow
- 2 or 3 star layers with different parallax factors (for example 0.1, 0.25, 0.5 of pan offset)
- Stars generated once from a seeded random number generator and wrapped with modulo so the field tiles infinitely
- Slow twinkle by varying alpha per star
- Handle `devicePixelRatio` so it stays sharp on high-DPI screens
- Read pan offsets from the viewport (`useViewport` or `useOnViewportChange`) rather than from DOM events

**Glow**

- Fake it with `radial-gradient` and a few layered `box-shadow`s
- Reserve real `filter: blur()` for one or two elements at most

**Motion**

- Subtle CSS float animation on moons (a few pixels, long duration)
- No JS-driven orbiting

## 9. Build phases

Commit to git at the end of each phase.

### Phase 0: Setup (small)

- [ ] `git init` in the `Nebula Planner App` folder
- [ ] `npm create vite@latest . -- --template react-ts`
- [ ] `npm i @xyflow/react zustand idb-keyval`
- [ ] Import `@xyflow/react/dist/style.css` once in `main.tsx`
- [ ] Add `.gitignore` entries for `node_modules` and `dist`

**Done when:** `npm run dev` shows an empty pannable, zoomable React Flow canvas.

### Phase 1: Look first (medium)

- [ ] `tokens.css` with palette and fonts
- [ ] `Starfield.tsx` with parallax layers and twinkle
- [ ] Transparent React Flow background so stars show through
- [ ] Restyle default edges as faint constellation lines

**Done when:** panning feels like moving through space, and it holds 60 fps on your laptop.

### Phase 2: Node types (medium)

- [ ] `ProjectNode`, `IdeaNode`, `ImageNode` registered in `nodeTypes` (defined outside the component)
- [ ] Double-click on empty canvas opens a small menu to add a node
- [ ] Inline editing of title and body
- [ ] Paste or drag-drop an image onto the canvas to create an image node (downscale large images before storing)
- [ ] Add a moon to a project via `parentId`

**Done when:** you can build a small map of one project, a few ideas, and an image by hand.

### Phase 3: Semantic zoom (medium)

- [ ] `useZoomLevel` hook
- [ ] Each node component renders three detail variants with cross-fades
- [ ] `pinned` toggle on any node
- [ ] Collapse and expand a project's moons
- [ ] Project size scales with child count

**Done when:** zooming from far to near feels continuous, with no flicker and no popping.

### Phase 4: Persistence (medium)

- [ ] Zustand store holds nodes, edges, viewport
- [ ] Debounced autosave of the map to IndexedDB
- [ ] Image blobs stored separately by `imageId`, shown via `URL.createObjectURL` and revoked on unmount
- [ ] JSON export and import (decide how to handle images: base64 inline, or a zip)
- [ ] Load on startup and restore viewport

**Done when:** you can close the tab, reopen it, and be exactly where you left off, and an exported file restores a map on a fresh browser profile.

### Phase 5: Polish (open-ended)

- [ ] Minimap restyled as a radar
- [ ] Search palette (`Ctrl+K`) that flies the camera to a node
- [ ] Keyboard shortcuts (new node, delete, pin, fit view)
- [ ] Tags and colour accents
- [ ] Undo and redo
- [ ] Galaxy region labels at far zoom
- [ ] Optional: wrap in Tauri or Electron later if you want it as a desktop app

## 10. Performance notes

- Draw stars once on one canvas. Never as DOM elements.
- Avoid `filter: blur()` and heavy shadows on many nodes.
- Use `onlyRenderVisibleElements` on the React Flow component once you pass roughly 100 nodes.
- Memoise node components with `React.memo` and keep Zustand selectors narrow.
- Define `nodeTypes` and `edgeTypes` outside the component body, or React Flow will remount nodes on every render.
- Downscale imported images to a sensible maximum edge length before storing.

## 11. Risks and gotchas

| Risk | Mitigation |
|---|---|
| Child node positions are relative to the parent, which confuses export and layout code | Keep coordinates in React Flow's native format and never convert manually |
| Parent nodes must come before children in the nodes array | Sort on load and on insert |
| Zoom-driven re-renders cause jank | Selectors that return a discrete level, not the raw zoom number |
| IndexedDB can be cleared by the browser under storage pressure | Regular JSON export reminder, and request persistent storage via `navigator.storage.persist()` |
| Scope creep into a full note-taking app | Stick to the phases. Anything not in them goes in section 13 |

## 12. Open questions

These shape the design, so answer them before Phase 3.

1. Roughly how many nodes will a typical map hold: tens, or hundreds? This decides whether you need virtualisation early.
2. Is this mainly a learning project, or something you will use daily for planning? Daily use raises the bar on persistence, backup, and undo.
3. Should moons be free-floating anywhere on the canvas, or constrained near their project?
4. Do you want connections between any two nodes, or only parent to child?
5. Should the map stay purely local, or will you want to open it on another device at some point?

## 13. Parking lot (not in v1)

- Sync across devices
- Real orbital animation for decorative bodies only (not draggable ones)
- Shared maps or read-only share links
- Markdown rendering inside nodes
- Timeline or deadline layer
- Import from Obsidian Canvas (the JSON Canvas format is open and documented)
- Sound design: very quiet ambient hum that changes with zoom

## 14. References

**Core libraries**

- React Flow docs: https://reactflow.dev
  - Learn: https://reactflow.dev/learn
  - Custom nodes: https://reactflow.dev/learn/customization/custom-nodes
  - Sub flows (`parentId`): https://reactflow.dev/learn/layouting/sub-flows
  - Performance guide: https://reactflow.dev/learn/advanced-use/performance
  - Examples: https://reactflow.dev/examples
- Vite: https://vite.dev
- Zustand: https://zustand.docs.pmnd.rs
- idb-keyval: https://github.com/jakearchibald/idb-keyval
- React docs: https://react.dev

**Browser APIs**

- Canvas API (MDN): https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API
- IndexedDB (MDN): https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
- Storage persistence (MDN): https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/persist
- `createImageBitmap` (MDN): https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap
- Clipboard and drag-drop for images (MDN): https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API

**Design and fonts**

- Space Grotesk (Google Fonts): https://fonts.google.com/specimen/Space+Grotesk
- Orbitron (Google Fonts): https://fonts.google.com/specimen/Orbitron

**Prior art worth a look**

- Obsidian Canvas and the open JSON Canvas spec: https://jsoncanvas.org
- Excalidraw: https://excalidraw.com
- tldraw: https://tldraw.dev
- Milanote: https://milanote.com

**Suggested VS Code extensions**

- ESLint, Prettier, ES7+ React snippets, and the built-in TypeScript tooling
