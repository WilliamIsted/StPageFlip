# StPageFlip - Core Parts and Principles

## 1. Project Identity

StPageFlip is a zero-dependency TypeScript library that simulates realistic page-turning effects in the browser. It transforms a flat collection of pages (images or HTML elements) into an interactive book with drag-to-flip, click-to-turn, and swipe gestures. The visual effect mimics a physical page curling away from the spine, complete with dynamic drop shadows.

## 2. Core Architectural Principles

### 2.1 Dual Rendering Strategy

The library supports two fundamentally different rendering approaches behind a single API:

- **Canvas Mode**: Pages are images drawn onto an HTML5 `<canvas>` element. The flip animation, clipping, and shadows are all computed and painted via the Canvas 2D API. This mode is best for image-heavy content (photo books, scanned documents).

- **HTML Mode**: Pages are actual DOM elements (`<div>`, etc.) positioned and transformed using CSS `transform`, `clip-path`, and `z-index`. Shadows are separate `<div>` elements with CSS gradients. This mode preserves DOM interactivity (links, buttons, forms inside pages).

Both modes share the same flip calculation engine and animation system; only the final rendering step differs.

### 2.2 Facade Pattern (Single Entry Point)

The `PageFlip` class (`src/PageFlip.ts`) is the only class a consumer instantiates. It:

- Accepts a root HTML element and a configuration object
- Internally creates the appropriate UI, Renderer, Flip controller, and Page collection
- Exposes a clean public API for navigation, state queries, and event subscription
- Shields consumers from the internal class hierarchy entirely

### 2.3 State Machine for Flip Lifecycle

The flip process is governed by a four-state machine (`FlippingState`):

```
READ  ->  FOLD_CORNER  (mouse hovers near corner)
READ  ->  USER_FOLD    (user starts dragging)
USER_FOLD -> FLIPPING  (user clicks to flip, or drag resolves)
FLIPPING  -> READ      (animation completes)
FOLD_CORNER -> READ    (mouse leaves corner area)
```

State transitions trigger events (`changeState`) that consumers can listen to. The state machine prevents conflicting interactions (e.g., starting a new flip while one is animating).

### 2.4 Separation of Calculation and Rendering

The mathematical model of a page flip is entirely isolated in `FlipCalculation`. Given a touch point, it computes:

- The **rotation angle** of the turning page
- The **clipping polygon** for both the flipping page and the page beneath it
- The **shadow origin, angle, and opacity**
- The **position** of the active corner

These pure geometric outputs are then consumed by whichever renderer is active. This separation means the math never needs to know whether it is driving Canvas draw calls or CSS transforms.

### 2.5 Coordinate Space Architecture

Three coordinate systems are used throughout:

| Space | Origin | Used By |
|-------|--------|---------|
| **Window (Global)** | Top-left of browser viewport | UI event handlers, DOM positioning |
| **Book** | Top-left of the book's bounding rectangle | Direction detection, corner proximity checks |
| **Page** | Top-left of the active (right or left) page | All flip calculations, clipping, shadow math |

The `Render` base class provides `convertToBook()`, `convertToPage()`, `convertToGlobal()`, and `convertRectToGlobal()` to translate between these spaces. This ensures the flip math always operates in a normalised page-local coordinate system regardless of where the book sits on screen.

### 2.6 Spread-Based Page Management

Pages are not simply a flat list. The `PageCollection` organises them into **spreads**:

- **Landscape mode**: Two-page spreads (left + right), with optional single-page cover spreads
- **Portrait mode**: Single-page spreads

Navigation, rendering, and flipping all operate on spread indices, not raw page indices. This abstraction handles the complexity of:

- Showing the correct left/right pages for a given spread
- Determining which page flips and which page is revealed underneath
- Cover pages (first/last) that display as single hard pages

### 2.7 Page Density (Soft vs Hard)

Each page has a "density" property:

- **Soft**: The page curls and bends realistically during a flip. Clipping polygons are computed, and the page is rotated around the touch point.
- **Hard**: The page rotates rigidly around its spine edge, like a book cover. Uses a simpler angle-based rotation (CSS `rotateY` in HTML mode, or angle-based draw in Canvas mode).

