# Page-Flip Library Rewrite - Scope of Work

## 1. Project Summary

This scope of work defines a from-scratch implementation of a page-flip library for the web. The new library will achieve the same visual effect and cover the same use cases as the reference implementation (StPageFlip v2.0.7) while delivering a clean, well-tested codebase with its own architecture.

No code from the original will be reused. The reference documentation provides context on the problem domain and desired behaviour:
- `docs/01-core-principles.md` -- Architectural principles
- `docs/02-compiled-deliverable.md` -- Consumer-facing behaviour
- `docs/03-reverse-engineered-files.md` -- File-level implementation detail
- `docs/04-agency-brief.md` -- Project requirements

---

## 2. Technical Specifications

### 2.1 Language and Tooling

- **Language:** TypeScript (strict mode).
- **Build output:** Minified UMD and ES Module JavaScript bundles, with CSS inlined and `.d.ts` type declarations.
- **Linting and testing:** Implementer's choice of tooling.

### 2.2 Architecture Overview

The library must address the following concerns. How they are organised into modules, classes, or files is the implementer's decision.

| Concern | Responsibility |
|---------|---------------|
| **Geometry** | Types for points, rectangles, and line segments; utility functions for distance, rotation, intersection, and interpolation |
| **Configuration** | Accepting, validating, and defaulting user-provided settings |
| **Events** | Notifying consumers of state changes, page turns, and orientation switches |
| **Page Representation** | Modelling individual pages with their visual state, density (soft/hard), and rendering logic for both Canvas and HTML |
| **Page Organisation** | Grouping pages into spreads, tracking the current position, and determining which pages participate in a flip |
| **Flip Geometry** | Computing rotation angles, constrained positions, clipping polygons, and shadow data from a touch position |
| **Flip Lifecycle** | Managing the flip state (idle, dragging, animating) and coordinating between user input, calculations, and rendering |
| **Rendering** | Driving the animation loop, converting between coordinate spaces, computing responsive layout, and painting frames in Canvas or HTML/CSS |
| **Input Handling** | Capturing mouse/touch events, detecting swipes, and translating raw DOM events into flip actions |
| **Public API** | A single entry-point class exposing all consumer-facing functionality |

### 2.3 Build Output

| File | Format | Usage |
|------|--------|-------|
| `dist/page-flip.browser.js` | UMD (global: `PageFlip`) | `<script>` tag, CommonJS, AMD |
| `dist/page-flip.module.js` | ES Module | Bundlers (Webpack, Rollup, Vite) |
| `dist/*.d.ts` | TypeScript declarations | TypeScript projects |

---

## 3. Phased Implementation Plan

### Phase 1: Foundation

**Milestone:** Compilable project with type system, geometry utilities, configuration, and event system passing all unit tests.

**Deliverables:**
- Project scaffolding (package.json, tsconfig.json, build config, linting, test runner)
- Core geometric type definitions (points, rectangles, line segments, and a book-area rectangle that distinguishes single-page width from total width)
- Configuration interface with sensible defaults and a validation step
- Geometry utility functions covering: Euclidean distance, angle between lines, 2D point rotation, constraining a point to a circle, line-line intersection, point-in-rectangle testing, and linear interpolation of points between two positions
- Event system (subscribe, unsubscribe, emit)

**Acceptance Criteria:**
- [ ] Geometry utilities pass unit tests including edge cases (null inputs, coincident lines, points on circle boundary)
- [ ] Configuration validation rejects invalid inputs (bad sizing mode, non-positive dimensions, non-positive animation time)
- [ ] Stretch mode applies sensible defaults for any unspecified min/max bounds
- [ ] Event system supports multiple handlers per event, method chaining on subscribe, and removal of handlers
- [ ] Project compiles with TypeScript strict mode, no errors

---

### Phase 2: Page Model and Collection System

**Milestone:** Pages can be created, organised into spreads, and navigated programmatically.

**Deliverables:**
- Page abstraction covering: visual state (angle, position, clipping area), density (soft/hard), orientation (left/right), and rendering for both Canvas and HTML
- HTML page implementation: DOM positioning, CSS transforms, clip-path for soft flips, 3D rotation for hard flips
- Image page implementation: Canvas 2D drawing, async image loading with a visual loading indicator
- Page collection with spread organisation algorithm
- Per-page density control via markup for HTML mode

**Acceptance Criteria:**
- [ ] Portrait mode: one page per spread
- [ ] Landscape mode: pages paired into two-page spreads, with single-page spreads for covers
- [ ] Cover pages (first/last, when enabled) treated as hard (rigid) density
- [ ] An odd number of pages results in the final page being a single hard spread
- [ ] Given a flip direction and orientation, the collection correctly identifies which page is flipping and which page is revealed underneath
- [ ] In HTML mode, individual pages can be marked as hard or soft via the markup
- [ ] A loading indicator is shown while images are still fetching
- [ ] In portrait mode, the HTML page that is flipping away can be visually duplicated (so the original remains in place while the copy animates)

