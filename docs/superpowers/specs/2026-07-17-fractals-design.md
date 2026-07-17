# Fractal Explorer — Design Spec

**Date:** 2026-07-17  
**Status:** Approved for planning  
**File:** `fractals.html` (standalone, same format as `index.html` / `galaxies.html`)

## Goal

A Mandelbrot / Julia escape-time explorer in the same single-file WebGPU style as Color Gravity and Galaxy Collision: dark panel UI, sliders, pan/zoom, with interactive performance as a first-class constraint.

## Decisions (from brainstorming)

| Topic | Choice |
|-------|--------|
| Fractal family | Escape-time only |
| Formulas (v1) | Mandelbrot + Julia |
| Interaction | Pan + wheel zoom; click Mandelbrot → Julia with that point as `c` |
| Zoom depth | Comfortable float32 now; architecture ready for deep zoom later |
| GPU architecture | Compute escape → storage texture → colorize blit |

## Architecture

### Envelope

- Single HTML file: `fractals.html`
- Same CSS tokens / layout vocabulary as `galaxies.html` (`--bg`, `--panel`, `--accent`, control groups, `#viewPanel` zoom)
- Nav links between tools (`index.html` ↔ `galaxies.html` ↔ `fractals.html`)
- WebGPU only; clear fallback message if unavailable

### Pipeline

```
uniforms (center, scale, maxIter, mode, c, color params, render size)
        │
        ▼
 compute: per-pixel smooth escape  →  r32float texture
        │
        ▼
 fragment blit: escape → colormap → canvas
```

### Modules (logical units in one file)

| Unit | Responsibility |
|------|----------------|
| `viewState` | center, scale; pan/zoom math (cursor-anchored zoom) |
| `fractalParams` | mode, `c`, maxIter, colormap preset / phase |
| `qualityController` | draft vs final resolution, iter budget, idle promote |
| `gpuEngine` | device init, buffer/texture lifecycle, compute + blit |
| UI wire-up | sliders ↔ state ↔ `requestRender` |

Each unit should be understandable without reading the others’ internals; GPU iterate stage must stay swappable for future perturbation without rewriting colorize/UI.

## Controls

| Group | Controls |
|-------|----------|
| Mode | Select Mandelbrot / Julia ; Reset view |
| Julia | `Re(c)`, `Im(c)` sliders (active in Julia; updated on Mandelbrot click) |
| Iter | `maxIter` (UI range ~50–2000); runtime may soft-cap while drafting |
| Color | 2–3 palette presets (e.g. fire, ocean, mono); phase / escape-power offset |
| View | Vertical zoom slider (galaxies pattern); read-only center / scale readout |

**Canvas gestures**

- Drag → pan
- Wheel → zoom toward cursor
- Click (Mandelbrot mode) → switch to Julia, set `c` to clicked complex point

## Performance

Hard rules:

1. Escape iteration runs entirely on GPU; CPU only uploads uniforms and handles input.
2. During pan / zoom / slider drag: render at ½ or ¼ resolution and soft-cap `maxIter`.
3. After idle (~80–120 ms): full resolution, user `maxIter`, optional light 2× supersample.
4. Coalesce GPU work to one in-flight frame via `requestAnimationFrame` (no queue pile-up while sliders spam `input`).
5. Discrete draft/final (or FPS) indicator so quality downgrade is visible.

Anti-blowup:

- UI + runtime clamp on `maxIter`
- No float64 / perturbation in v1
- Soft warning when float32 precision limit is approached (extreme zoom), rather than silent pixel garbage
- Canvas resize recreates escape texture / related resources (no leaks)

## Edge cases

- WebGPU missing → fallback message (same tone as existing tools)
- Extreme `|c|` in Julia → dark / empty image is acceptable
- Rapid resize → recreate textures safely
- Mode switch Mandelbrot ↔ Julia keeps sensible view (reset optional via button)

## Out of scope (v1)

- Deep zoom (perturbation / series approximation)
- Burning Ship, Newton, other formulas
- PNG export, shareable URL state
- WebGL fallback
- Advanced mobile gestures beyond basic pan/zoom

## Future hook (not built now)

Keep the compute “iterate” stage behind a narrow interface (uniforms in → escape texture out) so a later deep-zoom path can inject a CPU reference orbit + delta iteration without changing colorize or the control chrome.

## Success criteria

- Feels as responsive as galaxies while dragging (draft quality OK)
- Settles to a sharp full-res frame shortly after interaction stops
- Mandelbrot → click → Julia with matching `c` works in one gesture
- Same visual/UX family as the other tools in the repo
- No browser freeze under aggressive slider / zoom spam
