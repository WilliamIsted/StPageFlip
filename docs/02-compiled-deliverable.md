# StPageFlip - Compiled Deliverable Analysis

## 1. What the Compiled Output Is

The build produces two JavaScript bundles from the TypeScript source:

| File | Format | Global Name | Size | Use Case |
|------|--------|-------------|------|----------|
| `dist/js/page-flip.browser.js` | UMD | `St` | Minified | Script tag inclusion, CommonJS, AMD |
| `dist/js/page-flip.module.js` | ES Module | N/A | Minified | Modern bundlers (Webpack, Rollup, Vite) |

Both are built by Rollup with the `rollup-plugin-terser` minifier and `rollup-plugin-typescript2` for compilation. CSS from `src/Style/stPageFlip.css` is inlined into the bundle via `rollup-plugin-postcss`.

The `package.json` declares:
- `"main": "dist/js/page-flip.browser.js"` (Node/CommonJS resolution)
- `"browser": "dist/js/page-flip.browser.js"` (Bundler browser field)

TypeScript declaration files (`.d.ts`) are emitted to `dist/` via `declarationDir` in `tsconfig.json`.

## 2. What It Does (Consumer Perspective)

### 2.1 Primary Capability

The library turns a collection of content (images or HTML elements) into an interactive book with realistic page-turning animations. A consumer provides:

1. A root HTML element (the book container)
2. Configuration (dimensions, behaviour flags)
3. Content (image URLs or child HTML elements)

The library then:

- Renders a two-page spread (landscape) or single page (portrait) view
- Responds to mouse clicks, drags, touch gestures, and swipes to flip pages
- Shows a subtle corner-fold preview when the mouse hovers near page corners
- Animates page turns with curved page geometry and dynamic shadows
- Automatically switches between portrait and landscape based on container width
- Emits events for page changes, state transitions, and orientation changes

### 2.2 Two Usage Modes

**Canvas Mode** (image-based):
```javascript
const pageFlip = new St.PageFlip(document.getElementById('book'), {
    width: 400,
    height: 600,
    showCover: true
});
pageFlip.loadFromImages(['/pages/1.jpg', '/pages/2.jpg', ...]);
```

**HTML Mode** (DOM-based):
```javascript
const pageFlip = new St.PageFlip(document.getElementById('book'), {
    width: 400,
    height: 600,
    size: 'stretch'
});
pageFlip.loadFromHTML(document.querySelectorAll('.page'));
```

### 2.3 Public API Surface

**Initialisation:**
- `new PageFlip(element, settings)` - Create instance
- `loadFromImages(urls[])` - Load in Canvas mode
- `loadFromHTML(elements)` - Load in HTML mode
- `updateFromImages(urls[])` - Replace pages (Canvas)
- `updateFromHtml(elements)` - Replace pages (HTML)
- `destroy()` - Tear down and remove DOM
- `clear()` - Remove pages, return to initial state

**Navigation (no animation):**
- `turnToPage(pageNum)` - Jump to specific page
- `turnToNextPage()` - Go to next page
- `turnToPrevPage()` - Go to previous page

**Navigation (with animation):**
- `flip(pageNum, corner?)` - Animated flip to page
- `flipNext(corner?)` - Animated flip forward
- `flipPrev(corner?)` - Animated flip backward

**State queries:**
- `getPageCount()` - Total pages
- `getCurrentPageIndex()` - Current page (0-based)
- `getPage(index)` - Get page object
- `getOrientation()` - Portrait or landscape
- `getBoundsRect()` - Current size/position
- `getSettings()` - Configuration object
- `getState()` - Current flipping state
- `getRender()` - Render object
- `getFlipController()` - Flip controller
- `getUI()` - UI object
- `getPageCollection()` - Page collection

**Events:**
- `on(eventName, callback)` - Subscribe
- `off(eventName)` - Unsubscribe

**Event names:** `init`, `flip`, `changeState`, `changeOrientation`, `update`

## 3. How It Achieves the Page-Flip Effect

### 3.1 The Rendering Loop

On initialisation, `Render.start()` kicks off a perpetual `requestAnimationFrame` loop:

```
requestAnimationFrame -> render(timer) -> drawFrame() -> requestAnimationFrame
```

Every frame:
1. If an animation is active, the current frame index is computed from elapsed time
2. The corresponding frame function is executed (which updates page positions/angles)
3. `drawFrame()` is called - the abstract method overridden by Canvas or HTML renderers
4. The loop continues indefinitely

### 3.2 Canvas Rendering Pipeline (per frame)

