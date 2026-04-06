# StPageFlip - Reverse-Engineered File Analysis

This document provides a file-by-file breakdown of every source file in the library, organised by architectural layer. The level of detail is intended to enable a from-scratch reimplementation without reference to the original source code.

---

## 1. Foundation Layer

### 1.1 BasicTypes.ts (48 lines)

**Role:** Defines the core geometric primitives used by every other module. All types are pure data structures with no behaviour.

**Dependencies:** None.

**Exported Types:**

| Type | Shape | Purpose |
|------|-------|---------|
| `Point` | `{ x: number, y: number }` | A position on a 2D plane. The universal coordinate type. |
| `RectPoints` | `{ topLeft: Point, topRight: Point, bottomLeft: Point, bottomRight: Point }` | Four named corners of a rectangle. Used to represent the rotated page during flipping. |
| `Rect` | `{ left: number, top: number, width: number, height: number }` | An axis-aligned rectangle. Used for bounding boxes and hit-testing. |
| `PageRect` | `{ left: number, top: number, width: number, height: number, pageWidth: number }` | Extends `Rect` semantically by adding `pageWidth`. In landscape mode, `pageWidth` is half of `width` (one page); in portrait mode, `pageWidth` equals `width`. This distinction is critical for all coordinate conversions and flip calculations. |
| `Segment` | `[Point, Point]` (tuple) | A line segment defined by its two endpoints. Used for line intersection calculations. |

**Implementation Notes:**
- `PageRect` is structurally independent from `Rect` (not a TypeScript `extends`), but shares the same four fields plus `pageWidth`.
- `Segment` is a tuple type rather than an interface, making it compact for the heavy use in intersection calculations.

---

### 1.2 Settings.ts (121 lines)

**Role:** Defines the configuration interface (`FlipSetting`) and provides a validation/defaults class (`Settings`). This is the only place where user-provided configuration is processed.

**Dependencies:** None.

**Exported Types:**

**`SizeType` (const enum):**
- `FIXED` (`'fixed'`) -- Book dimensions are exactly as configured.
- `STRETCH` (`'stretch'`) -- Book scales to fill its parent container.

**`FlipSetting` (interface) -- 18 fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `startPage` | number | 0 | Page number to display on load |
| `size` | SizeType | FIXED | Sizing mode |
| `width` | number | 0 | Base page width in pixels |
| `height` | number | 0 | Base page height in pixels |
| `minWidth` | number | 0 | Minimum page width (stretch mode) |
| `maxWidth` | number | 0 | Maximum page width (stretch mode) |
| `minHeight` | number | 0 | Minimum page height (stretch mode) |
| `maxHeight` | number | 0 | Maximum page height (stretch mode) |
| `drawShadow` | boolean | true | Whether to render drop shadows |
| `flippingTime` | number | 1000 | Animation duration in milliseconds |
| `usePortrait` | boolean | true | Allow portrait mode when container is narrow |
| `startZIndex` | number | 0 | Base z-index for page stacking |
| `autoSize` | boolean | true | Set parent element width to 100% |
| `maxShadowOpacity` | number | 1 | Shadow intensity (0 = invisible, 1 = full) |
| `showCover` | boolean | false | Treat first/last pages as hard covers |
| `mobileScrollSupport` | boolean | true | Allow vertical scrolling on mobile |
| `clickEventForward` | boolean | true | Forward clicks on links/buttons inside pages |
| `useMouseEvents` | boolean | true | Enable mouse and touch event handling |
| `swipeDistance` | number | 30 | Minimum horizontal pixels for swipe detection |
| `showPageCorners` | boolean | true | Show corner fold preview on mouse hover |
| `disableFlipByClick` | boolean | false | Only allow flipping by dragging corners |

**`Settings` class:**

Single method: `getSettings(userSetting)` -- Merges user settings onto defaults via `Object.assign`, then validates:

1. `size` must be `'fixed'` or `'stretch'` (throws `Error`)
2. `width` and `height` must be > 0 (throws `Error`)
3. `flippingTime` must be > 0 (throws `Error`)
4. If stretch mode: auto-corrects `minWidth` to 100 if <= 0, `maxWidth` to 2000 if < `minWidth`, and same for height
5. If fixed mode: forces `minWidth` = `maxWidth` = `width` and `minHeight` = `maxHeight` = `height`

**Implementation Notes:**
- `Object.assign` mutates the internal `_default` object, which means calling `getSettings` multiple times on the same `Settings` instance would accumulate settings from previous calls. In practice, `Settings` is only instantiated once per `PageFlip`.

---

### 1.3 Helper.ts (203 lines)

**Role:** A static-only utility class providing the geometric primitives that underpin the flip calculation engine. Every method is pure (no side effects).

**Dependencies:** `Point`, `Rect`, `Segment` from `BasicTypes`.

**Key Methods:**

**`GetDistanceBetweenTwoPoint(p1, p2) -> number`:** Standard Euclidean distance. Returns `Infinity` if either point is null.

**`GetSegmentLength(segment) -> number`:** Wrapper around distance for a `Segment` tuple.

**`GetAngleBetweenTwoLine(line1, line2) -> number`:** Computes the angle between two lines (not segments) using the dot product formula. Converts each line to its general form (A, B coefficients from the two endpoints), then returns `acos((A1*A2 + B1*B2) / (|v1| * |v2|))`. Used for shadow angle calculations.

**`PointInRect(rect, pos) -> Point | null`:** Hit-test: returns the point if it falls within the rectangle, otherwise null. Used to clip intersection results to the book boundaries.

**`GetRotatedPoint(transformedPoint, startPoint, angle) -> Point`:** Applies a 2D rotation matrix. The formula:
- `x' = x*cos(a) + y*sin(a) + startPoint.x`
- `y' = y*cos(a) - x*sin(a) + startPoint.y`

Note: the `transformedPoint` coordinates are treated as offsets from the origin, and `startPoint` is added as a translation after rotation.

**`LimitPointToCircle(startPoint, radius, limitedPoint) -> Point`:** Constrains a point to lie within a circle. If the point is already inside, returns it unchanged. Otherwise, finds the intersection of the line from `startPoint` to `limitedPoint` with the circle. The intersection formula:
- `x = sqrt(r^2 * (a-n)^2 / ((a-n)^2 + (b-m)^2)) + a`
- Flips sign if `limitedPoint.x < 0`
- `y = ((x-a) * (b-m)) / (a-n) + b`
- Special case: if `a - n + b === 0`, sets `y = radius`

This is critical for the flip constraint system -- it prevents the user from dragging a page corner beyond a physically possible position.

**`GetIntersectBetweenTwoSegment(rectBorder, one, two) -> Point | null`:** Finds where two infinite lines intersect, then clips to within a bounding rectangle. Combines `GetIntersectBeetwenTwoLine` with `PointInRect`.

**`GetIntersectBeetwenTwoLine(one, two) -> Point | null`:** Finds the intersection of two infinite lines using the general line equation form (Ax + By + C = 0). Uses the determinant method:
- `C1 = x0*y1 - x1*y0` for each line
- `x = -(C1*B2 - C2*B1) / (A1*B2 - A2*B1)`
- `y = -(A1*C2 - A2*C1) / (A1*B2 - A2*B1)`
- Returns the point if both coordinates are finite
- Throws `Error('Segment included')` if the lines are coincident (determinant difference < 0.1)
- Returns null if lines are parallel but not coincident

