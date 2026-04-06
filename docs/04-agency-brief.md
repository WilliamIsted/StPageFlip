# Page-Flip Library - Agency Brief

## 1. Project Overview

We require a zero-dependency TypeScript library that produces realistic page-turning effects for web-based book and magazine viewers. The library will transform a flat collection of content (either images or HTML elements) into an interactive two-page spread with drag-to-flip, click-to-turn, and swipe gesture support. The visual effect should mimic a physical page curling away from the spine, complete with dynamic drop shadows.

A reference implementation exists (StPageFlip v2.0.7, MIT licence) and is documented in the accompanying technical files (`docs/01-core-principles.md`, `docs/02-compiled-deliverable.md`, `docs/03-reverse-engineered-files.md`). The new library should achieve the same visual result and cover the same use cases, but be designed and built from scratch.

### Business Context

Target applications include digital publishing platforms, portfolio websites, product catalogues, photo books, comic readers, and e-reader interfaces. The library must work as a standalone drop-in for any web project, with no framework dependency.

---

## 2. Project Goals

| Priority | Goal |
|----------|------|
| **Primary** | Produce a zero-dependency TypeScript library that renders realistic page-flip animations in the browser |
| **Secondary** | Support both image-based (Canvas) and DOM-based (HTML/CSS) content rendering |
| **Quality** | Match or exceed the visual fidelity of the reference implementation |
| **Performance** | Smooth 60fps animations on modern mid-range devices |
| **Maintainability** | Clean, well-documented, fully tested codebase suitable for long-term maintenance |

---

## 3. Target Users and Use Cases

**Direct Integration Developers:** Front-end developers who will include the library via a `<script>` tag (UMD) or `import` statement (ES Module) in vanilla JavaScript or TypeScript projects.

**Framework Integrators:** Developers building React, Vue, or Angular wrappers around the library. The library must not impose any framework dependency.

**Content Types:**
- Photo books and scanned documents (Canvas mode)
- Rich HTML content with interactive elements -- links, buttons, forms (HTML mode)
- Product catalogues and digital magazines
- Portfolio and presentation viewers
- Comics and graphic novels

---

## 4. Functional Requirements

### 4.1 Dual Rendering Modes

The library shall support two rendering approaches behind a single, unified API:

- **Canvas Mode:** Pages are images drawn onto an HTML5 Canvas. All clipping, rotation, and shadow effects are computed and painted via the Canvas 2D API.
- **HTML Mode:** Pages are DOM elements positioned and transformed using CSS `transform`, `clip-path`, and `z-index`. Shadows are rendered as separate `<div>` elements with CSS gradients. DOM interactivity (links, buttons, forms) must be preserved.

### 4.2 Page-Flip Interaction Model

The library shall support four interaction methods:

| Method | Description |
|--------|-------------|
| **Drag** | User presses on the page and drags the corner to flip. The page follows the pointer position. On release, the page either completes the flip or returns to its original position based on how far it was dragged. |
| **Click** | User clicks on the book. A full page-flip animation plays automatically. Optionally restricted to corner areas only. |
| **Swipe** | On touch devices, a fast horizontal swipe (configurable distance threshold, within a time window) triggers an animated flip. |
| **Programmatic** | API methods trigger flips with or without animation. |

An optional corner fold preview shall appear when the mouse hovers near a page corner.

### 4.3 Responsive Layout

The library shall support two sizing modes:

- **Fixed:** Book dimensions match the configured width/height exactly.
- **Stretch:** Book scales to fill its parent container while maintaining aspect ratio, bounded by min/max constraints.

The library shall automatically switch between landscape (two-page spread) and portrait (single-page) mode based on available container width. Orientation changes shall trigger an event.

### 4.4 Page Types

Each page shall have a density property:

- **Soft:** Pages curl and bend realistically. The flip is rendered using rotation, clipping polygons, and curved shadows.
- **Hard:** Pages rotate rigidly around the spine edge (like a book cover), using 3D CSS rotation in HTML mode or angle-based drawing in Canvas mode.

