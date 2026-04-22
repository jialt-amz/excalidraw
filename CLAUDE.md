# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Structure

Excalidraw is a **monorepo** with a clear separation between the core library and the application:

- **`packages/excalidraw/`** - Main React component library published to npm as `@excalidraw/excalidraw`
- **`excalidraw-app/`** - Full-featured web application (excalidraw.com) that uses the library
- **`packages/common`**, **`packages/element`**, **`packages/math`**, **`packages/utils`** - Core packages
- **`examples/`** - Integration examples (NextJS, browser script)

### Package Dependency Order

```
@excalidraw/math
    └── @excalidraw/common
            └── @excalidraw/element
                    └── @excalidraw/utils
                                └── @excalidraw/excalidraw (imports all four)
```

## Development Commands

```bash
yarn start               # Run the web app (excalidraw-app)
yarn build               # Build the web app
yarn build:packages      # Build all packages (common → math → element → excalidraw)

yarn test:app            # Run tests (watch mode)
yarn test:app --watch=false                    # Run tests once
yarn test:app --watch=false path/to/test.tsx   # Run a single test file
yarn test:app -t "test name"                   # Run tests matching a name
yarn test:update         # Run tests and update snapshots
yarn test:typecheck      # TypeScript type checking (tsc)
yarn test:code           # ESLint (max-warnings=0)
yarn fix                 # Auto-fix formatting and linting
```

## Architecture

### App.tsx — The Core Editor

`packages/excalidraw/components/App.tsx` is a **class component** (~12,800 lines) and the heart of the editor. Its constructor initializes: `Scene`, `ActionManager`, `Renderer`, `Library`, `Store`, `History`, `Fonts`, `RoughCanvas`. Its `render()` outputs a context-provider tree wrapping `<LayerUI>` and multiple canvas layers.

### State Management

A hybrid approach:
1. **React class state** (`this.state: AppState`) — primary editor state (zoom, scroll, active tool, selected elements). Defined in `packages/excalidraw/types.ts`. Passed down via React context hooks (`useExcalidrawAppState()`, `useApp()`, etc.)
2. **Jotai atoms** — scoped UI state via an isolated Jotai store (`packages/excalidraw/editor-jotai.ts`). Used for sidebar docked state, library menu open, eye dropper active, etc. The store is isolated via `jotai-scope` so atoms don't leak to the host app.
3. **Tunnel-based rendering** — `TunnelsContext` (`context/tunnels.tsx`) using `tunnel-rat` portals for MainMenu, sidebar trigger, and overwrite dialogs to render in specific DOM locations without prop drilling.

### UI Layer — LayerUI.tsx

`packages/excalidraw/components/LayerUI.tsx` is the top-level UI component. Key sections:
- **`renderFixedSideContainer()`** — the main top bar: left (menu + shape actions), center (shapes toolbar `<Island>` with `ShapesSwitcher`/`PenModeButton`/`LockButton`), right (UserList, sidebar trigger)
- **`renderSelectedShapeActions()`** — the properties panel (`<Island>` with `SelectedShapeActions`)
- **Mobile fork** — renders `<MobileMenu>` when `editorInterface.formFactor === "phone"`

### Island Component

`packages/excalidraw/components/Island.tsx` is a `React.forwardRef` wrapper rendering `<div className="Island">`. It is the visual "panel/card" container used throughout the UI. Styling is driven by CSS variables defined in `packages/excalidraw/css/theme.scss`:
- `--island-bg-color` — background (white / `#232329` dark)
- `--shadow-island` — drop shadow (acts as visual border; removed in `zen-mode`)
- `--border-radius-lg` — corner radius
- `--space-factor` — padding scale (`calc(var(--padding) * var(--space-factor))`)

To override styles for a specific Island, use a combined selector e.g. `.Island.App-toolbar` in `Toolbar.scss`.

### Rendering Pipeline

- **`packages/element/src/renderElement.ts`** — per-element draw logic using `roughjs`
- **`packages/excalidraw/renderer/staticScene.ts`** — non-interactive canvas (grid, elements, frame clipping, dark-mode filter)
- **`packages/excalidraw/renderer/interactiveScene.ts`** — interactive overlay (selection boxes, transform handles, snap indicators, collaborator cursors)
- **`packages/excalidraw/renderer/staticSvgScene.ts`** — SVG export path
- **`packages/excalidraw/renderer/renderNewElementScene.ts`** — in-progress element being drawn

### Action System

`packages/excalidraw/actions/` — every action (align, clipboard, canvas ops, etc.) calls `register(action)` from `register.ts` at module load. Each `Action` has: `name: ActionName`, `perform: ActionFn` (receives elements, appState, formData, app → returns `ActionResult | false`), optional `PanelComponent`, `keyTest`, `predicate`. `ActionManager` (in `manager.tsx`) holds the registry and dispatches analytics.

### Element Types

All element types extend `_ExcalidrawElementBase` (defined in `packages/element/src/types.ts`), which has shared fields: `id`, `x/y`, `width/height`, `angle: Radians`, stroke/fill/style props, `index: FractionalIndex | null`, `groupIds`, `frameId`, `boundElements`. Factory functions in `packages/element/src/newElement.ts` (`newElement`, `newTextElement`, `newLinearElement`, etc.) construct elements.

When writing math-related code, always reference `packages/math/src/types.ts` and use the `Point` type instead of `{ x, y }`.

### CSS Architecture

- `packages/excalidraw/css/theme.scss` — all CSS custom properties (colors, shadows, spacing, z-indices for both light and dark mode)
- `packages/excalidraw/css/styles.scss` — z-index layers, root layout, `.excalidraw` container styles
- Component-scoped SCSS files live alongside components (e.g. `Toolbar.scss`, `Island.scss`)
- All component styles are scoped under `.excalidraw { }` to avoid leaking into host apps

### Testing Setup

Vitest with path aliases resolving all `@excalidraw/*` packages to their `src/index.ts` (see `vitest.config.mts`). Test files live in `packages/excalidraw/tests/`.

## TypeScript Guidelines

- Prefer implementations without allocation where possible
- Trade RAM for fewer CPU cycles in hot paths
- Prefer `const` / `readonly` immutable data
- Use `?.` optional chaining and `??` nullish coalescing