---

### Phase 3: Flip Engine

**Milestone:** Flip calculations produce correct geometry for any valid input position, and the flip lifecycle is managed by a state machine.

**Deliverables:**
- Flip geometry calculator: given a touch position, computes the page rotation angle, constrained position, rotated page rectangle, clipping polygons for both pages, shadow geometry, and flip progress
- Flip lifecycle controller: state machine, direction/corner detection, animation orchestration

**Acceptance Criteria:**
- [ ] The page rotation angle responds correctly and continuously to the touch position
- [ ] The touch point is constrained so the page cannot stretch beyond its width
- [ ] A secondary constraint prevents the far edge of the page from crossing the spine
- [ ] Clipping polygons for both the flipping page and the revealed page are geometrically valid (no overlaps, no out-of-bounds drawing)
- [ ] Flip progress is trackable as a percentage (0% at rest, 100% at full flip)
- [ ] Shadow start point and angle are derived from the flip geometry
- [ ] The state machine prevents conflicting interactions (e.g., cannot start a new drag while animating)
- [ ] Flip direction is inferred from which side of the book the user touches
- [ ] Flip corner (top or bottom) is inferred from the vertical position of the touch
- [ ] Hard (rigid) pages rotate around the spine with a simple angular transform that progresses from flat to fully turned
- [ ] When "corner-only" mode is enabled, clicks outside the corner areas do not trigger flips
- [ ] Corner activation zones are proportional to the page diagonal

---

### Phase 4: Rendering System

**Milestone:** Static pages and animated flips render correctly in both Canvas and HTML modes.

**Deliverables:**
- Renderer core: animation loop, coordinate space conversions, responsive layout calculation
- Canvas renderer: full-canvas redraw with image drawing, clipping, and shadow gradients
- HTML renderer: CSS transform-based positioning, clip-path polygons, and shadow elements

**Acceptance Criteria:**
- [ ] Static pages render at correct positions in landscape (two-page spread) and portrait (single page)
- [ ] Animation system plays frames at the correct rate, and can skip to the end on demand
- [ ] Coordinate conversions correctly translate between viewport, book, and page-local coordinate spaces (accounting for flip direction)
- [ ] Responsive layout: stretch mode scales while maintaining aspect ratio; portrait triggers when the container is too narrow for two pages
- [ ] In portrait mode, only one page is visible (the two-page coordinate space is maintained internally but the unused half is hidden)
- [ ] Shadows comprise multiple layers: a permanent spine shadow, an outer shadow projected onto the underlying page, and an inner shadow on the flipping page's surface
- [ ] Shadow opacity decreases and shadow width increases as the flip progresses
- [ ] Hard pages show a back-face effect during their rigid rotation (the static page behind appears to be the reverse side of the cover)
- [ ] Safari's CSS clip-path rendering bug is handled (see `docs/03-reverse-engineered-files.md`, Section 2.2 for details)

---

### Phase 5: UI and Input Handling

**Milestone:** Full user interaction support across mouse, touch, and swipe.

**Deliverables:**
- Input handling layer: DOM event registration, coordinate normalisation, swipe detection
- Canvas UI: canvas element creation and logical resolution management
- HTML UI: page container creation and page element management
- CSS foundation file for layout, 3D context, and shadow positioning

**Acceptance Criteria:**
- [ ] Mouse click on the book triggers an animated page flip
- [ ] Mouse drag on the book folds the page to follow the pointer; on release, the page either completes or reverts based on drag distance
- [ ] A small movement threshold prevents accidental drags from single clicks
- [ ] Touch interactions support the same drag-to-flip behaviour as mouse
- [ ] Fast horizontal swipes are detected and trigger animated flips (swipe right = previous, swipe left = next)
- [ ] Swipe detection distinguishes swipes from slow drags using distance, direction, and time criteria
- [ ] When mobile scroll support is enabled, vertical scrolling works normally while horizontal gestures trigger flips
- [ ] Interactive elements inside pages (links, buttons) receive their click events rather than being captured as flip triggers
- [ ] Window resize recalculates the book layout and detects orientation changes
- [ ] The root element is styled to allow vertical touch scrolling while capturing horizontal gestures
- [ ] GPU compositing and 3D perspective are established for smooth CSS-based animations
- [ ] The CSS foundation is consistent (no class name mismatches between CSS and JS)

---

### Phase 6: Facade and Integration

**Milestone:** Complete public API covering all consumer-facing functionality.

**Deliverables:**
- Public API class integrating all subsystems behind a single entry point
- Full lifecycle, navigation, state query, and event subscription support
- Build configuration producing UMD and ES Module bundles