Cover pages (first and last, when enabled) shall be automatically set to hard density. In HTML mode, per-page density shall be configurable via a data attribute on the element.

### 4.5 Visual Effects

The shadow system shall comprise multiple layers:

| Shadow | Description |
|--------|-------------|
| **Book spine shadow** | A permanent vertical gradient at the centre of the book |
| **Outer shadow** | Projects from the fold line onto the underlying page, scaling with flip progress |
| **Inner shadow** | Projects onto the flipping page's surface from the fold line |
| **Hard page shadows** | Simplified shadows for rigid page rotation |

Shadow rendering shall be optional (configurable), with adjustable maximum opacity.

### 4.6 Spread-Based Navigation

- **Landscape mode:** Two-page spreads (left + right), with optional single-page cover spreads.
- **Portrait mode:** Single-page spreads.
- Navigation, rendering, and flipping shall operate on spread indices, correctly handling cover pages and the relationship between flipping and revealed pages.

### 4.7 Event System

The library shall notify consumers when significant state changes occur. At a minimum, it must be possible to observe:

- When the library has finished initialising
- When the displayed page changes
- When the flip lifecycle state changes (e.g., idle, user dragging, animating)
- When the book orientation switches between portrait and landscape
- When the page collection is replaced

Consumers shall be able to subscribe and unsubscribe from events. The subscribe method should support chaining.

### 4.8 Programmatic API

The public API shall expose methods covering these capabilities:

- **Lifecycle:** Initialise with images or HTML content, replace pages at runtime, tear down and clean up.
- **Navigation (instant):** Jump to next, previous, or a specific page without animation.
- **Navigation (animated):** Trigger an animated flip to next, previous, or a specific page.
- **State Queries:** Retrieve page count, current page index, current orientation, book dimensions, current flip state, and configuration.

---

## 5. Technical Requirements

### 5.1 Language and Tooling

- **TypeScript** with full type declarations (`.d.ts` files).
- Strict mode compatible.
- Zero runtime dependencies.

### 5.2 Build Output

| Format | File | Purpose |
|--------|------|---------|
| **UMD** | `page-flip.browser.js` | Script tag, CommonJS, AMD |
| **ES Module** | `page-flip.module.js` | Modern bundlers (Webpack, Rollup, Vite) |

Both bundles shall be minified. CSS shall be inlined into the JavaScript bundle.

### 5.3 Browser Support

- Chrome, Firefox, Safari, Edge (latest two major versions).
- Mobile Safari and Chrome on iOS/Android.
- Known workaround required for Safari's CSS `clip-path` rendering bug (WebKit bug #126207).

### 5.4 Mobile Support

- Touch events with swipe detection.
- Configurable coexistence with page scrolling (horizontal gestures trigger flipping, vertical gestures allow native scroll).

### 5.5 Performance

- Animations must target 60fps on modern mid-range devices.
- The rendering approach should leverage hardware acceleration where available.

### 5.6 Architecture

- A single entry-point class must be the only public API surface.
- The flip geometry engine must be independent of the rendering backend, so that the same calculations drive both Canvas and HTML output.
- The flip lifecycle must be managed to prevent conflicting interactions (e.g., starting a new flip while one is animating).

---

## 6. Visual and Behavioural Requirements

The following describe the expected visual result and interaction behaviour, not a prescribed implementation. The reference implementation's approach is documented in `docs/03-reverse-engineered-files.md` for study, but the implementer is free to achieve the same effect by any means.

### 6.1 Flip Geometry

- When a user drags a page corner, the page must visually curl as if it were a physical sheet of paper being folded. The rotation and curvature must respond continuously to the pointer position.
- The page must not stretch beyond its physical dimensions -- a corner dragged far from the spine should hit a natural limit rather than distorting.
- The opposite edge of the page must not cross the spine during a flip.
- Both the flipping page and the page revealed beneath must be correctly clipped so they do not overlap or extend beyond the book boundaries.

### 6.2 Coordinate Handling