1. Clear the entire canvas with white
2. Draw the **left static page** (if landscape mode) - simple `drawImage` at position
3. Draw the **right static page** - simple `drawImage` at position
4. Draw the **bottom page** (the page being revealed) - clipped by a polygon, positioned
5. Draw the **book spine shadow** - a vertical linear gradient at the centre
6. Draw the **flipping page** - clipped by its polygon, rotated by the flip angle
7. Draw the **outer shadow** - a linear gradient projected from the fold line outward
8. Draw the **inner shadow** - a linear gradient on the flipping page's surface

All clipping uses Canvas `beginPath()` / `lineTo()` / `clip()`. Shadows use `createLinearGradient()` with `translate()` and `rotate()` transforms.

### 3.3 HTML Rendering Pipeline (per frame)

1. **Clear**: Hide all page elements except the four active ones (left, right, bottom, flipping)
2. **Left page**: Position absolutely with `left`/`top`/`width`/`height` styles
3. **Right page**: Same, offset by `pageWidth`
4. **Bottom page**: Positioned with a z-index below the flipping page
5. **Flipping page (soft)**: CSS `transform: translate3d() rotate()` with `clip-path: polygon()` computed from the flip calculation's clipping area
6. **Flipping page (hard)**: CSS `transform: translate3d() rotateY()` with `backface-visibility: hidden`
7. **Shadows**: Four dedicated `<div>` elements (`.stf__outerShadow`, `.stf__innerShadow`, `.stf__hardShadow`, `.stf__hardInnerShadow`) styled with `linear-gradient` backgrounds, positioned and rotated via CSS transforms, clipped with `clip-path: polygon()`

### 3.4 The Flip Calculation

When the user interacts with the book, the `Flip` controller determines:

1. **Direction**: Forward (right-to-left) or backward (left-to-right), based on which half of the book was touched
2. **Corner**: Top or bottom, based on whether the touch is above or below the midpoint
3. **Which pages**: The flipping page and the page underneath are retrieved from the collection

Then, for each frame of interaction or animation, `FlipCalculation.calc(pos)` computes:

- **Rotation angle**: Using inverse cosine of the horizontal distance ratio
- **Constrained position**: The touch point is limited to a circle (radius = page width) to prevent over-extension
- **Rotated rectangle**: All four corners of the page are rotated around the touch point
- **Intersection points**: Where the rotated page edges cross the book boundaries (top, right side, bottom)
- **Clipping polygons**: Derived from the intersection points for both the flipping page and bottom page
- **Shadow data**: Start point, angle, width, and opacity based on flip progress

### 3.5 Animation Interpolation

When a flip is triggered (by click or drag release):

1. Start point and destination point are defined (e.g., right edge to left edge for forward flip)
2. `Helper.GetCordsFromTwoPoint()` generates an array of intermediate points, stepping 1 pixel at a time along the longest axis
3. Each point becomes a frame closure
4. Animation duration is proportional to the number of frames, capped at `flippingTime`
5. On each `requestAnimationFrame`, the elapsed time determines which frame to play
6. On completion, the callback finalises the page turn (updating the current spread index)

### 3.6 Shadow System

Shadows are composed of multiple layers to simulate light interaction:

**Book spine shadow**: A permanent vertical gradient at the centre of the book, simulating the dip at the binding.

**Outer shadow**: Projects outward from the fold line onto the page beneath. Width and opacity scale with flip progress - widest and most transparent at the start, narrowest and most opaque near completion.

**Inner shadow**: Projects onto the flipping page's surface from the fold line. Creates the illusion of the page curving away from a light source. Uses a multi-stop gradient with opacity peaks at the edges and a lighter centre.

**Hard page shadows**: Simplified shadows for rigid pages, using horizontal linear gradients that flip direction based on progress past the halfway point (>100 in the 0-200 progress scale).

## 4. CSS Foundation

The bundled CSS (`stPageFlip.css`) provides the structural foundation:

- `.stf__parent`: Root container with `touch-action: pan-y` (allows vertical scroll, captures horizontal gestures), GPU-composited via `translateZ(0)`
- `.stf__block`: Absolute-positioned page container with `perspective: 2000px` for 3D CSS transforms
- `.stf__item`: Hidden by default, absolute-positioned with `transform-style: preserve-3d` for hard page flips
- Shadow elements: All absolute-positioned at `(0,0)`, dynamically styled by the renderer

## 5. Browser Compatibility

- The code detects Safari specifically (`Version/[\d\.]+.*Safari/` regex) to work around a `clip-path` bug (WebKit bug #126207)
- In Safari, soft page transforms use `translate()` instead of `translate3d()` when the angle is zero
- The library uses `transform3d` heavily for GPU acceleration on non-Safari browsers
- Touch event handling includes passive listener detection for `touchmove` (controlled by `mobileScrollSupport`)
- The `clickEventForward` setting allows links and buttons inside pages to function normally by skipping event capture for `<a>` and `<button>` elements
