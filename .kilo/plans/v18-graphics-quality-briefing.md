# V18 GRAPHICS — BRIEFING v2 (aligned to the builder's own /tmp/v17 workspace)

**Baseline:** `/workspace/app/cosmodrome-v17.html` (3,112,675 B, `cosmodrome-v17-20261017-ultra-tier`)

**Builder's working set (use these paths for patches):**
- `/tmp/v17/script.js` — full extracted script
- `/tmp/v17/base_min.js` — minified base only (1,166,230 B, up to `// V7 "FIELD DRESSING"`)
- `/tmp/v17/base_pretty.js` — 2,325 lines, pretty-printed
- `/tmp/v17/sh/*.glsl` — extracted shaders: `VS FS FS_R2 SHADOW_VS SHADOW_FS POST_VS POST_FS AO_FS DUST_VS DUST_FS`

**Agreement status:** the builder's own greps independently confirm all nine defects below —
`renderbufferStorageMultisample: 0 | blitFramebuffer: 0 | texStorage: 0 | compressed: 0`,
`antialias:!0`, `clamp(uTaps,1,48)`, `Math.min(4, MAX_ANISOTROPY)`, `v8Lod` with no ultra→LOD-3 gap,
`"ultra"===settings.quality?3840`, `renderer.scale=Math.max(.64,…)`, `createProceduralPBR(512,84417)`,
`for(let t of[4,16,64,128])`, `perspective(2.3,1,.12,52)`, `perspective(1.24,1,.12,48)`,
`perspective(mix(b,66,o.aim)*…,this.w/this.h,.035,1600)`, `R=[E[0],…E.slice(1)….slice(0,11)]`,
`uLPos[12]`, 9-tap PCF, tri-planar 3+3, `clamp(…,0.,.72)` fog, 4-tap bloom, `1600.` hardcoded 3×.
**No contradictions. Analysis stands.**

---

## PART 0 — NEW FINDINGS THE BUILDER'S SHADER EXTRACT REVEALED

These were not visible until the shaders were pulled out to `.glsl`.

### N1. Only the material array gets mipmaps — decals, supply panel and in-world screens do not
`generateMipmap: 1` (one call, on the material `TEXTURE_2D_ARRAY`). But `texture2D(e)` — used for the
**decal atlas (1024²)**, the **supply panel (512×256)** and the **V8 screen atlas** — sets only
`MIN_FILTER=LINEAR`, `MAG_FILTER=LINEAR`, `WRAP=CLAMP_TO_EDGE`. **No mipmaps.**
→ Decals and every in-world screen alias and shimmer at distance. The V8 screen atlas is exactly the
thing the Ultra tier enlarged to 1024²/256px tiles — and it is still unfiltered.

**Patch** — in `texture2D()`, add
`e.generateMipmap(e.TEXTURE_2D)` and switch `MIN_FILTER` to `e.LINEAR_MIPMAP_LINEAR`, with
`WRAP_S/T = CLAMP_TO_EDGE` (already set) so mipmapping is safe.
⚠️ The V8 screen atlas is updated per-frame via `texSubImage2D` — regenerating its mipmap every frame
is expensive. Either (a) generate mipmaps only on the full-atlas draw (`v8ScreenInit` / the
`only === undefined` path) and accept stale mips on updated tiles, or (b) leave the screen atlas
un-mipped and mip only the decal + supply textures. **Decals are the big win** — they cover floors
and walls at distance.

### N2. `FS_R2 = FS` — there is only ONE shading model, and it is the extension point
The builder's acorn dump: `FS_R2 = FS`. The constructor:
```js
this.program=(()=>{try{return this.programOf(VS,FS_R2)}catch(e){
  return console.warn("R2 shading unavailable; legacy lighting active:",e&&e.message),this.programOf(VS,FS)}})()
```
Because `FS_R2` is an **alias**, the fallback is byte-identical to the primary path — the
"R2 shading unavailable; legacy lighting active" warning can never do anything useful.

**Consequence for the plan:** `FS_R2` is the *designed* place for a second shading variant and it is
already wired ahead of FS. If the builder wants an "ultra shading" path (more lights, better shadow
taps, IBL, different ambient), **make `FS_R2` a genuinely distinct shader** rather than an alias.
That is a clean, already-supported extension point — and it lets low/balanced/high keep the exact
current FS (satisfying the "no regression" guard) while Ultra gets a different program.
Recommend: `FS_R2` = Ultra variant, `FS` = legacy for the other three tiers, selected by
`settings.quality` at program-creation time. Watch out: the program is created **once** in the
constructor, so a runtime tier change would need a program rebuild (see Part 1, B5).

### N3. Volumetric ray length hard-capped at 34 units, density fixed at 0.013
`sh/POST_FS.glsl`: `float distance=min(34.,z/max(dot(ray,uCameraForward),.08))` and `density=.013`.
Light shafts can never reach more than 34 m from the camera — a hard ceiling on volumetric presence
regardless of tier. Cheap upgrade: make the cap and density tier-scaled (e.g. 34 → 60 on Ultra) and
expose the cap as a uniform.

