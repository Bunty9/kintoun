# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained `index.html` ("Golden Cloud") that renders an interactive, volumetric, ray-marched cumulus cloud at sunset using a WebGL2 fragment shader. No build step, no dependencies, no tests — all HTML, CSS, GLSL, and JS live in `index.html`.

## Running

Open `index.html` in a WebGL2-capable browser, or serve it. The VS Code Live Server extension is configured to use port 5501 (`.vscode/settings.json`). Falls back to a message div if WebGL2 is unavailable.

## Architecture

Everything is in `index.html`. The structure, top to bottom:

- **Fallback gradient + canvas** — CSS dusk gradient shows while the GPU warms up; canvas fades in (`#c.on`) once the first frame renders.
- **`FRAG` (the GLSL fragment shader string)** is the heart of the program. A single fullscreen triangle is rasterized and all the rendering happens per-pixel:
  - **Noise** — `hash33`/`hash13`, Perlin-style `gnoise`, `worley` (cellular), and `fbm` stack to build cloud detail.
  - **`densityHi` / `densityLo`** — the cloud's density field. `Hi` is full-quality (domain-warped fbm + worley billows, eroded edges) for the primary march; `Lo` is a cheap version sampled only for self-shadowing. Both gated by `envelope()` (the overall cumulus silhouette).
  - **`applyMouse`** — a soft vortex around the cursor that swirls/parts the vapour, driven by the `uStir` uniform.
  - **Lighting** — `hg` (Henyey-Greenstein phase function) gives forward-scattered golden lining; `lightMarch` marches toward the sun accumulating optical depth with a multiple-scattering approximation.
  - **`sky`** — analytic sunset gradient + sun glow, composited behind the cloud via transmittance.
  - **`main`** — sets up camera (with mouse parallax), intersects the cloud bounding sphere, runs the primary raymarch loop (`STEPS = 88`, energy-conserving integration with early-out on low transmittance), then grades the result (`aces` tonemap, gamma, saturation, vignette, film grain).
- **GL boilerplate** — `compile`/link program, one VAO + a 3-vertex buffer for the fullscreen triangle, uniform locations cached in `U`.
- **`resize`** — caps the rendered long edge at `MAX_DIM = 1280` and multiplies by `qScale` (a monotonically-decreasing quality factor).
- **Pointer interaction** — `pointermove`/`down`/`leave` update mouse position (`mx`,`my`, y-up normalized) and accumulate `stir`, which decays each frame.
- **`frame` render loop** — `requestAnimationFrame` driver. Tracks an EMA of frame time and, after a warmup, drops `qScale` one notch (×0.85) on sustained slowness (>45 slow frames) — a one-way self-throttle to avoid oscillation. Pushes all uniforms and draws.

## Conventions / gotchas

- **Uniforms are the JS↔shader contract.** Adding a new shader parameter means: declare the `uniform` in `FRAG`, add it to the `U` location map, and set it each frame in `frame()`. Keep all three in sync.
- **Two density functions must stay consistent.** `densityHi` and `densityLo` share the same domain-warp/envelope setup; if you change the cloud shape in one, mirror the relevant parts in the other or self-shadowing will diverge from the visible cloud.
- **Performance is shader-bound.** Cost scales with `STEPS`, the `lightMarch` loop count, and resolution. Tune `MAX_DIM`, `STEPS`, or the throttle thresholds rather than expecting CPU-side wins.
- **Mouse y is flipped** to be y-up (`1.0 - y/height`) before reaching the shader.
