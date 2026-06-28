# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

HeartSaga ("Saga") is a browser-based 3D mini-game built to raise awareness/funds for congenital heart disease. A small girl, Saga, starts at half a heart — slow and fragile, with a hospital-monitor heartbeat beep — and grows stronger by collecting 8 donation hearts. The emotional core is the low-health beeping giving way to a calm ambient tone as she heals.

## The entire app is one file

`index.html` (~1100 lines) is the whole game: inline CSS, an inline ES-module `<script>`, and all logic. There is **no build step, no bundler, no package.json, no tests, no dependencies installed locally**. Everything (Three.js + addons) loads from CDN at runtime. Do not introduce a toolchain unless explicitly asked — the "no build step, single file" constraint is a core design goal.

The other files: `README.md` (title only) and seven `*.glb` assets (`Saga.glb` + six Quaternius nature models). **The GLBs are currently unused** — the character and all environment props were migrated to fully procedural Three.js geometry. Leave the GLB files unless asked to remove them; the code references none of them.

## Running and validating

- **Run it:** open `index.html` in a browser **over http(s)** — the production URL is https://abby-world.github.io/HeartSaga/ (GitHub Pages, served from `main`). It will **not** work from `file://`: it uses ES-module imports and fetches assets by relative path, both of which need an origin. To test locally, serve the folder (e.g. `python3 -m http.server`) rather than double-clicking the file.
- **First click is required at runtime:** audio (Web Audio) and pointer-lock only start on the start-screen click — a browser security rule, not a bug.
- **No linter/test runner exists.** Validate JS edits by parsing the module body (imports stripped, since `new Function` can't hold `import`):
  ```bash
  node -e 'const fs=require("fs");const h=fs.readFileSync("index.html","utf8");
  const m=h.match(/<script type="module">([\s\S]*?)<\/script>/);
  let c=m[1].replace(/^\s*import[\s\S]*?;\s*$/gm,"");
  new Function("THREE","EffectComposer","RenderPass","OutlinePass","ShaderPass","FXAAShader","window","document","performance","requestAnimationFrame","console",c);
  console.log("SYNTAX OK");'
  ```
  Real verification is visual, in a browser — state that you could not do visual verification when you cannot run one.

## Network constraint when working here

This environment's egress **blocks CDN hosts (cdnjs, jsDelivr)** — `curl`/WebFetch to them returns 403. You cannot fetch the Three.js files to verify URLs from here; the user's browser and GitHub Pages reach them fine. Verify CDN paths against known Three.js release facts, not by fetching. Notably: Three.js removed `examples/js/` (UMD example builds) in **r148**, so at the pinned **r152** only `examples/jsm/` ES-module addons exist — they are imported by full jsDelivr URL, not via the importmap.

## Module / CDN setup (r152)

- An `<script type="importmap">` maps the bare specifier `"three"` to `three.module.min.js` on jsDelivr. The main `<script type="module">` does `import * as THREE from "three"` plus full-URL imports of `GLTFLoader`-style addons (currently `EffectComposer`, `RenderPass`, `OutlinePass`, `ShaderPass`, `FXAAShader`). Addon modules resolve their own `three` import through the importmap.
- To add another Three.js addon: add an `import { X } from "https://cdn.jsdelivr.net/npm/three@0.152.0/examples/jsm/.../X.js"` line — **not** an importmap entry.
- r152 has `ColorManagement` on by default; raw `ShaderMaterial` output (the sky dome) and the post-processing chain can shift brightness/saturation vs. direct rendering — a known, fixable concern if colors look off.

## Code architecture (inside the one module IIFE)

All code lives in one `(function(){ "use strict"; ... })()`. It is organized top-to-bottom into commented sections that build the scene once, then run a single `requestAnimationFrame` loop:

- **Constants / State** — `MAX_HEALTH` 5, `START_HEALTH` 0.5, `HEART_VALUE` 0.5, `TOTAL_HEARTS` 8, `WORLD_SIZE`. Mutable `health`, `collected`, `started`, `won`, `keys`.
- **Renderer / Scene / Camera** — gradient sky is a large inside-out sphere with a `ShaderMaterial`; `scene.fog` is `FogExp2`. `camera.far` is deliberately 1000 (not 400) so the radius-400 sky dome isn't clipped — keep it ≥ the sky radius if you touch either.
- **Post-processing** — `EffectComposer` → `RenderPass` → `OutlinePass` → FXAA `ShaderPass`. Cel outlines are driven by `outlinePass.selectedObjects`; objects must be **pushed** to it when created (trees push synchronously; Saga pushes after she's built). `composer.render()` is called in the loop instead of `renderer.render()`; resize must update composer + outline pass + FXAA `resolution` uniform.
- **Ground / Environment** — procedural Lambert ground + scattered translucent circle patches, and procedural tree/rock/bush builders. Trees register their canopy `Group` in a `windCanopies` array for sway. The `obstacles` array holds `{x,z,radius}` and is the single source of truth for collision; it is filled **synchronously** at placement (radii: trees 1.4, rocks 0.9; bushes are decorative, no collision).
- **Saga** — a procedural cel-shaded girl `Group` named `sagaMesh`, child of the `saga` position/rotation container. Limbs (`upperArmL/R`, `legL/R`) are sub-groups pivoted at shoulder/hip so `rotation.x` swings them; the model is built facing **+Z** (the movement-forward direction, so no facing offset is needed). `saga` is the thing that moves; `sagaMesh` is the visual; `sagaGlow` (PointLight) is a child of `saga`.
- **Donation hearts** — array of floating heart `Group`s; auto-collected on proximity to Saga.
- **Camera control / Audio / Heart UI** — drag/pointer-lock orbit; all sound synthesized via Web Audio (heartbeat scheduler, ambient pad, collect chime — no audio files); hearts HUD drawn on `<canvas>`.
- **Game loop (`animate`)** — reads `health/MAX_HEALTH` into `healthFrac`, which scales movement speed, walk-cycle pace, and the audio state. WASD is camera-relative; movement integrates velocity, then `resolveObstacles()` pushes Saga out of obstacle radii and clamps to the world. Procedural walk animation advances `walkCycle` and swings limbs only while moving.

### Cross-cutting invariant: health drives everything

`healthFrac = health / MAX_HEALTH` is the central dial. Raising `health` (via `collectHeart`) must keep these in sync: hearts HUD, movement speed/acceleration, walk-animation pace, and the audio mode (constant beep below ~4 hearts → slows & softens as health rises → ambient pad above 4). When all 8 are collected the win overlay shows and movement halts. Touching one of these systems usually means checking the others.

## Workflow / deployment

- **Develop on branch `claude/zelda-charity-game-3d-u7m0fc`.** Never push to a different branch without explicit permission.
- **Deploying = merging to `main`.** GitHub Pages serves `main` at the production URL, so a change is only "live" once it reaches `main`. The established pattern is: commit on the feature branch and push it, then `git checkout main`, `git reset --hard origin/main` (local `main` goes stale between sessions — always resync first), `git merge --no-ff <feature-branch>`, push `main`, and `git checkout` back to the feature branch. Pages rebuilds in ~1–2 minutes; a hard refresh is needed to bust cache.