### N4. AO floor hard-clamped to 0.34 — contact shadows can never go truly dark
`sh/AO_FS.glsl`: `float ao=clamp(1.-occlusion/max(float(taps),1.)*2.75*uAmount,.34,1.)`
Maximum occlusion is 66%. On Ultra, lowering the floor to ~.22 deepens contact shadowing — a
one-character-ish change with a visible payoff.

### N5. Dust motes are clamped to 2.8 device pixels — invisible on high-DPR displays
`sh/DUST_VS.glsl`: `gl_PointSize=clamp(11./max(distance,1.),.75,2.8)`.
2.8 *device* pixels is far below one CSS pixel on a Retina/4K display, so the dust reads as sparkle
noise rather than volume, and mostly disappears. Fix: multiply by the device pixel ratio —
`gl_PointSize=clamp(11./max(distance,1.),.75,2.8)*max(1.,uDPR)` (or bake the DPR into the clamp).
Also note `DUST_FS` outputs **non-premultiplied** `vec4(tint,alpha*exp(-r*r*18.))` and the dust pass
uses `SRC_ALPHA, ONE_MINUS_SRC_ALPHA` — that one is **correct**; do not change it.

### N6. `compressedTex: 0` — no compressed texture support anywhere
The material set is 512²×12 layers RGBA8 **plus** a 512²×12-layer relief array ≈ **25 MB VRAM
uncompressed**. Raising the atlas to 1024² (Part 3) would push that to ~100 MB. For a single-file
offline build that is a real tradeoff: either add ETC2/ASTC upload paths (needs the compressed
payloads embedded as base64 — big file growth), or keep the atlas at 512² and buy quality elsewhere
(normal maps, IBL, shadow cascades) which cost no VRAM.

### N7. `texStorage: 0` — `texImage2D/3D` used instead of immutable `texStorage`
Minor, but `texStorage` is faster and lets the driver optimize the allocation. Optional cleanup.

### N8. Confirmed NOT broken — do not "fix"
- `DUST_FS` premultiplication: **not** an issue (non-premultiplied + correct blend func). Only
  `V8_PART_FS` has the premultiplied/blend mismatch (defect B9).
- The `uWeapon==1` path in FS.glsl uses a **fixed 3-light rig** (`cookTorrance` with two hardcoded
  directions) instead of the 12-light loop — that is the viewmodel's own lighting, intentional.

---

## PART 0B — RUNTIME-VERIFIED FINDINGS (builder's Playwright/SwiftShader measurements)

These are now **measured**, not inferred. They include the single largest visual defect in the game.

### R1. **Light leaks through walls — only 1 of 12 sampled lights is shadowed** ← BIGGEST DEFECT
`sh/FS.glsl`, main light loop:
```glsl
sh = i==0 ? mix(.10,1.,shadowPCF(uKeyShadow,uKeyVP,vP,Ng,L)) : 1.
col += cookTorrance(base,metal,rough,N,V,L,uLColor[i]*radiance*sh);
```
**Only light index 0 gets a shadow.** The other 11 contribute with `sh = 1.0` — they shine straight
through concrete walls. The render loop picks the 12 nearest/strongest, so most rooms are lit by lamps
in *other* rooms.

Builder's leak test (`leak.py`, 578 sampled nav nodes, `canSee()` raycast against `WORLD.colliders`):
```
lights: 25   radii: [17,17,17,17,17,17,11,11,11,32,32,7,17,17,17,7,7,20,20,20,12,12,34,30,7]
powers:  [42,42,42,42,42,42,26,26,40,130,130,13,52,52,52,13,13,55,55,55,55,55,180,170,13]
samplePoints: 578   pointsWithLight: 577
pointsWithMoreThan25pctThroughWall: 260        <-- 45% of lit points
medianLeakFraction: 0.130
p90LeakFraction: 1.0                          <-- at 10% of points, 100% of light comes through walls
lightContribsTotal: 2106   lightContribsBlocked: 1034   <-- 49% occluded and STILL APPLIED
```
**49% of all light contributions at sampled points come from lamps behind walls, and the shader
applies every one at full strength.** This is why the facility reads flat and evenly lit despite
having 25 individual lamps with distinct radii and powers.

**Fix — pick one, ascending cost:**
- **(a) Cheapest, biggest win — per-light 2D occlusion grid.** The facility is essentially horizontal
  (26 zones; `WORLD.floors` has x0/x1/z0/z1 + y + optional `axis` ramps). Bake at load, for each of
  the 25 lights, a low-res XZ visibility grid (extent x[-78,104] z[-19,97] → ~2 m cells = 91×58 =
  5,278 cells/light; 25 lights = 132 KB). Populate by DDA-raycasting each cell against
  `WORLD.colliders` — `rayBox`/`rayWorld` already provide the raycast. Sample it in the shader via
  one `sampler2DArray` (25 layers) and multiply `radiance` by the stored visibility.
  **No extra render passes, no extra shadow maps, one texture fetch.** This alone should transform
  the lighting.