Cover pages are automatically set to `HARD` density. In landscape mode, adjacent pages inherit matching densities to prevent visual inconsistency during a flip.

### 2.8 Event-Driven Communication

The `EventObject` base class provides a simple pub/sub system:

- **`on(eventName, callback)`** - Subscribe to events
- **`off(eventName)`** - Unsubscribe all handlers for an event
- **`trigger(eventName, app, data)`** - Emit events (protected, internal use)

Key events emitted:
- `init` - Book initialised
- `flip` - Page number changed
- `changeState` - Flip state transitioned
- `changeOrientation` - Portrait/landscape switch
- `update` - Pages reloaded

### 2.9 Responsive Layout

The library supports two sizing modes:

- **Fixed** (`size: 'fixed'`): Book dimensions are exactly as configured. Portrait mode triggers when the container is too narrow to fit two pages.
- **Stretch** (`size: 'stretch'`): Book scales to fill its parent container, bounded by `minWidth`/`maxWidth`/`minHeight`/`maxHeight`. Aspect ratio is preserved.

Orientation (portrait vs landscape) is recalculated on every `resize` event and on explicit `update()` calls. The rendering area (`boundsRect`) is always centred within the parent container.

### 2.10 Animation System

Animations are pre-computed arrays of frame functions:

1. The start and end points are defined
2. `Helper.GetCordsFromTwoPoint()` interpolates pixel-by-pixel between them
3. Each interpolated point becomes a frame closure: `() => this.do(point)`
4. The `Render.startAnimation()` method plays these frames using `requestAnimationFrame`, distributing them evenly across the configured `flippingTime`
5. A callback fires on completion to finalise the page turn or reset state

This approach gives smooth, deterministic animations regardless of frame rate variation, though the total frame count scales with the pixel distance between start and end points.

## 3. Core Mathematical Model

The flip effect is built on these geometric operations:

1. **Angle Calculation**: Given the touch position relative to the page corner, compute the page rotation angle using `2 * acos(dx / hypotenuse)`.

2. **Circle Constraint**: The touch point is constrained to a circle centred at the page corner with radius equal to the page width. This prevents the page from being "pulled" beyond physically possible positions.

3. **Diagonal Constraint**: A secondary constraint uses the page diagonal as a radius to prevent the opposite corner from crossing the spine.

4. **Rectangle Rotation**: The four corners of the page rectangle are rotated by the computed angle around the touch point, producing the `RectPoints` of the flipping page.

5. **Intersection Clipping**: The rotated page rectangle is intersected with the book boundaries (top, side, bottom edges) to produce clipping polygons for both the flipping page and the revealed page beneath.

6. **Shadow Geometry**: Shadow start point, angle, width, and opacity are derived from the intersection points and flip progress (0-100%).

## 4. Input Handling Model

The UI layer normalises mouse and touch events into three abstract actions:

| Action | Trigger | Effect |
|--------|---------|--------|
| `startUserTouch(pos)` | mousedown / touchstart (after swipe timeout) | Records initial position, sets touch flag |
| `userMove(pos, isTouch)` | mousemove / touchmove | If touching and moved >5px: drag-fold. If hovering: show corner peek |
| `userStop(pos, isSwipe)` | mouseup / touchend | If didn't move: click-flip. If moved: resolve drag direction |

Swipe detection runs separately: if a touch starts and ends within 250ms and covers more than `swipeDistance` pixels horizontally (with limited vertical movement), it triggers an animated flip without requiring a drag.

## 5. Configuration Surface

The `FlipSetting` interface exposes 18 configuration options covering:

- **Dimensions**: `width`, `height`, `minWidth`, `maxWidth`, `minHeight`, `maxHeight`
- **Sizing mode**: `size` (fixed or stretch)
- **Behaviour**: `startPage`, `flippingTime`, `usePortrait`, `showCover`, `showPageCorners`, `disableFlipByClick`
- **Visual**: `drawShadow`, `maxShadowOpacity`, `startZIndex`
- **Input**: `useMouseEvents`, `mobileScrollSupport`, `clickEventForward`, `swipeDistance`
- **Layout**: `autoSize`

All have sensible defaults. Validation rejects invalid sizes, negative dimensions, and unknown size types.