- User input arrives in browser viewport coordinates. The library must correctly translate these to book-relative and page-relative positions, accounting for the book's position on screen and the current flip direction.

### 6.3 Flip Lifecycle

- The library must manage its flip state to prevent conflicting interactions. At minimum, it must distinguish between: idle, corner hover preview, user actively dragging, and animation in progress.
- A new flip must not begin while an animation is playing (the existing animation should complete or be skipped first).

### 6.4 Animation

- Animated page flips should appear smooth and natural, with duration proportional to the distance the page needs to travel.
- Animation duration should be configurable, with a sensible default.

---

## 7. Configuration Surface

The library shall accept a configuration object at construction time. The configuration must cover the following areas, with sensible defaults for all optional values:

**Dimensions and Sizing:**
- Base page width and height (required).
- Sizing mode: fixed dimensions or stretch-to-fit parent container.
- Min/max bounds for width and height in stretch mode.

**Behaviour:**
- Starting page number.
- Animation duration.
- Whether to allow portrait mode.
- Whether first/last pages are treated as hard covers.
- Whether clicking anywhere flips, or only clicking on corners.
- Whether to show a corner fold preview on mouse hover.

**Visual:**
- Whether to render drop shadows.
- Shadow intensity.
- Base z-index for page stacking.

**Input:**
- Whether mouse/touch events are enabled.
- Swipe detection sensitivity (distance threshold).
- Whether vertical scrolling is allowed on mobile alongside page flipping.
- Whether clicks on interactive elements (links, buttons) inside pages are forwarded rather than captured as flips.

**Layout:**
- Whether the parent element should auto-size to the book dimensions.

Validation shall reject clearly invalid inputs (e.g., non-positive dimensions, unrecognised sizing modes).

---

## 8. Deliverables

| # | Deliverable | Description |
|---|-------------|-------------|
| 1 | **Source code** | TypeScript, modular architecture matching the specified subsystems |
| 2 | **Built bundles** | UMD and ES Module, minified, with bundled CSS |
| 3 | **Type declarations** | `.d.ts` files for TypeScript consumers |
| 4 | **Test suite** | Unit tests for utilities, calculations, state machine; integration tests for rendering and interaction |
| 5 | **API documentation** | Complete reference for the public API |
| 6 | **Usage examples** | Working examples for both Canvas and HTML modes |

---

## 9. Success Criteria

1. **Visual quality:** Page-flip animation, shadow rendering, and corner fold preview produce a convincing physical page-turn effect across all supported browsers.
2. **Functional completeness:** All capabilities described in Section 4 are implemented and working.
3. **Performance:** 60fps animation on mid-range devices (tested on throttled Chrome DevTools "mid-tier mobile" profile).
4. **Zero dependencies:** `npm install` adds no transitive runtime dependencies.
5. **Cross-browser:** All interactions work correctly on Chrome, Firefox, Safari, Edge, mobile Safari, and mobile Chrome.
6. **Test coverage:** Core calculation and state management logic covered by automated tests.
7. **Accessibility:** DOM content remains accessible in HTML mode (links clickable, text selectable when not flipping).

---

## 10. Out of Scope

- Server-side rendering (SSR) support.
- PDF parsing, loading, or rendering.
- Page content authoring or editing tools.
- Framework-specific wrapper components (React, Vue, Angular) -- these are separate downstream projects.
- Print stylesheets.
- Right-to-left (RTL) page ordering (may be considered as a future enhancement).
- WebGL or WebGPU rendering backend.
- Accessibility of Canvas-rendered content (inherent Canvas limitation).

---

## 11. Reference Materials

- `docs/01-core-principles.md` -- Architectural principles and design patterns.
- `docs/02-compiled-deliverable.md` -- Consumer-facing behaviour and rendering pipelines.
- `docs/03-reverse-engineered-files.md` -- File-by-file implementation analysis.
- Original library: StPageFlip v2.0.7 (MIT licence, authored by Oleg Litovski / Nodlik).
