# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Single-file personal website with a Three.js + custom GLSL gravitational lensing / black hole starfield background. No build system, no dependencies beyond a CDN-hosted Three.js 0.157.

## How to develop

Open `index.html` directly in a browser, or serve it:

```bash
npx serve .          # then open http://localhost:3000
```

No build, no lint, no tests — this is a pure static page.

## Architecture

Everything lives in `index.html`: CSS (~20 lines), HTML skeleton (~5 lines), and the JS+GLSL application (~300 lines).

### Shader pipeline (in order, within `main()`)

1. **`renderBackground(uv)`** — dark space + FBM nebula layers (gas/dust/emission)
2. **`renderStars(uv, time)`** — 7-octave procedural star field with multi-temperature color (orange → white → blue)
3. **`lensDeflect(px, bh, r)`** — Schwarzschild weak-field deflection `~1/d` + soft shadow falloff; returns `(deflect, shadow)`
4. **`renderLens(uv, px, bh, r, lensStrength)`** — applies deflection vector to UV coordinates for lensed sampling
5. **`renderEventHorizon(px, bh, r)`** — 3D sphere with Fresnel rim light, ambient occlusion, and specular
6. **`renderPhotonRing(px, bh, r, time)`** — three-layer ring (inner white, mid blue-white, outer glow) with cinematic bloom
7. **`renderDisk(px, bh, r, time, mouse, lensBoost)`** — accretion disk with procedural turbulence, Doppler asymmetry, front disk + reverse-ray-traced back disk
8. **`toneMapping(color)`** — relaxed ACES approximation (not exact ACES, tuned for cinematic look)
9. **`postProcess(color, uv, time)`** — chromatic aberration, film grain, vignette

### JS-side state

- **Black hole position**: autonomous drift (sin/cos of `camDrift` accumulators) + gentle mouse attraction; clamped to viewport edges
- **Entry animation**: ease-out cubic (`1-(1-t)³`) over ~3.2s growing the BH radius from 0 to target
- **Mouse tracking**: `pointermove` / `pointerleave` on the canvas; `mouse.y` is in CSS pixels (0 = top) in `uMouse`, but BH uniforms use Three.js convention (y-up)
- **Reset**: `R` key recenters BH to `(50%, 45%)`
- **Uniform `uBH`**: Three.js coordinate space — `y` is flipped from CSS (`RE.y - bh.y`)
- **Uniform `uMouse`**: `(mouse.x / innerWidth, mouse.y)` — x normalized, y in CSS pixels

## Key constants

| Constant | Value | Location |
|---|---|---|
| Target BH radius | `Math.min(w,h) * 0.06` | JS `TR` (line 301) |
| Lens influence radius | `r * 12.0` | FS `infR` (line 269) |
| Shadow inner | `r * 0.35` | `lensDeflect` |
| Shadow outer | `r * 3.2` | `lensDeflect` |
| Photon ring radius | `r * 1.5` | `renderPhotonRing` |
| Disk inner/outer | `r*2.2` / `r*6.5` | `renderDisk` |
| Star layer count | 7 | `renderStars` loop |
| FBM octaves | 4 | `fbm()` loop |
| Canvas texture | 2048×512 | `tc` (line 40) |

## Known bugs & pitfalls

### GLSL NaN propagation through `smoothstep(a, a, x)`

When `r == 0` (initial state before entry animation), `smoothstep(0, 0, dist)` returned NaN on some GPUs. The NaN propagated through `mix()`, then to `gl_FragColor`, causing a fully black screen. The fix was guarding with `if (influenceR > 0.5)` before calling `smoothstep`. This pattern should be checked for every `smoothstep` call where edge0 and edge1 could ever be equal.

### Other GLSL undefined behaviors to watch for

- `smoothstep(a, a, x)` — undefined when edges equal
- `normalize(vec2(0))` — produces NaN
- Division by zero in any expression
- `atan(0, 0)` — undefined

### Coordinate space mismatch

`uBH` uses Three.js y-up coordinates (`RE.y - bh.y`), while `uMouse.y` is in CSS pixels (0 = top). Be careful when passing mouse-relative coordinates into shader functions that operate on `uBH` space.

## UI summary

- Fixed navbar with glassmorphism effect (`backdrop-filter: blur`)
- "移动鼠标探索 · 按 R 重置" hint at bottom (replaced by GL error text if shader compile fails)
- Responsive: mobile breakpoint at 640px
- Font stack: native system fonts with Chinese fallbacks (PingFang SC, Microsoft YaHei)
