# StPageFlip - Project Context

## Overview

StPageFlip is a TypeScript library (v2.0.7, MIT license) that creates realistic page-turning effects for web-based book/magazine viewers. Originally authored by Oleg Litovski (Nodlik). It supports two rendering modes: **Canvas** (image-based pages) and **HTML** (DOM element-based pages).

## Architecture Summary

The codebase follows a modular OOP architecture with these core subsystems:

| Subsystem | Directory | Purpose |
|-----------|-----------|---------|
| **Entry Point** | `src/PageFlip.ts` | Main facade class; orchestrates all subsystems |
| **Flip Engine** | `src/Flip/` | Page-turning state machine + trigonometric calculations |
| **Rendering** | `src/Render/` | Abstract renderer with Canvas and HTML implementations |
| **Pages** | `src/Page/` | Page abstraction (HTMLPage, ImagePage) with density/orientation |
| **Collections** | `src/Collection/` | Page management, spread logic, navigation |
| **UI Layer** | `src/UI/` | DOM event handling (mouse, touch, swipe detection) |
| **Events** | `src/Event/` | Simple pub/sub event system |
| **Math Helpers** | `src/Helper.ts` | Geometry utilities (rotation, intersection, distance) |
| **Config** | `src/Settings.ts` | Settings with defaults and validation |
| **Types** | `src/BasicTypes.ts` | Point, Rect, RectPoints, Segment, PageRect |
| **Styles** | `src/Style/` | Base CSS for layout, shadows, 3D transforms |

## Key Design Patterns

- **Facade**: `PageFlip` class is the sole public API
- **Strategy**: Render (Canvas vs HTML), Page (Image vs HTML), UI (Canvas vs HTML)
- **Template Method**: Abstract `Render.drawFrame()` implemented by subclasses
- **State Machine**: `FlippingState` enum drives flip lifecycle (READ -> USER_FOLD/FOLD_CORNER -> FLIPPING -> READ)
- **Observer/Pub-Sub**: `EventObject` base class for event emission

## Build System

- **Rollup** produces UMD (`dist/js/page-flip.browser.js`) and ES module (`dist/js/page-flip.module.js`)
- **Webpack** config exists for development (watch mode)
- Entry point: `src/PageFlip.ts`; exported under `St` namespace (UMD)

## Critical Algorithms

1. **Flip Calculation** (`FlipCalculation.ts`): Computes page rotation angle, clipping polygons, and shadow geometry using trigonometry. The page corner is constrained to a circle (page diagonal radius) to prevent impossible positions.
2. **Animation System** (`Render.ts`): Frame-based animation using `requestAnimationFrame`. Pre-computes interpolated points between start/end positions, then replays them at calculated frame durations.
3. **Coordinate Systems**: Three coordinate spaces - window (global), book (relative to book rect), and page (relative to active page). Conversion methods handle transforms between them.

## Documentation Task

We are creating comprehensive documentation to serve as the basis for a from-scratch rewrite:
- `docs/01-core-principles.md` - Core parts and principles
- `docs/02-compiled-deliverable.md` - What the compiled output does and how
- `docs/03-reverse-engineered-files.md` - Detailed file-by-file analysis
- `docs/04-agency-brief.md` - Web agency style brief
- `docs/05-scope-of-work.md` - Scope of work for rewrite

## Branch

All work on branch: `claude/pageflip-documentation-GYU6Q`