Note: The method name contains a typo (`Beetwen`), preserved from the original.

**`GetCordsFromTwoPoint(p1, p2) -> Point[]`:** Generates an array of intermediate points between two endpoints, stepping 1 pixel at a time along the longer axis. The shorter axis is interpolated proportionally. This is the animation frame generator -- each point in the returned array becomes one frame of a flip animation.

Algorithm:
1. Compute `sizeX = |p1.x - p2.x|` and `sizeY = |p1.y - p2.y|`
2. `lengthLine = max(sizeX, sizeY)` determines the total step count
3. For each step `i` from 1 to `lengthLine`, compute each coordinate by linear interpolation: if `c2 > c1`, add `i * (size / length)`; if `c2 < c1`, subtract it; if equal, hold constant.

---

### 1.4 Event/EventObject.ts (56 lines)

**Role:** Provides a minimal pub/sub event system. This is an abstract base class; only `PageFlip` (which extends it) is ever instantiated.

**Dependencies:** `PageFlip` (circular reference -- used only as a type in the `WidgetEvent` interface).

**Exported Types:**

- `DataType` = `number | string | boolean | object` -- Union type for event payloads.
- `WidgetEvent` = `{ data: DataType, object: PageFlip }` -- The event object passed to callbacks.
- `EventCallback` = `(e: WidgetEvent) => void` -- Handler function type (not exported, internal).

**Class: `EventObject` (abstract)**

Internal state: `events = new Map<string, EventCallback[]>()` -- Maps event names to arrays of handler functions.

Methods:
- **`on(eventName, callback) -> EventObject`:** Appends the callback to the handler array for the event. Creates a new array if the event has no handlers yet. Returns `this` for method chaining.
- **`off(event) -> void`:** Deletes all handlers for the given event name. There is no way to remove a single handler -- `off` is all-or-nothing.
- **`trigger(eventName, app, data?) -> void`:** Protected method. Invokes all registered handlers for the event, passing a `WidgetEvent` with the data and PageFlip reference. No-ops silently if no handlers are registered.

**Implementation Notes:**
- The `on` method supports multiple handlers per event (push to array), but `off` removes them all at once.
- There is a circular import: `EventObject` imports `PageFlip` for typing, and `PageFlip` extends `EventObject`. This works because TypeScript resolves it at compile time.

---

## 2. Page Model Layer

### 2.1 Page/Page.ts (186 lines)

**Role:** Abstract base class representing a single page in the book. Defines the state interface, density/orientation enums, and the contract that concrete page types must fulfil.

**Dependencies:** `Render`, `Point`.

**Exported Types:**

**`PageState` (interface):**
| Field | Type | Purpose |
|-------|------|---------|
| `angle` | number | Current rotation angle (radians) for soft page flipping |
| `area` | Point[] | Polygon points defining the visible (clipped) region of the page |
| `position` | Point | Current position of the active corner |
| `hardAngle` | number | Rotation angle for hard (rigid) pages |
| `hardDrawingAngle` | number | Actual angle applied at render time (may differ from `hardAngle` for back-face rendering) |

**`PageOrientation` (const enum):** `LEFT` (0), `RIGHT` (1).

**`PageDensity` (const enum):** `SOFT` (`'soft'`), `HARD` (`'hard'`).

**Class: `Page` (abstract)**

Constructor: Accepts a `Render` reference and a `PageDensity`. Initialises `state` with zeroed values, sets both `createdDensity` and `nowDrawingDensity` to the provided density.

Two density properties:
- `createdDensity` -- The permanent density assigned at creation (e.g., cover pages are HARD).
- `nowDrawingDensity` -- A temporary override for the current render frame. Used when landscape mode forces adjacent pages to share the same density.

Abstract methods that subclasses must implement:
- `simpleDraw(orient: PageOrientation)` -- Render as a static (non-flipping) page.
- `draw(tempDensity?: PageDensity)` -- Render as a dynamic (flipping) page using current state.
- `load()` -- Perform any async loading (e.g., image fetch).
- `newTemporaryCopy() -> Page` -- Create a clone for portrait mode flipping.
- `getTemporaryCopy() -> Page` -- Retrieve the existing clone.
- `hideTemporaryCopy()` -- Remove the clone from the DOM.

Setter methods: `setDensity`, `setDrawingDensity`, `setPosition`, `setAngle`, `setArea`, `setHardDrawingAngle`, `setHardAngle` (also sets `hardDrawingAngle`), `setOrientation`.

Getter methods: `getDrawingDensity`, `getDensity`, `getHardAngle`.

---

### 2.2 Page/HTMLPage.ts (176 lines)

**Role:** Concrete page implementation for HTML mode. Wraps a DOM element and controls its visibility, positioning, and CSS transforms during flipping.

**Dependencies:** `Page`, `Render`, `Helper`, `FlipDirection`, `Point`, `PageDensity`, `PageOrientation`.

**Constructor:** Accepts a `Render`, an `HTMLElement`, and a `PageDensity`. Adds CSS classes `stf__item` and `--soft` or `--hard` to the element.

**Key Methods:**

**`draw(tempDensity?)`:** Dispatches to `drawHard` or `drawSoft` based on current density.

**`drawHard(commonStyle)`:** Renders a rigid page flip using CSS 3D rotation:
- Transform origin is set to the spine edge: `pageWidth px 0` for left pages, `0 0` for right pages.
- Transform: `translate3d(posX, 0, 0) rotateY(angle deg)`.
- `backface-visibility: hidden` prevents showing the reverse side.
- For right-oriented pages, translates to the book centre (`rect.left + rect.width / 2`).