- **(b) Per-light shadow maps in one atlas** — 12 lights × 512² in a 4×3 atlas of 2048×1536 = 12 MB
  and **one** extra scene pass per frame. True soft shadows per lamp, but costly.
- **(c) Screen-space shadow** (ray-march the depth buffer toward the light) — no extra geometry,
  but only works for lights in view.

**Do (a) or (c) first.** (a) is the best value in this whole document.

### R2. **The 512² PBR atlas costs 2.3 SECONDS at load — 1024² costs only 3.0s**
Measured: `createProceduralPBR(512,84417)` = **2275 ms**, `createProceduralPBR(1024,84417)`
= **3040 ms**. Cost is dominated by the 4-octave value-noise sampling loop, **not** the texel count.
→ **Raising the material atlas to 1024² is nearly free** (+765 ms on an already-2.3 s load). Do it.
→ Separately, **2.3 s of frozen startup is itself a quality problem.** Options: Web Worker,
generate the noise fields once at 256² and upsample (the finest octave is 4 texels wide anyway), or
generate progressively (128² first, refine on Play). `materialsReady` is already a Promise — use it
for a progress bar instead of a 2.3 s freeze.

### R3. **Device headroom the code is not using** (measured on the build's own GL context)
```
maxTex: 8192    maxSamples: 4    maxArrayLayers: 2048    aniso: 16
colorFloat: true    floatLinear: true    s3tc: true    astc: true
```
- `maxSamples: 4` → 4× MSAA is available (B4 is a code problem, not a device limit).
- `aniso: 16` but the code uses `Math.min(4, MAX_ANISOTROPY_EXT)` → **12 of 16 taps unused.**
  Raise to 8 or 16 — free quality on grazing floor/wall angles.
- `s3tc: true`, `astc: true` → compressed textures available but the code uses **zero**
  (`compressedTex: 0`). 512²×12-layer RGBA8 materials + 512²×12 relief ≈ 25 MB uncompressed; at
  1024² that is ~100 MB. **Add an S3TC upload path** and the VRAM problem disappears — but the
  payload must be embedded as base64, growing the single file. Pragmatic split: **S3TC for the
  material array, keep relief uncompressed** (relief is read for bump + metal/cavity, and DXT5's
  alpha channel can carry height).
- `maxTex: 8192` → the 1024² screen atlas and 1024² decal atlas are well within budget.

### R4. **Measured per-frame budget at Ultra, 960×540** (SwiftShader *software* rasteriser)
```
calls: 307–411      tri: 219,885–456,564      pts: 1792
ms:    3.4–11.0      (first frame 155–162 ms = shader compile)
archVerts: 25,743  (centrifuge 3936 · skyline 5610 · courtyard 1626 · bunker 2670 ·
                     control 2202 · gantry 7911 · launch 1284 · links 504)
archDraws: 1–3     archDrawnVerts: 18,897–23,073
worldVerts: 86,448  groups: 103  nodes: 1,156  zones: 26
```
**3.4–11 ms per frame in a pure-software rasteriser at 960×540.** On a real GPU this is a fraction of
a 16.7 ms budget — **enormous headroom** for more lights, more shadow work, higher resolution and a
real bloom chain. The current tier ladder is far more conservative than the engine needs.
Note `archDraws` is only 1–3 because `v8DrawArch` frustum-culls the 8 zones and merges contiguous
runs — that culling works well.

### R5. **The PolyArt zombie texture is only 256×256 RGB**
Decoded from the embedded base64: `zombie texture size: (256, 256) RGB`. At close range on a high-res
screen the zombies are the softest thing on screen while the world is procedurally sharp. This is an
**asset** limitation, not a code one — it cannot be fixed without a bigger source texture. Note it in
the changelog; consider a sharpening or detail-normal pass on the zombie material as a code-side
mitigation.

### R6. Confirmed working — do not touch
- `v8Stats.lod` correctly reports 3 on Ultra, `archVerts: 25,743`, `particles: 1024` — the Ultra
  architecture and particle pool **do** build when the render loop runs. (In the builder's first
  headless probe `archVerts` was 0 and `lod` 1 for all tiers — an artifact of stubbing
  `requestAnimationFrame`, so `v8BeforeRender` never ran. **Not a bug.**)
- The zone culling in `v8DrawArch` (8 zones → 1–3 merged draw runs) works.
- `polyZombieParts` / VAT: 6552×24×10 RGBA16F position + normal arrays, NEAREST-filtered, 24 frames
  × 10 variants, PCA-compressed. Works.
- `uV9Sharp` is written correctly (only when `uSpeed` is set on the post program — that is the
  speed-smear path, and the sharpen is applied alongside it in `POST_FS`).
- PNG export via `toDataURL` is safe because `preserveDrawingBuffer:false` and V9 exports inside the
  same task that submitted the frame.
