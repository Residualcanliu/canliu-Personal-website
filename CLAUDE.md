# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Single-file personal website (v1.2) — Three.js + custom GLSL black hole / gravitational lensing background with four real constellation star charts at screen corners. Five-page panel navigation system. No build system, no framework.

## How to develop

```bash
npx serve .          # then open http://localhost:3000
```

Or open `index.html` directly. No build, no lint, no tests.

## Architecture

`index.html` contains everything: CSS, HTML, JS navigation, Three.js rendering, GLSL shaders (~450 lines total).

### Shader pipeline (render order in `main()`)

1. `renderBackground(uv)` — dark space + FBM nebula
2. `renderStars(uv, time)` — 7-octave procedural star field
3. `lensDeflect(px, bh, r)` — gravitational deflection + shadow
4. `renderLens(uv, px, bh, r, lensStrength)` — apply deflection to UV
5. Background blend: unlensed → lensed (via `blendLens` at `infR = r*12`)
6. `renderConstellations(luv, time)` — four real star charts (lensed by BH)
7. `renderEventHorizon(px, bh, r)` — pure dark sphere, no ring
8. `renderDisk(px, bh, r, time, mouse, lensBoost)` — three-zone accretion disk
9. Text overlay + tone mapping + post processing

### Disk system (three-zone angular)

| Zone | Mask | Angular center | Purpose |
|------|------|---------------|---------|
| Lower disk | `lowerMask` | ang = -π/2 (bottom) | Directly visible front disk |
| Upper disk | `upperMask` | ang = π/2 (top) | Same radial profile as lower |
| Middle bridge | `midMask` | d.y distance | Equatorial bridge, tapered width |

Upper/lower share identical radial parameters: `exp(-diskT*0.3)*1.8`, edge window `smoothstep(diDisk, diDisk+r*.3, dd) * (1-smoothstep(duDisk-r*3.5, duDisk, dd))`.

Middle bridge: cone-shaped width `σ² 0.6~2.5`, fixed warm color, traveling hotspot animation, independent gradient.

### Constellation system

Four real constellations at screen corners, based on user-measured coordinates:

| Position | Constellation | Stars | Panel | Feature |
|----------|--------------|-------|-------|---------|
| Top-left | ♌ Leo | 9 | 个人 | Sickle arc + body + tail |
| Top-right | 🗡 Orion | 17 | 项目 | Hourglass + belt + arms |
| Bottom-left | ♫ Lyra | 5 | 文章 | Parallelogram + Vega |
| Bottom-right | 🦢 Cygnus | 9 | 联系 | Northern Cross |

Constellations use lensed UV (`luv`) so they warp near the black hole. White stars, gray lines, subtle breathing animation.

### Navigation & panels

- Navbar: 首页 / 个人 / 项目 / 文章 / 联系 (5 items)
- Four corner `.cz` click zones (22vmin) overlaid on constellations
- Semi-transparent panels (`rgba(2,2,12,.65)` + `backdrop-blur`)
- Home panel: two-column layout (avatar + bio + tags)
- Esc / ✕ / "返回主页" close panels

### BH movement

- Mouse near (<45% screen) → gravitational attraction (pull 0.4~1.6)
- Mouse far → patrol to random waypoints every 5~8s at constant speed (50px/s)
- Movement uses constant-velocity (not lerp) to avoid acceleration artifacts

## Key constants (v1.2)

| Constant | Value | Location |
|----------|-------|----------|
| BH radius | `min(w,h) * 0.035` | JS `TR` |
| Lens influence | `r * 12.0` | FS `infR` |
| Deflection strength | `r² * 14` | FS `lensDeflect` |
| Disk inner/outer | `r*2.2` / `r*6.5` | FS `diDisk/duDisk` |
| Disk falloff | `exp(-diskT * 0.3)` | FS `rad` |
| Middle bridge length | `duDisk * 1.65` | FS if-condition |
| Middle bridge width | `σ² = 0.6 ~ 2.5` (tapered) | FS `midW` |
| Attract range | `min(w,h) * 0.45` | JS |
| Star density | `thr = 0.014 + L * 0.007` | FS `renderStars` |

## GLSL pitfalls

- `smoothstep(a, a, x)` — undefined when edges equal (causes NaN → black screen)
- `normalize(vec2(0))` — NaN
- `atan(0, 0)` — undefined
- `doppler` was fixed: `sin(beamAngle*0.5)` → `cos(beamAngle)` to remove ±π discontinuity

## Coordinate spaces

- `uBH`: Three.js y-up (`RE.y - bh.y`)
- `uMouse.y`: CSS pixels (0 = top)
- GLSL UV: (0,0) = bottom-left
- CSS positioning: (0,0) = top-left

---

## Migration plan → Next.js (v2.0)

Target stack: **Next.js + Vercel + Neon PostgreSQL + Cloudflare R2 + GitHub OAuth**

### Phase 1: Framework migration
- Scaffold Next.js project
- Port GLSL shaders into React component (useEffect + useRef for Three.js)
- Port CSS to Tailwind or CSS modules
- Port panel system to React state / Next.js routes
- **Goal**: same visual, modern architecture

### Phase 2: Database + auth
- Prisma schema: users, articles, projects, messages, ai_chat
- NextAuth.js with GitHub OAuth
- Admin-only content management

### Phase 3: Dynamic content
- Article CRUD (Markdown)
- Project cards from DB
- Contact form → DB

### Phase 4: AI assistant
- Floating AI chat widget (bottom-right)
- API key stored in Vercel env vars
- Chat history in DB

### Phase 5: Storage + domain
- Cloudflare R2 for images
- Custom domain + DNS

### Core assets (framework-agnostic, must preserve)
- GLSL shader source (renderBackground, renderStars, lensDeflect, renderLens, renderEventHorizon, renderDisk, renderConstellations, toneMapping, postProcess)
- Constellation star coordinates
- Black hole visual parameters
- Panel layout design