**`drawSoft(position, commonStyle)`:** Renders a curved page flip using CSS clip-path and 2D rotation:
1. Constructs a `clip-path: polygon(...)` string from the `state.area` points.
2. For each point, transforms from page-local to clip-local coordinates. The transform differs by direction: for BACK direction, x is negated (`-p.x + position.x`); for FORWARD, x is relative (`p.x - position.x`).
3. Each transformed point is then rotated by `state.angle` around the origin using `Helper.GetRotatedPoint`.
4. **Safari workaround:** When the angle is exactly 0 and the browser is Safari, uses `transform: translate(x, y)` instead of `translate3d(x, y, 0) rotate(angle rad)` to avoid a WebKit clip-path rendering bug (WebKit bug #126207).

**`simpleDraw(orient)`:** Renders a static page at its normal position. Applies `--simple` CSS class. Positions absolutely using the book rect's coordinates.

**`newTemporaryCopy()`:** If density is HARD, returns `this` (hard pages don't need copies). Otherwise, clones the DOM element via `cloneNode(true)`, appends the clone to the same parent, and creates a new `HTMLPage` wrapping the clone. This is needed in portrait mode where the same page needs to appear in two states simultaneously.

**`hideTemporaryCopy()`:** Removes the cloned element from the DOM and nulls the reference.

**`setOrientation(orient)`:** Updates the element's CSS classes (`--left` or `--right`).

**`setDrawingDensity(density)`:** Updates CSS classes (`--soft` or `--hard`) in addition to the base class behaviour.

---

### 2.3 Page/ImagePage.ts (126 lines)

**Role:** Concrete page implementation for Canvas mode. Manages an `HTMLImageElement` and draws it onto a Canvas 2D context.

**Dependencies:** `CanvasRender`, `Page`, `Render`, `Point`, `PageDensity`, `PageOrientation`.

**Constructor:** Creates a new `Image()` element and sets its `src` to the provided URL.

**Internal State:**
- `image: HTMLImageElement` -- The loaded image.
- `isLoad: boolean` -- Whether the image has finished loading.
- `loadingAngle: number` -- Current rotation angle for the loading spinner animation (starts at 0).

**Key Methods:**

**`draw(tempDensity?)`:** Draws the flipping page onto the Canvas:
1. Gets the 2D context via `(render as CanvasRender).getContext()`.
2. Translates to the global page position.
3. Constructs a clipping path from `state.area` points (each converted to global coordinates).
4. Rotates the canvas by `state.angle`.
5. Clips and draws either the image or the loading spinner.

**`simpleDraw(orient)`:** Draws the page at its static position without clipping or rotation.

**`drawLoader(ctx, shiftPos, pageWidth, pageHeight)`:** Renders a loading spinner:
1. Draws a white rectangle with a grey border (the page background).
2. Draws a circular arc centred on the page: `arc(centerX, centerY, 20, loadingAngle, 3*PI/2 + loadingAngle)` with a line width of 10.
3. Increments `loadingAngle` by 0.07 radians per frame, resetting to 0 at `2*PI`. This creates a continuously spinning 270-degree arc.

**`load()`:** Sets up the `onload` callback on the image element to flip `isLoad` to true.

**`newTemporaryCopy()` / `getTemporaryCopy()`:** Both return `this`. Canvas pages don't need DOM cloning because the Canvas API draws pixels directly -- there is no DOM element to duplicate.

**`hideTemporaryCopy()`:** No-op.

---

## 3. Collection Layer

### 3.1 Collection/PageCollection.ts (293 lines)

**Role:** Abstract base class managing the ordered list of pages and organising them into spreads for navigation. This is where the concept of "current page" lives, and where the logic for which pages participate in a flip is determined.

**Dependencies:** `Render`, `Orientation`, `Page`, `PageDensity`, `PageFlip`, `FlipDirection`.

**Internal State:**
- `pages: Page[]` -- The full list of pages.
- `currentPageIndex: number` -- Index into `pages` of the currently displayed page (first page of current spread).
- `currentSpreadIndex: number` -- Index into the current spread array.
- `landscapeSpread: number[][]` -- Two-page spreads for landscape mode.
- `portraitSpread: number[][]` -- Single-page spreads for portrait mode.
- `isShowCover: boolean` -- From settings, determines cover handling.

**Spread Creation Algorithm (`createSpread()`):**

Portrait spreads: One spread per page. `[[0], [1], [2], ...]`

Landscape spreads:
1. If `showCover` is true: page 0 is a single-page spread, set to HARD density. Start pairing from page 1.
2. Pair consecutive pages: `[1,2], [3,4], ...`
3. If the total page count is such that the last page is unpaired, it becomes a single-page spread and is set to HARD density.

Example with 7 pages and `showCover: true`:
- Landscape: `[[0], [1,2], [3,4], [5,6]]`
- Portrait: `[[0], [1], [2], [3], [4], [5], [6]]`

**Page Selection for Flipping:**

**`getFlippingPage(direction) -> Page`:**
- **Portrait, forward:** Returns `currentPage.newTemporaryCopy()` -- a clone of the current page that will be animated as the flipping page.
- **Portrait, backward:** Returns `pages[currentSpreadIndex - 1]`.
- **Landscape, forward:** Returns the first page of the next spread.
- **Landscape, backward:** Returns the second page of the previous spread (the right-hand page).
- For single-page spreads (covers), returns that single page.

**`getBottomPage(direction) -> Page`:**
- **Portrait, forward:** Returns `pages[currentSpreadIndex + 1]`.
- **Portrait, backward:** Returns `pages[currentSpreadIndex - 1]`.
- **Landscape, forward:** Returns the second page of the next spread.
- **Landscape, backward:** Returns the first page of the previous spread.
- For single-page spreads, returns that single page.

**`showSpread()`:** Determines which pages to display:
- Two-page spread: left page is `spread[0]`, right page is `spread[1]`.
- Single-page landscape spread: if it is the last page, place it on the left (right is null); if it is the first page, place it on the right (left is null).
- Portrait: always right side only (left is null).
- Updates `currentPageIndex` to `spread[0]` and emits the `flip` event via `app.updatePageIndex`.

Other methods: `show(pageNum)` looks up the spread index for a given page number and calls `showSpread()`. `showNext/showPrev` increment/decrement `currentSpreadIndex` and call `showSpread()`.

---

### 3.2 Collection/HTMLPageCollection.ts (40 lines)

**Role:** Concrete collection for HTML mode. Creates `HTMLPage` instances from DOM elements.

**Dependencies:** `HTMLPage`, `PageCollection`, `Render`, `PageFlip`, `PageDensity`.

**`load()` method:** Iterates over the provided `NodeListOf<HTMLElement>` or `HTMLElement[]`. For each element:
1. Reads the `data-density` HTML attribute. If the value is `'hard'`, the page gets `PageDensity.HARD`; otherwise `PageDensity.SOFT`.
2. Creates an `HTMLPage` wrapping the element.
3. Calls `page.load()` and pushes to the pages array.
4. Calls `createSpread()` to build the spread structure.

This is the only mechanism for per-page density control -- it is a data attribute on the HTML element itself.

---

### 3.3 Collection/ImagePageCollection.ts (29 lines)

**Role:** Concrete collection for Canvas mode. Creates `ImagePage` instances from image URLs.

**Dependencies:** `ImagePage`, `PageCollection`, `Render`, `PageFlip`, `PageDensity`.

**`load()` method:** Iterates over the provided string array of image URLs. For each URL:
1. Creates an `ImagePage` with `PageDensity.SOFT` (all image pages are soft by default -- no per-page density override).
2. Calls `page.load()` and pushes to the pages array.
3. Calls `createSpread()`.

---

## 4. Flip Engine Layer

### 4.1 Flip/FlipCalculation.ts (433 lines)

**Role:** The mathematical core of the library. Given a touch position (in page-local coordinates), computes the rotation angle, the four corners of the rotated page rectangle, intersection points with book boundaries, clipping polygons for both the flipping and bottom pages, and shadow geometry. All outputs are pure geometric data -- no rendering occurs here.

**Dependencies:** `Helper`, `Point`, `Rect`, `RectPoints`, `Segment`, `FlipCorner`, `FlipDirection`.

**Constructor:** Accepts `direction` (FlipDirection), `corner` (FlipCorner), `pageWidth` (string), and `pageHeight` (string). The width and height are passed as strings and parsed back to integers via `parseInt(x, 10)`. This is an intentional workaround for a type-casting bug where JavaScript would sometimes treat these values as strings during arithmetic operations.

**Internal State:**
- `angle: number` -- The computed rotation angle.
- `position: Point` -- The constrained position of the active corner.
- `rect: RectPoints` -- The four corners of the rotated page rectangle.
- `topIntersectPoint`, `sideIntersectPoint`, `bottomIntersectPoint` -- Where the rotated page edges cross the book boundaries. Any of these can be null if the intersection falls outside the page bounds.

**Main Calculation Flow (`calc(localPos) -> boolean`):**

1. `calcAngleAndPosition(localPos)` -- Computes angle, constrains position, builds rotated rect.
2. `calculateIntersectPoint(position)` -- Finds where the rotated page crosses the book edges.
3. Returns true on success, false if any error occurs (catches exceptions internally).

**Angle Calculation (`calculateAngle(pos) -> number`):**

The rotation angle represents how far the page has been folded. Given the touch position relative to the page corner at `(pageWidth, 0)` or `(pageWidth, pageHeight)`:

1. `left = pageWidth - pos.x + 1` (horizontal distance from touch to right edge, with +1 offset)
2. `top` depends on corner: for BOTTOM, `top = pageHeight - pos.y`; for TOP, `top = pos.y`.
3. `angle = 2 * acos(left / sqrt(top^2 + left^2))` -- This derives the rotation from the inscribed angle formed by the fold line.
4. If `top < 0`, negate the angle.
5. Guard: if angle is not finite, or if `PI - angle < 0.003` (near-flat page), throw an error to abort the calculation.
6. For BOTTOM corner, negate the final angle.

**Position Constraint System (`checkPositionAtCenterLine`):**

Two sequential constraints prevent physically impossible page positions:

1. **Primary constraint (circle at active corner):** The touch point is limited to a circle centred at the active corner (either `(0,0)` or `(0, pageHeight)`) with radius equal to `pageWidth`. This prevents the page from being "stretched" beyond its width. Uses `Helper.LimitPointToCircle`.

2. **Secondary constraint (diagonal circle):** After applying the primary constraint and recomputing the rotated rectangle, checks whether the far edge of the rotated page has crossed the spine (x <= 0). If so, constrains the opposite corner point to a circle centred at the other corner, with radius equal to the page diagonal (`sqrt(pageWidth^2 + pageHeight^2)`). This prevents the bottom of the page from swinging past the spine during a top-corner flip (and vice versa).

After each constraint application, the angle and geometry are recalculated via `updateAngleAndGeometry`.

**Page Rectangle Construction (`getPageRect`, `getRectFromBasePoint`):**

Defines the four unrotated corners of the page:
- **Top corner flip:** `(0,0), (pageWidth,0), (0,pageHeight), (pageWidth,pageHeight)`
- **Bottom corner flip:** `(0,-pageHeight), (pageWidth,-pageHeight), (0,0), (pageWidth,0)` -- Y-axis is shifted so the bottom edge is at y=0 (the touch point).

Each corner is then rotated around the touch position by the computed angle using `getRotatedPoint` (the local version, identical to `Helper.GetRotatedPoint`).

**Intersection Point Calculation (`calculateIntersectPoint`):**

Uses a bounding rect slightly larger than the page (`left: -1, top: -1, width: pageWidth+2, height: pageHeight+2`) to account for floating-point edge cases.

For **top corner** flips, finds intersections between:
- Top boundary: line from position to rotated topRight vs horizontal line at y=0
- Side boundary: line from position to rotated bottomLeft vs vertical line at x=pageWidth
- Bottom boundary: line from rotated bottomLeft to bottomRight vs horizontal line at y=pageHeight

For **bottom corner** flips, uses different edge pairs:
- Top boundary: line from rotated topLeft to topRight vs y=0
- Side boundary: line from position to rotated topLeft vs x=pageWidth
- Bottom boundary: same as top corner (bottomLeft to bottomRight vs y=pageHeight)

**Clipping Polygon Construction:**

**`getFlippingClipArea() -> Point[]`:** Builds a polygon representing the visible portion of the flipping page:
1. Always starts with `rect.topLeft` and `topIntersectPoint`.
2. If `sideIntersectPoint` is not null, includes it.
3. Always includes `bottomIntersectPoint`.
4. If the clip wraps around the bottom or the flip is from the bottom corner, includes `rect.bottomLeft`.

**`getBottomClipArea() -> Point[]`:** Builds a polygon for the page revealed beneath:
1. Starts with `topIntersectPoint`.
2. Adds the top-right corner `(pageWidth, 0)` for top flips, or both top-right and bottom-right for bottom flips.
3. Includes `sideIntersectPoint` if it exists and is at least 10px from `topIntersectPoint` (prevents degenerate polygons).
4. Adds `bottomIntersectPoint`.
5. Closes with `topIntersectPoint` again.

**Flip Progress (`getFlippingProgress() -> number`):**
- Formula: `|((position.x - pageWidth) / (2 * pageWidth))| * 100`
- Returns 0% when position is at the page edge, 100% when fully flipped to the opposite edge.

**Shadow Data:**
- `getShadowStartPoint()`: For top corner, returns `topIntersectPoint`. For bottom corner, returns `sideIntersectPoint` if available, otherwise `topIntersectPoint`.
- `getShadowAngle()`: Computes the angle between the shadow line (from shadow start to the next intersection point) and a horizontal reference line. Negated for forward direction (returns `PI - angle`).

**Other Getters:**
- `getAngle()`: Returns the negated angle for forward direction, raw angle for backward.
- `getActiveCorner()`: Returns `rect.topLeft` for forward, `rect.topRight` for backward. This is the visible active corner used for positioning.
- `getBottomPagePosition()`: Returns `(pageWidth, 0)` for backward, `(0, 0)` for forward.

---

### 4.2 Flip/Flip.ts (449 lines)

**Role:** The flip controller and state machine. Orchestrates the interaction between user input, the calculation engine, and the renderer. Determines flip direction, selects pages, manages state transitions, and drives animations.

**Dependencies:** `Render`, `Orientation`, `PageFlip`, `Helper`, `PageRect`, `Point`, `FlipCalculation`, `Page`, `PageDensity`.

**Exported Enums:**

**`FlipDirection` (const enum):** `FORWARD` (0), `BACK` (1).

**`FlipCorner` (const enum):** `TOP` (`'top'`), `BOTTOM` (`'bottom'`).

**`FlippingState` (const enum):** `USER_FOLD` (`'user_fold'`), `FOLD_CORNER` (`'fold_corner'`), `FLIPPING` (`'flipping'`), `READ` (`'read'`).

**Internal State:**
- `flippingPage: Page` -- The page currently being animated.
- `bottomPage: Page` -- The page revealed beneath the flip.
- `calc: FlipCalculation` -- The active calculation object (null when idle).
- `state: FlippingState` -- Current state machine state.

**State Machine:**

```
READ  <-->  FOLD_CORNER    (mouse enters/leaves corner area)
READ  --->  USER_FOLD      (user starts dragging)
READ  --->  FLIPPING       (click-to-flip triggers animation)
FOLD_CORNER --> FLIPPING   (corner peek resolves)
USER_FOLD  -->  FLIPPING   (drag resolves to animation)
FLIPPING   -->  READ       (animation completes)
```

State changes emit `changeState` events via `app.updateState(newState)`.

**`start(globalPos) -> boolean`:** Initialisation method called at the beginning of any flip interaction:
1. Resets internal state (nulls calc, flippingPage, bottomPage).
2. Converts global position to book coordinates.
3. **Direction detection:** In portrait mode, if the touch is within the left 1/5 of the book width (`touchPos.x - rect.pageWidth <= rect.width / 5`), direction is BACK; otherwise FORWARD. In landscape mode, the left half is BACK, right half is FORWARD.
4. **Corner detection:** Above the vertical midpoint is TOP, below is BOTTOM.
5. **Direction validation:** Forward requires `currentPageIndex < pageCount - 1`; backward requires `currentPageIndex >= 1`.
6. Retrieves the flipping page and bottom page from the collection.
7. **Landscape density matching:** If the flipping page and its neighbour have different densities, both are temporarily set to HARD. For backward flips, checks `nextBy(flippingPage)`; for forward, checks `prevBy(flippingPage)`.
8. Creates a new `FlipCalculation` with direction, corner, and dimensions (as strings -- the type-casting workaround).

**`fold(globalPos)`:** Called continuously during drag. Sets state to USER_FOLD, lazily calls `start()` if calc is null, then calls `do()` with the page-local position.

**`flip(globalPos)`:** Triggered by click-to-flip. If `disableFlipByClick` is true, only proceeds if the click is on a corner. Finishes any existing animation, calls `start()`, sets FLIPPING state, then:
1. Computes start and end points for animation: starts 10% from the top/bottom edge, ends at the opposite edge (`-pageWidth`).
2. Calls `animateFlippingTo(start, end, isTurned: true)`.

**`do(pagePos)`:** The per-frame update method:
1. Calls `calc.calc(pagePos)` to compute geometry.
2. Sets the bottom page's area, position, and angle (always 0).
3. Sets the flipping page's area, active corner position, and angle.
4. Computes the hard angle: `(90 * (200 - progress * 2)) / 100`. This maps progress 0% to 90 degrees and progress 100% to -90 degrees. Negated for backward direction.
5. Updates the renderer with page rect and shadow data.

**`animateFlippingTo(start, dest, isTurned, needReset=true)`:**
1. Generates interpolated points via `Helper.GetCordsFromTwoPoint(start, dest)`.
2. Creates a frame array where each frame calls `do(point)`.
3. Computes duration: if >= 1000 frames, use `flippingTime`; otherwise scale proportionally: `(frameCount / 1000) * flippingTime`.
4. Starts the animation on the renderer.
5. Completion callback: if `isTurned`, calls `app.turnToNextPage()` or `app.turnToPrevPage()`. If `needReset`, clears the renderer state and resets to READ.

**`stopMove()`:** Called when the user releases a drag. If the position is at or past the spine (x <= 0), animates to completion. Otherwise, animates back to the starting edge.

**`showCorner(globalPos)`:** Manages the corner fold preview:
1. Only proceeds if in READ or FOLD_CORNER state.
2. If the mouse is on a corner: initialises a new flip if needed, sets FOLD_CORNER state, then animates a small 50px fold (from the edge to 50px inward).
3. If the mouse leaves the corner: resets to READ and finishes any animation.

**`isPointOnCorners(globalPos) -> boolean`:** Determines if a position is within a corner activation zone:
- Operating distance = `sqrt(pageWidth^2 + height^2) / 5` (roughly 20% of the page diagonal).
- Returns true if the book-relative position is within `operatingDistance` of any of the four corners.

**`flipToPage(page, corner)`:** Navigates to a specific page by adjusting the spread index then calling `flipNext` or `flipPrev`.

**`flipNext(corner)` / `flipPrev(corner)`:** Constructs a synthetic global position on the appropriate edge of the book and calls `flip()`.

---

## 5. Rendering Layer

### 5.1 Render/Render.ts (495 lines)

**Role:** Abstract base class for both Canvas and HTML renderers. Manages the animation loop (`requestAnimationFrame`), coordinate system conversions, layout calculation (responsive sizing), shadow state, and page references.

**Dependencies:** `PageFlip`, `Point`, `PageRect`, `RectPoints`, `FlipDirection`, `Page`, `PageOrientation`, `FlipSetting`, `SizeType`.

**Exported Types:**

**`Orientation` (const enum):** `PORTRAIT` (`'portrait'`), `LANDSCAPE` (`'landscape'`).

**Internal Types (not exported):**

**`Shadow`:** `{ pos: Point, angle: number, width: number, opacity: number, direction: FlipDirection, progress: number }` -- All data needed to render drop shadows.

**`AnimationProcess`:** `{ frames: FrameAction[], duration: number, durationFrame: number, onAnimateEnd: () => void, startedAt: number }` -- Describes an in-flight animation.

**Internal State:**
- `leftPage`, `rightPage` -- Static pages on display.
- `flippingPage`, `bottomPage` -- Pages involved in the current flip.
- `direction: FlipDirection` -- Current flip direction.
- `orientation: Orientation` -- Current book orientation.
- `shadow: Shadow` -- Current shadow parameters (null when no shadow).
- `animation: AnimationProcess` -- Current animation (null when idle).
- `pageRect: RectPoints` -- Rotated page corners during flipping.
- `boundsRect: PageRect` -- Cached book dimensions and position (private).
- `timer: number` -- The current `requestAnimationFrame` timestamp.
- `safari: boolean` -- Safari browser detection flag.

**Animation System:**

**`start()`:** Initiates the perpetual render loop. Calls `update()` once, then enters a `requestAnimationFrame` loop that calls `render(timer)` on every frame.

**`render(timer)`:** The main loop body:
1. If an animation is active, computes `frameIndex = round((timer - startedAt) / durationFrame)`.
2. If `frameIndex` is within bounds, executes that frame function.
3. If past the last frame, calls `onAnimateEnd()` and clears the animation.
4. Updates `this.timer` and calls `drawFrame()` (the abstract method implemented by subclasses).

**`startAnimation(frames, duration, onAnimateEnd)`:** Finishes any existing animation, then creates a new `AnimationProcess`. `durationFrame` is `duration / frames.length`. `startedAt` is the current timer value.

**`finishAnimation()`:** Immediately executes the last frame and calls the callback. Used to skip to the end of an animation.

**Layout Calculation (`calculateBoundsRect() -> Orientation`):**

This method computes the book's position and dimensions within its parent container:

1. Gets the container dimensions from the dist element's `offsetWidth` / `offsetHeight`.
2. Computes the aspect ratio from the configured `width / height`.
3. Default `pageWidth` and `pageHeight` are the configured values.
4. For **STRETCH** mode:
   - If container width < `minWidth * 2` and portrait is enabled, switches to PORTRAIT.
   - `pageWidth` = full container width (portrait) or half container width (landscape).
   - Caps `pageWidth` at `maxWidth`.
   - Derives `pageHeight` from aspect ratio (`pageWidth / ratio`).
   - If `pageHeight` exceeds container height, caps it and recalculates `pageWidth`.
5. For **FIXED** mode:
   - If container width < configured `pageWidth * 2` and portrait is enabled, switches to PORTRAIT.
6. `left` position:
   - Landscape: `centerX - pageWidth` (left edge of the book is one pageWidth left of centre).
   - Portrait: `centerX - pageWidth/2 - pageWidth` (shifts an additional `pageWidth` to the left). This creates a hidden left "page" off-screen, maintaining the two-page coordinate space even when only one page is visible.

Returns the calculated `Orientation`.

**`update()`:** Clears the cached `boundsRect`, recalculates it, and if orientation changed, calls `app.updateOrientation(newOrientation)`.

**Shadow Data (`setShadowData`):**
- `width = (pageWidth * 3/4 * progress) / 100` -- Shadow width scales with flip progress.
- `opacity = ((100 - progress) * maxShadowOpacity) / 100 / 100` -- Opacity decreases as the flip progresses.
- `progress` is doubled to a 0-200 scale for use by the hard page shadow system.

**Coordinate Conversions:**

**`convertToBook(pos) -> Point`:** Subtracts `rect.left` and `rect.top` from global coordinates.

**`convertToPage(pos, direction?) -> Point`:** Converts global to page-local:
- Forward: `x = pos.x - rect.left - rect.width/2` (origin at the right page's left edge).
- Back: `x = rect.width/2 - pos.x + rect.left` (x-axis mirrored -- origin at the left page's right edge, with x increasing leftward).
- `y = pos.y - rect.top`.

**`convertToGlobal(pos, direction?) -> Point`:** Inverse of `convertToPage`.

**`convertRectToGlobal(rect, direction?) -> RectPoints`:** Applies `convertToGlobal` to all four corners.

**Safari Detection:** Regex pattern `Version/[\d\.]+.*Safari/` against `navigator.userAgent`. The `isSafari()` method exposes this flag.

**Page Orientation Assignment:**
- `setRightPage(page)`: Sets orientation to RIGHT.
- `setLeftPage(page)`: Sets orientation to LEFT.
- `setBottomPage(page)`: LEFT for backward flip, RIGHT for forward.
- `setFlippingPage(page)`: LEFT for forward flip in landscape, RIGHT otherwise.

---

### 5.2 Render/CanvasRender.ts (167 lines)

**Role:** Canvas 2D implementation of the renderer. Draws pages as bitmap images and renders shadows using Canvas gradient APIs.

**Dependencies:** `Render`, `Orientation`, `PageFlip`, `FlipDirection`, `PageOrientation`, `FlipSetting`.

**Constructor:** Accepts a `PageFlip`, `FlipSetting`, and the `HTMLCanvasElement`. Gets the 2D context.

**`drawFrame()` rendering order:**
1. `clear()` -- Fill entire canvas with white.
2. Draw left page (landscape only, if not null) -- `leftPage.simpleDraw(LEFT)`.
3. Draw right page (if not null) -- `rightPage.simpleDraw(RIGHT)`.
4. Draw bottom page (if not null) -- `bottomPage.draw()`.
5. Draw book spine shadow -- `drawBookShadow()`.
6. Draw flipping page (if not null) -- `flippingPage.draw()`.
7. Draw outer and inner shadows (if shadow data exists).
8. **Portrait clipping:** In portrait mode, clips the right half of the canvas to hide the left "hidden page" area. This is the last operation, applied as a post-processing clip.

**Shadow Methods:**

**`drawBookShadow()`:** A permanent spine shadow at the book centre:
- Width: `rect.width / 20`.
- Position: centred horizontally.
- Gradient: 6 colour stops creating a dark valley at the 50% mark (spine), fading to transparent at edges. The 49-51% stops create a sharp shadow line.

**`drawOuterShadow()`:** Projects from the fold line outward onto the underlying page:
- Translates to the shadow start point (converted to global coords).
- Rotates by `PI + shadow.angle + PI/2`.
- Direction-dependent: forward direction has the opaque end at the start; backward has it at the end.
- Clipped to the book rectangle.

**`drawInnerShadow()`:** Projects onto the surface of the flipping page:
- Width is 3/4 of the outer shadow width.
- Clipped to the rotated page rectangle (not the book rectangle).
- Uses a multi-stop gradient: opaque at the edge (0% or 100%), a dip to 0.05 opacity at 10%/90%, then back to opaque at 30%/70%, fading to transparent. This creates the illusion of a page curving away from the light.
- Direction-dependent: forward translates by `-innerShadowWidth`; backward starts at 0.

**`reload()`:** No-op (Canvas mode doesn't need DOM rebuild).

---

### 5.3 Render/HTMLRender.ts (382 lines)

**Role:** HTML/CSS implementation of the renderer. Uses CSS transforms, clip-path, and dynamically styled `<div>` elements for shadows.

**Dependencies:** `Render`, `Orientation`, `PageFlip`, `FlipDirection`, `PageDensity`, `PageOrientation`, `HTMLPage`, `Helper`, `FlipSetting`.

**Constructor:** Creates four shadow `<div>` elements via `innerHTML`: `.stf__outerShadow`, `.stf__innerShadow`, `.stf__hardShadow`, `.stf__hardInnerShadow`. All are appended to the dist element.

**`drawFrame()` rendering order:**
1. `clear()` -- Hides all pages except the four active ones (left, right, bottom, flipping) by setting `display: none`. Also removes any temporary copies that aren't the current flipping page.
2. `drawLeftPage()` -- Static left page, or during a backward hard-page flip, renders the left page with a hard draw angle of `180 + flippingPage.hardAngle` (showing the back face).
3. `drawRightPage()` -- Same logic as left, but for forward hard-page flips.
4. `drawBottomPage()` -- Draws the revealed page. Skips in portrait mode with backward direction (the bottom page would be off-screen).
5. Draw flipping page with z-index `startZIndex + 5`.
6. Draw shadows: if the flipping page is SOFT, draws outer and inner shadows; if HARD, draws hard outer and inner shadows.

**Hard Page Back-Face Rendering:** When a hard page is flipping backward, the left (static) page is drawn using `drawHard` with angle `180 + hardAngle`. This creates the back face of the cover. Similarly for forward flips with the right page. The z-index is set to `startZIndex + 5` to appear above the bottom page.

**Shadow Methods:**

**`drawHardInnerShadow()`:** Shadow on the inside of a hard page:
- Progress is normalised: if > 100 (past halfway), uses `200 - progress`.
- Shadow size: `((100 - progress) * 2.5 * pageWidth) / 100 + 20`, capped at `pageWidth`.
- Gradient: `linear-gradient(to right, rgba(0,0,0, opacity*progress/100) 5%, transparent 100%)`.
- Origin at the book spine (`rect.left + rect.width/2`).
- Flips via `rotateY(180deg)` depending on direction and progress (past/before halfway).

**`drawHardOuterShadow()`:** Shadow cast by a hard page:
- Same size calculation as inner.
- Gradient: `linear-gradient(to left, rgba(0,0,0, opacity) 5%, transparent 100%)`.
- Flips direction opposite to the inner shadow.

**`drawInnerShadow()`:** Shadow on a soft flipping page:
- Width: `shadow.width * 3/4`.
- Constructs a clip-path polygon from the four rotated page corners, transformed relative to the shadow position and rotated by `shadow.angle + 3*PI/2`.
- Gradient: multi-stop (5%: opaque, 15%: 0.05, 35%: opaque, 100%: transparent) -- matches the Canvas version's visual effect.
- Transform: `translate3d(shadowPos.x, shadowPos.y - 100, 0) rotate(angle rad)` with transform-origin at the shadow translate offset.

**`drawOuterShadow()`:** Shadow cast by a soft page onto the page beneath:
- Clip-path polygon is the full page rectangle `(0,0) (pageWidth,0) (pageWidth,pageHeight) (0,pageHeight)`, transformed and rotated.
- Simple two-stop gradient.
- Same transform approach as inner shadow.

**`update()`:** Extends the base `update()` by re-setting page orientations (RIGHT for rightPage, LEFT for leftPage).

**`reload()`:** Re-creates shadow elements if they were removed from the DOM (happens after page updates).

**`clearShadow()`:** Hides all four shadow elements via `display: none`.

---

## 6. UI Layer

### 6.1 UI/UI.ts (286 lines)

**Role:** Abstract base class handling all user interaction. Creates the DOM wrapper structure, registers mouse/touch event listeners, implements swipe detection, and manages responsive sizing.

**Dependencies:** `PageFlip`, `Point`, `FlipSetting`, `SizeType`, `FlipCorner`, `FlippingState`, `Orientation`.

**DOM Structure Created:**
```
inBlock (user-provided)
  +-- .stf__parent (class added to inBlock)
      +-- .stf__wrapper (created by UI)
          +-- [canvas or .stf__block] (created by subclass)
```

**Constructor:** Sets up the wrapper, applies min-width/min-height styles based on settings, and registers the resize handler.

Sizing logic:
- Portrait factor `k` = 1 if `usePortrait` is true, 2 otherwise.
- `minWidth = setting.minWidth * k`.
- Fixed mode: `minWidth = setting.width * k`.
- Auto-size: sets `width: 100%` and `maxWidth: setting.maxWidth * 2`.

**Event Handler Registration (`setHandlers()`):**
- `window.resize` -> `onResize` (always registered).
- If `useMouseEvents` is false, no other handlers are registered.
- `distElement.mousedown` -> `onMouseDown`.
- `distElement.touchstart` -> `onTouchStart`.
- `window.mousemove` -> `onMouseMove`.
- `window.touchmove` -> `onTouchMove` (passive flag = `!mobileScrollSupport`; passive when mobile scroll is disabled).
- `window.mouseup` -> `onMouseUp`.
- `window.touchend` -> `onTouchEnd`.

Note: Move and up/end events are on `window`, not the element. This ensures flipping continues even if the pointer leaves the book area.

**Coordinate Conversion (`getMousePos(x, y) -> Point`):** Uses `distElement.getBoundingClientRect()` to convert client coordinates to element-relative coordinates.

**Click Event Forwarding (`checkTarget(target) -> boolean`):** If `clickEventForward` is enabled, returns `false` for `<a>` and `<button>` elements (preventing flip capture), `true` for everything else.

**Mouse Event Handlers:**
- **`onMouseDown`:** Converts to relative coords, calls `app.startUserTouch(pos)`, prevents default.
- **`onMouseMove`:** Converts to relative coords, calls `app.userMove(pos, isTouch: false)`.
- **`onMouseUp`:** Converts to relative coords, calls `app.userStop(pos)`. No swipe flag (mouse never triggers swipe).

**Touch Event Handlers:**

**`onTouchStart`:**
1. Records the touch point and timestamp in `touchPoint`.
2. Schedules a deferred `startUserTouch` call after 250ms. If the touch resolves as a swipe within that window, `touchPoint` is cleared and the deferred call is cancelled.
3. Prevents default only if `mobileScrollSupport` is disabled.

**`onTouchMove`:**
- With `mobileScrollSupport` enabled:
  - Only triggers `userMove` if horizontal movement exceeds 10px (a non-configurable dead zone) or the flip state is not READ.
  - Calls `preventDefault()` only when the flip state is not READ (allows vertical scrolling when idle).
- Without `mobileScrollSupport`: always calls `userMove`.

**`onTouchEnd` -- Swipe Detection Algorithm:**
1. Compute `dx = endPos.x - startPos.x` and `distY = |endPos.y - startPos.y|`.
2. **Swipe conditions (all must be true):**
   - `|dx| > swipeDistance` (default 30px) -- sufficient horizontal movement.
   - `distY < swipeDistance * 2` (default 60px) -- limited vertical deviation.
   - `Date.now() - touchPoint.time < 250ms` -- completed quickly.
3. If swipe is detected: `dx > 0` triggers `flipPrev`, `dx < 0` triggers `flipNext`. Corner is TOP if the original touch was above the vertical midpoint, BOTTOM otherwise.
4. Clears `touchPoint` regardless.
5. Calls `app.userStop(pos, isSwipe)`.

**Orientation Styling (`setOrientationStyle(orientation)`):**
- Adds `--portrait` or `--landscape` class to the wrapper.
- If `autoSize` is true, sets `paddingBottom` to maintain aspect ratio:
  - Portrait: `(height / width) * 100%`.
  - Landscape: `(height / (width * 2)) * 100%`.
- Calls `update()`.

---

### 6.2 UI/CanvasUI.ts (43 lines)

**Role:** Canvas mode UI implementation. Creates a `<canvas>` element inside the wrapper.

**Dependencies:** `UI`, `PageFlip`, `FlipSetting`.

**Constructor:** Sets `wrapper.innerHTML` to a `<canvas class="stf__canvas">`. Sets `distElement` to the canvas. Calls `resizeCanvas()` and `setHandlers()`.

**`resizeCanvas()`:** Reads the canvas's computed CSS width/height (via `getComputedStyle`) and sets the canvas's logical `width`/`height` attributes to match. This ensures the canvas rendering resolution matches its display size.

**`update()`:** Resizes the canvas and calls `app.getRender().update()`.

---

### 6.3 UI/HTMLUI.ts (59 lines)

**Role:** HTML mode UI implementation. Creates a `.stf__block` container and reparents page elements into it.

**Dependencies:** `UI`, `PageFlip`, `FlipSetting`.

**Constructor:**
1. Inserts `<div class="stf__block"></div>` into the wrapper.
2. Sets `distElement` to this block.
3. Moves all provided page elements into the block via `appendChild`.
4. Calls `setHandlers()`.

**`updateItems(items)`:** For page replacement:
1. Removes all event handlers.
2. Clears `distElement.innerHTML`.
3. Moves new items into the block.
4. Re-registers event handlers.

**`clear()`:** Returns all page elements to the original parent element (restoring the DOM to its pre-initialisation state).

**`update()`:** Calls `app.getRender().update()`.

---

## 7. Facade

### 7.1 PageFlip.ts (399 lines)

**Role:** The sole public-facing class. Orchestrates all subsystems (UI, Render, Flip, PageCollection) behind a clean consumer API. Extends `EventObject` for event support.

**Dependencies:** All modules (this is the integration point).

**CSS Import:** `import './Style/stPageFlip.css'` -- The CSS file is imported here and bundled into the JavaScript output by the build system (Rollup postcss plugin).

**Constructor:** `new PageFlip(inBlock: HTMLElement, setting: Partial<FlipSetting>)`. Processes settings via `Settings.getSettings()` and stores the root element.

**Initialisation Methods:**

**`loadFromImages(imagesHref[])`:**
1. Creates `CanvasUI` (which creates the canvas element).
2. Creates `CanvasRender` with the canvas.
3. Creates `Flip` controller.
4. Creates `ImagePageCollection`, loads pages.
5. Starts the render loop.
6. Shows the start page.
7. After a 1ms `setTimeout` (Safari fix), calls `ui.update()` and triggers the `init` event.

**`loadFromHTML(items)`:** Same flow but with `HTMLUI`, `HTMLRender`, and `HTMLPageCollection`.

**Update Methods:**

**`updateFromImages(imagesHref[])`:** Destroys the old collection, creates a new one, loads, and shows the same page index. Triggers `update` event.

**`updateFromHtml(items)`:** Same, but also calls `(ui as HTMLUI).updateItems(items)` to update the DOM and `render.reload()` to recreate shadow elements.

**Cleanup:** `destroy()` calls `ui.destroy()` then removes the root element. `clear()` destroys pages and restores DOM via `(ui as HTMLUI).clear()`.

**Navigation (no animation):** `turnToPrevPage()`, `turnToNextPage()`, `turnToPage(page)` -- delegate directly to the page collection.

**Navigation (with animation):** `flipNext(corner)`, `flipPrev(corner)`, `flip(page, corner)` -- delegate to the flip controller.

**User Interaction Bridge:**

**`startUserTouch(pos)`:** Records the mouse position, sets `isUserTouch = true`, `isUserMove = false`.

**`userMove(pos, isTouch)`:**
- If not touching and not a touch event and `showPageCorners` is enabled: calls `flipController.showCorner(pos)` for the hover preview.
- If touching and distance from initial position > 5px: sets `isUserMove = true` and calls `flipController.fold(pos)` for drag-folding.

**`userStop(pos, isSwipe = false)`:**
- If was touching and not a swipe: if didn't move, calls `flipController.flip(pos)` (click-to-flip); if moved, calls `flipController.stopMove()` (resolve drag).
- If swipe, does nothing extra (swipe already triggered `flipNext`/`flipPrev`).

**Event Triggers:** `updateState(newState)`, `updatePageIndex(newPage)`, `updateOrientation(newOrientation)` -- called by subsystems to emit events through the facade.

**State Queries:** All delegate to the appropriate subsystem (`getPageCount`, `getCurrentPageIndex`, `getPage`, `getRender`, `getFlipController`, `getOrientation`, `getBoundsRect`, `getSettings`, `getUI`, `getState`, `getPageCollection`).

---

## 8. Styles

### 8.1 Style/stPageFlip.css (60 lines)

**Role:** Provides the structural CSS foundation for both rendering modes. Establishes the layout, 3D context, and shadow element positioning.

**Classes:**

**`.stf__parent`:**
- `position: relative` -- Positioning context for all children.
- `display: block`, `box-sizing: border-box`.
- `transform: translateZ(0)` -- Triggers GPU compositing (new stacking context).
- `touch-action: pan-y` (also `-ms-touch-action`) -- Tells the browser to only handle vertical scrolling, allowing horizontal touches to be captured by the library.

**`.sft__wrapper`:** (**Note: this is a typo** -- the CSS uses `sft` but the JavaScript creates elements with class `stf__wrapper`. The CSS rule will not match the generated DOM. This is a bug in the original code.)
- `position: relative`, `width: 100%`, `box-sizing: border-box`.
- In responsive mode, JavaScript injects `paddingBottom` to maintain aspect ratio.

**`.stf__parent canvas`:**
- `position: absolute`, filling the parent (`width: 100%`, `height: 100%`, `left: 0`, `top: 0`).
- CSS dimensions control the display size; JavaScript sets the logical canvas resolution from computed styles.

**`.stf__block`:**
- `position: absolute`, `width: 100%`, `height: 100%`, `box-sizing: border-box`.
- `perspective: 2000px` -- Establishes the 3D rendering context for hard page rotateY transforms. This value determines how dramatic the perspective distortion appears (larger = more subtle, smaller = more dramatic).

**`.stf__item`:**
- `display: none` -- Hidden by default; the renderer controls visibility.
- `position: absolute`.
- `transform-style: preserve-3d` -- Ensures child elements and transforms participate in the 3D context.

**`.stf__outerShadow`, `.stf__innerShadow`, `.stf__hardShadow`, `.stf__hardInnerShadow`:**
- All: `position: absolute`, `left: 0`, `top: 0`.
- Positioned at the origin; JavaScript dynamically controls their actual position, size, gradient, rotation, clip-path, z-index, and visibility.

---

## 9. Cross-Cutting Concerns

### 9.1 Known Issues in the Original

1. **CSS class name typo:** `stPageFlip.css` line 11 defines `.sft__wrapper` but all JavaScript code creates/queries `.stf__wrapper`. The wrapper element receives no CSS styling from the shipped stylesheet.

2. **Type-casting workaround:** `FlipCalculation` constructor accepts `pageWidth` and `pageHeight` as strings and parses them with `parseInt`. The comment in `Flip.ts` line 163 says "fix bug with type casting". This indicates that under certain conditions, these values arrived as strings rather than numbers.

3. **Error swallowing:** `Flip.start()` wraps the page selection logic in try/catch and returns false on any error. `FlipCalculation.calc()` similarly catches all exceptions and returns false. `Flip.flipToPage()` has an empty catch block. This prevents crashes but also hides bugs.

4. **`Settings.getSettings()` mutates defaults:** Using `Object.assign(result, userSetting)` on the internal `_default` object means the defaults are permanently modified after the first call.

5. **Safari detection regex:** The regex `Version/[\d\.]+.*Safari/` may not match all Safari versions and could produce false positives on other WebKit browsers.

### 9.2 Dependency Graph

```
PageFlip (facade)
  |-- EventObject (extends)
  |-- Settings
  |-- UI (CanvasUI | HTMLUI)
  |   |-- FlipSetting, FlipCorner, FlippingState, Orientation
  |-- Render (CanvasRender | HTMLRender)
  |   |-- FlipSetting, SizeType, FlipDirection, Page, PageOrientation
  |   |-- HTMLRender uses Helper, HTMLPage
  |-- Flip
  |   |-- FlipCalculation
  |   |   |-- Helper
  |   |   |-- BasicTypes (Point, Rect, RectPoints, Segment)
  |   |-- Render, PageFlip, Page, PageDensity
  |-- PageCollection (HTMLPageCollection | ImagePageCollection)
  |   |-- Render, Page, PageFlip, FlipDirection
  |-- Page (HTMLPage | ImagePage)
  |   |-- Render, BasicTypes
  |-- Helper
  |-- BasicTypes
  |-- Style/stPageFlip.css (imported, bundled)
```