- `rayWorld` also uses 1600 as its max distance — consistent with the projection far plane. If the far
  plane changes, `rayWorld` must change too.

---

## PART 1 — THE NINE DEFECTS (unchanged, now with builder-aligned patch sites)

### B1 — Ultra's 64 AO taps clamp to 48
**File:** `sh/AO_FS.glsl`
```glsl
int taps=clamp(uTaps,1,48)      →  int taps=clamp(uTaps,1,96)
for(int i=0;i<48;i++)            →  for(int i=0;i<96;i++)
```
`render()` uploads `v8Lod()===3?64:…` — the shader discards it. **Ultra AO ≡ High AO today.**
⚠️ `base_min.js` holds the same string inside the `AO_FS` template literal — patch the `.glsl` and
re-inject, or patch the template literal directly. Count-assert `clamp(uTaps,1,48)` occurs exactly once.

### B2 — Ultra's volumetric ray strength `.69` is dead
`render()` writes `uRayStrength` from `D=v8Lod()===3?.69:…` → `L=Math.max(O?D:0,_)` →
`return this.f(this.post,"uRayStrength",L)`.
**Then** `render()` calls `this.configureBodycam(...)`, which writes the same uniform again:
`this.f(l,"uRayStrength",d)` where `d="high"===settings.volumetric?.95:"medium"===settings.volumetric?.64:0`.
**The second write wins.** Every tier gets `.95`/`.64`/`0`; Ultra's `.69` never reaches the GPU.
V17's own header claims "volumetric 24 steps / .69 ray strength" — that claim is false.
**Note:** `settings.volumetric` Off/Medium/High *is* respected. The dead value is Ultra's own boost.

**Patch** — make `configureBodycam`'s `d` tier-aware and give Ultra a value **above** High's `.95`:
```js
d = v8Lod()===3 ? ("high"===settings.volumetric?1.15:"medium"===settings.volumetric?.8:0)
                : ("high"===settings.volumetric?.95:"medium"===settings.volumetric?.64:0)
```
(Optionally delete the now-provably-dead `D`/`L` computation in `render()`.)
`uVolumeSteps` is unaffected — written only in `configureBodycam`
(`v8Lod()===3?24:"low"===settings.quality?8:"high"===settings.quality?20:14`) and 24 is exactly the
shader's ceiling `clamp(uVolumeSteps,1,24)`.

### B3 — V9 `v9res` mode 1 is a dead branch; mode 2 caps below Ultra
`v9Resize` has only `if (mode === 2) { …Math.min(r.width*Math.min(dprRaw,2),2560)… }`.
Mode 1 ("True 1080p · 1920 cap") does nothing — you get the quality cap (3840 on Ultra).
V15 worked because V15 temporarily set `settings.quality="ultra"` before calling resize; V17 removed
that hack and the mode died with it. Mode 2's 2560 cap is **below** Ultra's 3840.
**Patch:** add a `mode === 1` branch (1920 cap) and raise mode 2 to 3840. Preserve the
`finally { settings.quality = q; this.scale = sc; }` restore.

### B4 — MSAA is a no-op
`getContext("webgl2",{alpha:!1,antialias:!0,depth:!0,powerPreference:"high-performance",preserveDrawingBuffer:!1})`
`antialias` applies to the **default framebuffer**; the scene renders single-sampled into
`this.scene.fbo` (`DEPTH_COMPONENT24` single-sample texture, `checkFramebufferStatus` guarded).
`renderbufferStorageMultisample: 0`, `blitFramebuffer: 0` — the builder confirmed both are absent.
The post pass then draws a fullscreen triangle to the default framebuffer.
**MSAA smooths a fullscreen triangle's edges — there are none.** FXAA in `POST_FS` is the only edge AA.
**Patch:** see Part 2, or revert `antialias:!0` → `antialias:!1` (free win, zero visual loss).

### B5 — Shadow maps never reallocate on a runtime quality change
`this.shadow=this.shadowBuffer(v8Lod()===3?2048:"low"===settings.quality?256:"balanced"===settings.quality?512:1024)`
`this.keyShadow=this.shadowBuffer(v8Lod()===3?2048:"low"===settings.quality?512:"balanced"===settings.quality?768:1536)`
`resize()` deletes and recreates the scene/ao/hero FBOs + textures but **not** the shadow buffers.
**Patch:** add `Renderer.prototype.setQualityTier()` that disposes + recreates both (mirror the
`destroyMesh` / `deleteFramebuffer`+`deleteTexture` pattern already in `resize()`), and call it from
the quality `oninput` handler. Also rebuild the particle pool here (B7) and the shader program if
the builder adopts the FS_R2 split (N2).