**Acceptance Criteria:**
- [ ] Consumers can initialise the library in Canvas mode (from image URLs) or HTML mode (from DOM elements)
- [ ] Pages can be replaced at runtime without losing the current position
- [ ] The library can be fully torn down (DOM removed, event handlers cleaned up)
- [ ] In HTML mode, the DOM can be restored to its pre-initialisation state
- [ ] Instant and animated navigation work for next, previous, and specific page targets
- [ ] All documented event types emit with appropriate data
- [ ] State queries (page count, current page, orientation, bounds, flip state) return correct values
- [ ] UMD bundle works via `<script>` tag with a global namespace
- [ ] ES Module bundle works via `import`
- [ ] TypeScript declaration files are generated and accurate

---

### Phase 7: Testing and Polish

**Milestone:** Production-ready release with comprehensive test coverage.

**Deliverables:**
- Unit test suite (geometry helpers, configuration validation, flip calculations, state machine, event system, spread creation)
- Integration test suite (rendering pipeline, user interaction flow, orientation switching)
- Performance benchmarks (animation frame rate under load)
- API documentation (generated or hand-written reference)
- Usage examples (Canvas mode, HTML mode, responsive layout, event handling)

**Acceptance Criteria:**
- [ ] All unit tests pass
- [ ] No TypeScript errors in strict mode
- [ ] Bundle size is reasonable for a zero-dependency library
- [ ] 60fps animation sustained on Chrome DevTools "mid-tier mobile" throttle
- [ ] Cross-browser validation: Chrome, Firefox, Safari, Edge, mobile Safari, mobile Chrome
- [ ] API documentation covers all public methods, events, and configuration options
- [ ] Both usage examples (Canvas and HTML) run correctly

---

## 4. Dependencies and Sequencing

```
Phase 1: Foundation
    |
    +-- Phase 2: Page Model     (depends on Phase 1)
    |       |
    +-- Phase 3: Flip Engine    (depends on Phase 1)
    |       |
    +-------+
        |
    Phase 4: Rendering          (depends on Phases 2 + 3)
        |
    Phase 5: UI                 (depends on Phase 1; can start structure in parallel with Phase 4)
        |
    Phase 6: Facade             (depends on Phases 4 + 5)
        |
    Phase 7: Testing            (ongoing; final acceptance after Phase 6)
```

**Parallelism opportunities:**
- Phases 2 and 3 can proceed simultaneously after Phase 1 completes.
- Phase 5 UI structure (DOM creation, event registration) can begin alongside Phase 4, though full integration requires rendering to be complete.
- Testing should be written incrementally alongside each phase.

---

## 5. Known Challenges and Risks

| # | Challenge | Mitigation |
|---|-----------|------------|
| 1 | **Flip geometry edge cases** -- Near-zero angles, positions near the spine, and coincident line segments can cause division-by-zero or degenerate polygons | Comprehensive unit tests with boundary inputs; guard clauses for degenerate angle values |
| 2 | **Safari CSS clip-path bug** -- WebKit bug #126207 causes incorrect rendering when `clip-path` is combined with certain 3D transforms | Browser detection and fallback transform when the rotation is zero |
| 3 | **Mobile scroll vs flip conflict** -- Horizontal touch movement must trigger page flipping while vertical movement must allow page scrolling | A horizontal dead zone before activating flip; only suppress scroll when a flip is actively in progress |
| 4 | **Hard/soft density matching in landscape** -- Adjacent pages with different densities create visual artefacts during flipping | Temporarily force both pages to the same density for the duration of the flip |
| 5 | **Portrait mode coordinate space** -- Only one page is visible, but the flip geometry may assume a two-page layout | Ensure the hidden half of the coordinate space does not leak into the visible area |
| 6 | **Per-frame polygon construction performance** -- HTML mode may need to rebuild clip-path strings on every animation frame | Minimise allocations; benchmark and optimise if frame rate drops |
| 7 | **Image loading races** -- Pages may be flipped before their images have loaded | Show a loading indicator; guard drawing logic against incomplete loads |

---

## 6. Improvements Over Reference

The following are recommended quality improvements over the reference implementation:

| # | Improvement | Rationale |
|---|-------------|-----------|
| 1 | **CSS/JS consistency** | Ensure all class names referenced in JavaScript match the shipped CSS |
| 2 | **Granular event unsubscription** | Support removing individual event handlers, not just all handlers for a given event |
| 3 | **TypeScript strict mode** | Compile under strict mode from the start |
| 4 | **Comprehensive test suite** | Include unit and integration tests (the reference has none) |
| 5 | **Immutable configuration** | Do not mutate default values when merging user settings |
| 6 | **Clean type handling** | Maintain proper types throughout -- avoid string-to-number workarounds |
| 7 | **Explicit error handling** | Use meaningful error handling rather than silently swallowing exceptions |
| 8 | **Consider easing functions** | Easing curves (ease-in-out) could produce smoother animation with fewer frames than linear pixel-stepping |