### B6 — Screen atlas desyncs on runtime quality change
`v8ScreenInit()` bakes `const tp = v8Lod() === 3 ? 256 : 128;` **once** (guarded
`if (v8ScreenTex) return;`), but `v8DrawScreens()` and the `texSubImage2D` call read `v8Lod()` live.
Change quality at runtime → wrong tile offsets, 2×-scaled content clipped into a 128px tile canvas,
uploads landing in the wrong quadrant. All in-world screens corrupt.
**Patch — make the atlas tier-independent (recommended; one 1024² texture is negligible memory):**
- `const tp = v8Lod() === 3 ? 256 : 128;` → `const tp = 256;`
- `const tp = v8Lod() === 3 ? 256 : 128, x =` → `const tp = 256, x =`
- both `(v8Lod() === 3 ? 256 : 128)` in `texSubImage2D` → `256`

### B7 — Particle pool never resizes (third boot-frozen resource)
`v8PartInit()`: `const gl = rend.gl, n = v8Lod() === 3 ? 1024 : v8Lod() === 0 ? 96 : v8Lod() === 2 ? 512 : 256`
then `v8PartData = new Float32Array(n*8)` and a one-shot `bufferData` sized to `n*8`.
**Patch:** rebuild the pool (VAO + buffer + program) at the new `n` inside the same `setQualityTier()`
as B5.
(Also update V9's stale comment — it still says *"pool caps stay V8's (96/256/512)"*; Ultra is 1024.)

### B8 — "Default is Ultra" is false for every returning player
`installV17Hooks()` (runs before `boot()`) does `delete settings.quality; settings.quality = "ultra";`
Then `boot()` merges localStorage over it:
```js
const e=JSON.parse(localStorage.getItem("last-signal-cosmodrome-v5-settings")||"null")
if(e)for(const t of Object.keys(settings))t in e&&(settings[t]=e[t])
```
`wireControls()` writes the **whole** blob on any slider change, so `quality` is in every existing save
→ the merge overwrites Ultra. **Ultra reaches fresh installs only.**
**Patch:** one-time migration inside `installV17Hooks()` (runs before `boot()`): read the
`"last-signal-cosmodrome-v5-settings"` blob; if it has no `v17QualityChosen` flag, set
`blob.quality="ultra"` and write it back. Set the flag when the user picks a tier.

### B9 — Particle additive blending double-multiplies alpha
`V8_PART_FS` outputs **premultiplied** `vec4(tint*alpha*s, alpha*s)` but the draw uses
`gl.blendFunc(gl.SRC_ALPHA, gl.ONE)` → the source is multiplied by alpha a second time → particles
render at ~alpha² brightness (steam, sparks, embers, thruster plumes all dimmer than authored).
**Patch:** `gl.blendFunc(gl.SRC_ALPHA, gl.ONE)` → `gl.blendFunc(gl.ONE, gl.ONE)` — **only the V8
particle pass.** The builder's grep shows three `blendFunc` calls; the dust pass and the glass pass
use `SRC_ALPHA, ONE_MINUS_SRC_ALPHA` and are **correct** (see N5) — leave them.

---

## PART 2 — REAL MSAA

`resize()` already owns the scene FBO creation and teardown, so the multisample path plugs in there.

1. In `resize()`, beside the existing `scene` FBO, create `sceneMS`:
   - colour: `gl.renderbufferStorageMultisample(gl.RENDERBUFFER, samples, gl.RGBA16F, w, h)`
     (fall back to `gl.RGBA8` when `EXT_color_buffer_float` is absent — mirror the existing `c` check)
   - depth: `gl.renderbufferStorageMultisample(gl.RENDERBUFFER, samples, gl.DEPTH_COMPONENT24, w, h)`
   - `samples = Math.min(4, gl.getParameter(gl.MAX_SAMPLES))`
2. `render()`: bind `sceneMS.fbo` instead of `scene.fbo` for the scene pass.
3. Resolve before the AO pass:
   ```js
   gl.bindFramebuffer(gl.READ_FRAMEBUFFER, sceneMS.fbo)
   gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, scene.fbo)
   gl.blitFramebuffer(0,0,w,h, 0,0,w,h, gl.COLOR_BUFFER_BIT|gl.DEPTH_BUFFER_BIT, gl.NEAREST)
   ```
   Depth blit requires identical formats on both sides — keep `DEPTH_COMPONENT24` on both.
4. The AO pass reads `scene.color` + `scene.depth`; the post pass reads `aoColor` — **both unchanged**,
   they now read the resolved textures.
5. Soften or gate `POST_FS`'s `fxaa()` when MSAA is on (double-AA mushes edges).
6. Fallback: if this proves unreliable, revert `antialias:!0` → `antialias:!1` and defer. Strictly
   better than today's no-op.

---

## PART 3 — RAISE THE TIER-INVARIANT CAPS

The builder has already confirmed each of these exists. None is reachable by any quality setting.

| Cap | Current (verified) | Target | File / exact site |
|---|---|---|---|
| Dynamic lights | **12 of 25, and 11 of the 12 are UNSHADOWED (R1)** | 16–24 **plus per-light occlusion (R1a)** | `sh/FS.glsl` `uLPos[12],uLColor[12]` / `uLPower[12],uLRadius[12]`; the `for(int i=0;i<12;i++)` loops (**two**: main path and zombie path); `base_min.js` `R=[E[0],…E.slice(1)….slice(0,11)]` + the `for(;R.length<12;)R.push(…)` padding. **All four must move together.** |
| Shadow filter | 9-tap PCF, `sum/9.` | 16-tap Vogel disc; PCSS on key light | `sh/FS.glsl` `shadowPCF()` (`sampler2DShadow`, hardware compare) |
| Key shadow frustum | `perspective(2.3,1,.12,52)` — a 132° **point-light** cone on the brightest lamp, not a sun | 3–4 cascades around the camera | `base_min.js` `render()`; also `lookAt(M.p,c,…)` and the `keySource`/`keyFocus` refit logic. **Biggest shadow-crispness win.** |
| Flashlight shadow | re-rendered **every frame** at 2048² on Ultra; **no architecture in it** | add architecture to the `'fill'` map | `base_min.js` `(k>0||this.shadowDirty)&&this.shadowPass(this.shadow,P,e)`; `v8DrawArch` only draws into the `'key'` map (`this.v8Pass==='key'&&v8Lod()>=1`) |
| Material atlas | 512², 12 layers, **4× aniso of 16 available**, **2.3 s load cost** | **1024² — only +765 ms (R2)**, aniso 8–16, S3TC/ASTC (R3) | `base_min.js` `createMaterials()`: `createProceduralPBR(512,84417)`, `texImage3D(TEXTURE_2D_ARRAY,0,RGBA8,…)`, `Math.min(4,MAX_ANISOTROPY_EXT)` |
| Texture noise | **4-octave value noise** (4/16/64/128); finest octave = 4-texel features at 512² | gradient/simplex, ≥6 octaves | `base_min.js` `createProceduralPBR()`: `for(let t of[4,16,64,128])` |
| Normal maps | **none** — `uRelief` R=height, converted by `derivativeRelief` | real tangent-space normal array | `sh/FS.glsl`; add a third texture array alongside `uMaterials`/`uRelief` |
| Bump fade | **off beyond 38 m** | remove/extend | `sh/FS.glsl` `float micro=1.-smoothstep(16.,38.,length(uEye-vP));` |
| Ambient / IBL | **none** — `hemi=mix(vec3(.19,.235,.28),uAmbientColor,…)`, `ambient=.15` | 32² irradiance + 128² prefiltered cube | `sh/FS.glsl` |
| Gloss reflection | **fake 2-colour analytic** | cubemap / SSR | `sh/FS.glsl` `v8env=mix(vec3(.012,.018,.026),vec3(.09,.12,.16),smoothstep(-.15,.55,v8R.y))` |
| Sky | **2-colour vertical gradient** | cubemap / stars / clouds | `sh/FS.glsl` `mix(vec3(.035,.050,.068),vec3(.008,.018,.033),horizon)`, `horizon=clamp((vP.y+9.)/180.,0.,1.)` |
| "Bloom" | **4 taps, radius 6px, threshold .85, weight .025** | real chain: bright-pass → 4–5 downsample/blur → upsample — **headroom exists (R4: 3.4–11 ms/frame in software)** | `sh/POST_FS.glsl` — V12–V15 added a lot of emissive content (RGB strobe lamps, screen panels, cyan/amber strips, rocket fire) that a real chain would transform |
| AO depth | floor clamped to **.34** | .22 on Ultra | `sh/AO_FS.glsl` (N4) |
| Volumetric reach | ray length capped at **34 units**, density fixed **.013** | tier-scaled cap + density | `sh/POST_FS.glsl` (N3) |
| Fog | clamped **.72** | allow denser + height fog | `sh/FS.glsl` `clamp(1.-exp(-length(uEye-vP)*uFog*.65),0.,.72)` |
| Far plane | 1600, hardcoded **3×** in post | make it a uniform | `sh/POST_FS.glsl` (×2) + `sh/AO_FS.glsl` (×1) — `z=.035*1600./max(.00001,1600.-d*(1600.-.035))` |
| Dynamic res | floor **.64**, **only ever decreases** | tiered floor (.64/.75/.85/.9) + restore upward | `base_min.js` `frame()`: `e>8e3&&renderer.fps<30&&renderer.scale>.64&&(renderer.scale=Math.max(.64,renderer.scale-.12),renderer.fps=50)` |
| Base FOV | **88°** | 78° + user slider | `base_min.js` `render()` `b=88+(2===y?19:1===y?5:0)*settings.imageStrength+8*g+…` |
| HDR | RGBA16F only if `EXT_color_buffer_float`; **zombie path clamps to LDR** | keep zombies HDR | `sh/FS.glsl` zombie path `outColor=vec4(clamp(zc0,0.,1.),1.)` vs main path `outColor=vec4(max(col+emit,vec3(0.)),alpha)` |
| Decal/supply/screen mipmaps | **none** (N1) | generate mipmaps | `base_min.js` `texture2D()` |
| Dust point size | clamped **2.8 px** (N5) | DPR-aware | `sh/DUST_VS.glsl` `gl_PointSize=clamp(11./max(distance,1.),.75,2.8)` |
| Un-scaled geometry | pipes 8 segs, trays 2–5 brackets, cylinders 6–8, hazard rings 24/32 | scale on `lod > 1` | V8 architecture module |
| Gore | maxSplats **72**, maxDebris 26, maxSpray 190 | 200 / 40 / 400 | `GORE_TUNING` |
| Dust count | **56/floor, not tier-gated** | 32/56/96/160 | `base_min.js` `uploadWorld()` `for(let t=0;t<56;t++)` |

---

## PART 4 — NEW CAPABILITY

- **TAA** — Halton-jittered projection, history buffer, depth/prevVP reprojection, history blend in the
  existing fullscreen post pass. Highest AA quality, biggest effort. The renderer already has `uTime`
  and a fullscreen post pass.
- **Use `FS_R2` as the Ultra shading variant** (N2) — it is already `programOf(VS,FS_R2)`-first and
  already catches compile failure. Give it a real body: 24 lights, 16-tap PCF, IBL, normal maps —
  and leave `FS` byte-identical for the other three tiers. **This is the cleanest way to satisfy the
  no-regression guard while giving Ultra genuinely different shading.**
- **User-facing rows:** Resolution (0.75×/1×/1.5×/2× DPR, independent of quality — right now 3840 is
  only reachable through Ultra's internal cap and 1440p users cannot request 1.5× supersampling),
  Shadow quality, Max lights, FOV, and an Ultra preset.
- **SSR** on floors; **cascaded volumetric**; **uniform-ize the far plane**.

---

## PART 5 — ORDER OF WORK

**A0. Light occlusion (R1a) — do this FIRST, ahead of everything else.** 49% of light contributions
come from lamps behind walls and the shader applies them at full strength. It is the single largest
visual defect in the game, and the fix is a 132 KB baked texture plus one texture fetch — no extra
render passes, no extra shadow maps, no per-frame cost. It will change the look of every room in the
facility. Everything below is incremental polish; this is the structural fix.

**A. Correctness.** B1, B2, B3, B4-revert, B5, B6, B7, B8, B9 + N1 (decal mipmaps),
N4 (AO floor), N5 (DPR-aware dust), N3 (volumetric reach). All are small count-asserted replacements
against `sh/*.glsl` and `base_min.js`. **Do these before any feature work** — otherwise you are
debugging visual regressions against a broken baseline.

**B. Real MSAA** (or revert `antialias` to `!1` and defer).

**C. Raise the caps**, highest impact first:
C1 material atlas → 1024² (+765 ms only) + aniso 8–16 · C2 per-light shadow atlas or screen-space
shadows (R1b/R1c) · C3 cascaded key shadows (replace the 132° point-light cone) ·
C4 16-tap/PCSS · C5 lights 12→24 · C6 `FS_R2` as the Ultra shading variant ·
C7 normal maps + better noise · C8 IBL · C9 real sky · C10 multi-pass bloom · C11 the rest of the table.
**There is headroom for all of it** — R4 measured 3.4–11 ms/frame in a *software* rasteriser at
960×540, against a 16.7 ms budget on real hardware.

**D. TAA + the new user-facing rows.**

**E. Startup:** R2's 2.3 s material-generation freeze → Web Worker or progressive generation with
the existing `materialsReady` promise driving a progress bar.

---

## PART 6 — REGRESSION GUARDS (non-negotiable)

1. **low / balanced / high must be pixel-identical to V17.** Only Ultra gains. Count-assert that every
   non-ultra branch of every edited expression is byte-identical.
2. **Strobe (V15) out of scope** — do not touch `V15_SOFT_HZ`, `V15_SOFT_FLOOR`, `v15PaintSoft`,
   `v15PaintAtmSoft`, `v15OffPaint`, the `strobeLevel` row.
3. **V12/V13/V14 out of scope** — do not touch `V13_STROBE_*`, `V13_SPEED`, `V14_MUSIC_GAIN`,
   `V14_SKY_A_BASE`/`SKY_B_BASE`/`FOG_BASE`/`HEMI_BASE`, or `V14_FS_MARK`/`V14_SKY_OLD`/`V14_SKY_NEW`
   — `v14PatchFS` count-asserts them (`c(x) !== 1`, `c(x) !== 2`) and will fail.
   ⚠️ **N2 warning:** if `FS_R2` is edited, `v14PatchFS` operates on the FS source at `shaderSource`
   time. `v14PatchFS` requires `V14_FS_MARK` (`uniform int uWeapon,uCircuit;uniform float uV7Detail;`)
   to appear **exactly once** and `V14_SKY_OLD` exactly once in whichever source it patches. If
   `FS_R2` is a different shader, **V14 will no longer patch it** and the RGB-atmosphere feature
   (V14) breaks on Ultra. So: if you split `FS_R2`, it must also contain the same `V14_FS_MARK` /
   `V14_SKY_OLD` markers, **or** `v14PatchFS` must be taught to patch both.
4. Protected gameplay numbers stay byte-identical — prices, geometry tables, `PACE`, `WEAPONS`, `MAT`,
   `GORE_TUNING`, `DIRECTOR_TUNING`.
5. **No new RNG streams.** V8/V9/V10 each have their own; do not touch `S.rng` or the global random.
6. Preserve the `finally { settings.quality = q; this.scale = sc; }` restore in `v9Resize`.

---

## PART 7 — VALIDATION CHECKLIST

| # | Check | Expected |
|---|---|---|
| 1 | `settings.quality` on a **fresh** profile | `"ultra"` |
| 2 | `settings.quality` on a profile with an **old** localStorage blob | `"ultra"` (B8 migration) |
| 3 | `v8Lod()` | 3 ultra / 2 high / 1 balanced / 0 low |
| 4 | `uTaps` uploaded vs `AO_FS` clamp | 64 uploaded, `taps` = 64 in-shader (**not 48**) |
| 5 | `uRayStrength` on Ultra + volumetric High | **1.15**, not .69 and not .95 |
| 6 | `uVolumeSteps` on Ultra | 24 |
| 7 | `renderer.shadow.size` / `keyShadow.size` | 2048 / 2048, **and still 2048 after a runtime quality change** |
| 8 | `renderer.w`/`h` on Ultra, 4K display | ≤ 3840 wide |
| 9 | Screen atlas canvas | 1024×1024, **and still 1024×1024 after a runtime quality change** |
| 10 | Particle pool count | 1024, **and still 1024 after a runtime quality change** |
| 11 | In-world screen text | sharp, no wrong-quadrant content |
| 12 | Steam / spark / ember brightness | visibly brighter than V17 |
| 13 | Decal / in-world-screen texture filtering | mipmapped, no shimmer at distance |
| 14 | Dust motes on a HiDPI display | visible as volume, not sub-pixel sparkle |
| 15 | `gl.getParameter(gl.MAX_SAMPLES)` + scene FBO | multisample renderbuffers + blit resolve |
| 16 | **Switch quality at runtime 4× (ultra→low→ultra→high), watch screens, shadows, particles** | no corruption, no stale atlas, no resync |
| 17 | Run low / balanced / high and diff against V17 | **pixel-identical** except intended Ultra-only branches |
| 18 | Fire Sale (V12–V15) still strobes; Strobe Full/Soft/Off still works | unchanged |
| 19 | `v8Stats`/`v9Stats`/`v10Stats`/`v13Stats`/`v14Stats`/`v15Stats` error counters | all zero |

---

## PART 8 — NOT BUGS (checked, do not "fix")

`v8Tile()` UV insets (normalised to the 4×4 atlas — correct for 128px and 256px) ·
the V14 shader-patch mechanism (working) · V16's quality-lock removal (V17's `delete settings.quality`
works) · HDR for world geometry (real: RGBA16F + emit 2.0–3.7 + ACES in post) · the 88° FOV
(a design choice) · `wireControls()` string-vs-number handling ·
`DUST_FS` premultiplication (non-premultiplied + correct blend) · `uWeapon==1`'s fixed 3-light viewmodel rig (intentional) ·
the V10/V11 recorded-audio pipeline.

---

## PART 9 — ONE PARAGRAPH

Start with the structural fix: **only 1 of the 12 sampled lights is shadowed**, so 49% of light
contributions come from lamps behind walls and are applied at full strength (measured: 45% of lit
sample points leak >25%, p90 leak = 100%). Bake a per-light XZ occlusion grid at load (132 KB, one
texture fetch) and the facility stops looking flat. Then the nine defects that nullify the Ultra tier:
AO taps clamp at 48, the volumetric ray strength is overwritten before it reaches the GPU, the V9
resolution mode 1 is a dead branch, MSAA lands on the wrong framebuffer, and shadow maps / the screen
atlas / the particle pool are all frozen at boot while localStorage overwrites the Ultra default.
Three more cheap wins sit in the shader extract: decals and in-world screens have **no mipmaps**, the
AO floor is clamped to .34 so contact shadows can never darken, and dust motes are clamped to 2.8
device pixels so they vanish on HiDPI. After that, raise the tier-invariant caps — 12 lights of 25,
a 132° point-light shadow cone with no cascades, 9-tap PCF, 512² 4-octave value-noise materials with
no normal maps, no IBL, a 2-colour sky, a 4-tap in-pass "bloom", 88° FOV, and a 0.64 resolution floor
that only ever decreases. **Use the already-wired `FS_R2` slot as the Ultra shading variant** — it is
currently an alias of FS, so it is the cleanest way to give Ultra different shading while keeping the
other three tiers byte-identical. And there is room for all of it: the build renders in **3.4–11 ms
at 960×540 in a pure-software rasteriser**, the device reports `maxSamples:4` and `aniso:16` (the code
uses 4 of the 16), and `s3tc`/`astc` are available but unused. Those caps, not the quality ladder,
are the ceiling.
