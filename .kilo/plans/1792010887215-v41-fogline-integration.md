# V41 Fogline Integration Plan — Bodycam Post-Processing & Weather Systems

## Goal
Wire the already-transplanted V40 fogline weather/camera/lens code (rain, camera bob, lens dirt) into the cosmodrome render loop, and add the missing **Wy** post-processing class (bloom pyramid, auto-exposure, lens drops, motion blur, scope overlay, film grain, ACES tone mapping).

## Status of V40 Code (already present in target file, NOT wired)
| Function | Line in target | Depends on | Current state |
|---|---|---|---|
| `v40RainFrame(rend)` | 428 | `rend.lastEye`, `rend.lastVP`, `rend.gl`, `rend.programOf/m/v` | Defined, never called |
| `v40CamState(o,t)` | 510 | player state `o`, time `t`, globals `S` and `input` | Defined, never called |
| `v40CamEye/Pitch/Yaw/RollV` | 555-564 | `v40CamState`, `eyeOf()` | Defined, never called |
| `v40LensTick(o,dt)` | 566 | `v40IndoorProbe`, `v40Ceil` | Defined, never called |
| `v40CoverInit/Step` | 300-332 | `WORLD.floors`, `WORLD.colliders`, `floorAt` | Defined, never called |

## Renderer API (from `last-signal.txt`)
The base `Renderer` class (`last-signal.txt:3888`) provides:
- `gl` — WebGL2 context (`this.gl`)
- `w`, `h` — viewport dimensions
- `scene = { fbo, color, depth, aoColor, aoFbo, heroFbo, hdr }`
- `emptyVAO` — bound for fullscreen triangle draws
- `programOf(vs, fs)`, `loc(p, n)`, `f(p, n, v)`, `i(p, n, v)`, `v(p, n, v)`, `m(p, n, mat)`
- `render(s)` method (line 5412) — `s` is game state, `s.p` is player, `s.time` is time
  - Line 5424: camera eye/direction computed inline from `pl = s.p`
  - Line 5437: `this.frameEye = eye` (camera position stored, but NOT `this.lastVP`)
  - Line 5535: `this.drawStatic(vp, p)` — **VP matrix passed here**
  - Line 5565: AO pass → `this.scene.aoColor`
  - Line 5618-5638: POST_VS/POST_FS pass to default framebuffer (uses `this.scene.aoColor`)

## Critical Gaps
1. The V40 functions require `rend.lastEye` and `rend.lastVP` on the renderer, but the base renderer only sets `this.frameEye` (line 5437). VP (`vp`) is a local variable in render(), passed to `drawStatic(vp, p)` at line 5535 (`drawStatic(vp, p = this.program)` at line 4008). The V41 patch must set `this.lastEye` and `this.lastVP`.
2. The base renderer's `settings` has NO `bodycam`, `fisheye`, `grain`, `lensRain` fields — Wy params must be derived from `settings.imaging` ('clear'|'recorded'|'bodycam') + `settings.imageStrength`.
3. `settings.quality` values are 'low'|'balanced'|'high'|'ultra' (NOT fogline's 'low'|'medium'|'high'|'ultra').
4. Player state `s.p` has: `vx,vy,vz` (velocity), `aim`, `sprintFov`, `shake`, `pitch,yaw,roll`, `stamina` (0-100), `sliding` — all compatible with `v40CamState(o,t)`. The global `input` has `input.strafe`.

## Integration Steps

### Step 1: Add `lastEye` / `lastVP` capture via drawStatic hook

Hook `Renderer.prototype.drawStatic` to capture VP, and `render` to set `lastEye`:

```js
function installV41Hooks() {
  // Capture lastEye and lastVP via drawStatic (called at render line 5535)
  const origDrawStatic = Renderer.prototype.drawStatic;
  Renderer.prototype.drawStatic = function(vp, p) {
    this.lastVP = vp;              // store VP matrix for v40RainFrame
    this.lastEye = this.frameEye;  // frameEye set at render line 5437
    return origDrawStatic.call(this, vp, p);
  };
  // ... (remainder of Step 1 below)
}
```

### Step 2: Wire render() to call v40RainFrame + Wy pipeline

Wrap `Renderer.prototype.render` to:
1. Apply camera overrides from v40CamState (modify player state before base render computes camera)
2. Let original render() run (scene + AO + hero + POST_FS)
3. After original render: call `v40RainFrame(this)` (draws rain particles to screen)
4. Replace POST_FS output with Wy by feeding `this.scene.aoColor` to Wy

```js
  const origRender = Renderer.prototype.render;
  let wy = null;  // Wy instance, created lazily on first render

  Renderer.prototype.render = function(s) {
    // --- Pre: Camera overrides via v40CamState ---
    // v40CamState returns {pitch, yaw, roll, ex, ey, ez, bobR}
    // We add deltas to pl so base render's camera uses fogline-style bob
    if (pl && s.time) {
      try {
        const st = v40CamState(pl, s.time);
        pl._v40Pitch = st.pitch;  pl._v40Yaw = st.yaw;  pl._v40Roll = st.roll;
      } catch(e){}
    }

    // --- Original render (scene + AO + hero + POST_FS) ---
    const result = origRender.call(this, s);

    // lastEye/lastVP now available via drawStatic hook
    // --- Post: Rain frame ---
    try { v40RainFrame(this); } catch(e){}

    // --- Post: Wy post-processing (lens dirt, grain, scope overlay) ---
    if (!wy && this.gl) wy = new Wy(this.gl, this);
    if (wy) {
      if (this._wyResW !== this.w || this._wyResH !== this.h) {
        wy.resize(this.w, this.h, this.gl);
        this._wyResW = this.w; this._wyResH = this.h;
      }
      wy.params.lastEye = this.lastEye || [0,0,0];
      wy.params.lensDirt = v40LensTick(pl, 1/60);
      wy.render(this.scene.aoColor, this, s);
    }

    return result;
  };
```

**Rationale**: Wy runs as an OVERLAY pass after POST_FS. It reads `this.scene.aoColor` (scene color after AO) and applies its FULL pipeline (bloom, exposure, composite, grain, scope, lens drops) to the default framebuffer. POST_FS provides the base tone-mapping; Wy's FINAL_FS adds the fogline-specific grain, scope overlay, signal loss, and compression artifacts on top.

### Step 3: Camera integration via v40CamState

The base render computes camera angles from `pl.pitch`, `pl.yaw`, `pl.roll` (line 5428). v40CamState returns deltas we need to add. Two options:

**Option A (recommended)**: Modify `pl` state before base render reads it. The render wrapper sets `pl.height` to include v40 eye offset, and adds v40 pitch/yaw/roll to pl's computed angles. Since the base camera uses `pl.pitch + pl.recoil + sway + ...`, we add v40 deltas to `pl.recoil` or a dedicated field.

```js
// In render wrapper, before origRender:
const st = v40CamState(pl, s.time);
pl._v40Bob = st;  // store for later
// Modify pl.pitch/yaw/roll to include v40 deltas
pl.pitch += st.pitch * 0.5;   // damp to avoid over-shaking with base
pl.roll  += clamp(st.roll, -0.26, 0.26);
// Eye height offset
pl.viewBob = (pl.viewBob || 0) + st.ey;
```

**Option B**: Hook `eyeOf()` to add v40 eye deltas, and hook camera angle computation. More surgical but harder to target.

Use **Option A**. The v40CamState already accounts for sprint, aim, stamina, landing. Dampening by 0.5 prevents double-shake with base renderer.

### Step 4: Port Wy class (adapted to cosmodrome renderer API)

Port from `/workspace/app/fogline.html` (883 lines, all 6 shaders) into a class that uses the renderer's WebGL context and uniform helpers instead of standalone globals:

```js
class Wy {
  constructor(gl, rend) {
    this.gl = gl;
    this.rend = rend;
    // Shaders: POST_VS, PRE_FS, DOWN_FS, UP_FS, EXP_FS, COMPOSITE_FS, FINAL_FS
    // (same shader sources from fogline.html lines 66-437)
    this.matPre  = gl.createProgram(...) via rend.programOf(POST_VS, PRE_FS);
    this.matDown = rend.programOf(POST_VS, DOWN_FS);
    this.matUp   = rend.programOf(POST_VS, UP_FS);
    this.matExp  = rend.programOf(POST_VS, EXP_FS);
    this.matA    = rend.programOf(POST_VS, COMPOSITE_FS);  // composite
    this.matB    = rend.programOf(POST_VS, FINAL_FS);       // final
    // Framebuffers: bloom pyramid (4 levels), exposure chain (3 levels), ping-pong
    // Noise texture for grain/lens dirt
    // params object (from fogline.html line 512-523, adapted)
  }

  resize(w, h) {
    // Create/update FBOs: sceneRT, ldrRT, bloom down/up, lum chain, expRT
    // Use gl.createFramebuffer + gl.texImage2D
  }

  render(sceneTex, rend, s) {
    // Pass 0: PRE_FS — bloom extraction from sceneTex → bloom[0]
    // Pass 1-3: DOWN/UP — bloom pyramid
    // Pass 4-5: EXP — auto-exposure from luminance
    // Pass 6: COMPOSITE_FS (matA) — composite to ldrRT
    // Pass 7: FINAL_FS (matB) — final tonemap + grain + scope → default framebuffer
    // Use rend.m/v/f/i/loc for uniforms (same API as base renderer)
  }
}
```

Key adaptations from fogline.html:
- Replace global `gl` → `this.gl`
- Replace `createProgram(vs, fs)` → `this.rend.programOf(vs, fs)`
- Replace `setF/setI/setV2/setV3/setV4/setTex` → `this.rend.f/i/v/m/loc`
- Replace `createRT(w,h)` → inline framebuffer creation using `this.gl`
- Replace `quadVAO` → `this.rend.emptyVAO`

### Step 5: Quality-tier gating

Map cosmodrome's `settings.quality` to Wy parameters and rain intensity:

```js
const qualityTier = () => {
  const q = settings.quality;
  if (q === 'ultra') return 3;
  if (q === 'high') return 2;
  if (q === 'balanced') return 1;
  return 0;  // low
};

const v40QualityScale = () => {
  return { 0: 0.4, 1: 0.7, 2: 1.0, 3: 1.0 }[qualityTier()];
};

// In installV41Hooks, set v40Intensity:
const q = qualityTier();
v40Intensity = q <= 0 ? 0.3 : q === 1 ? 0.7 : 1.0;
```

Bloom levels by quality: `q === 0 ? 2 : q === 1 ? 3 : 4`
Bloom threshold: `1.0` (quality doesn't change)
Lens dirt: gated by `settings.imaging === 'bodycam'`

### Step 6: Settings mapping (cosmodrome → Wy params)

```js
const imaging = settings.imaging;  // 'clear' | 'recorded' | 'bodycam'
const strength = settings.imageStrength;  // 0-1, default 0.5
const q = qualityTier();
const isBodycam = imaging === 'bodycam';
const isRecorded = imaging === 'recorded';

wy.params.bodycam    = isBodycam ? 1 : isRecorded ? 0.7 : 0;
wy.params.fisheye    = isBodycam ? 1 : isRecorded ? 0.7 : 0;
wy.params.grain      = isBodycam ? 1 : isRecorded ? 0.8 : 0;
wy.params.compress   = isBodycam ? 1 : isRecorded ? 0.8 : 0;
wy.params.ca         = isBodycam ? 1 : isRecorded ? 0.7 : 0;
wy.params.ca2        = isBodycam ? 1 : isRecorded ? 0.7 : 0;
wy.params.vignette   = 0.55;
wy.params.motionBlur = settings.motion * 0.6;  // scaled by quality
// Rain lens: driven by v40LensTick (weather + shelter status)
wy.params.rainLens   = 0; // set per-frame by v40LensTick
```

Apply settings at construction and on `setSetting` changes. Hook `settingsChanged` event or call via `renderer.setSetting('quality', val)`:
```js
// In Wy constructor or applySettings:
const imaging = settings.imaging;
const strength = settings.imageStrength;
wy.params.bodycam  = (imaging === 'bodycam' ? 1 : imaging === 'recorded' ? 0.7 : 0) * strength;
// etc.
```

### Step 7: Hook chain + initialization

Modify the hook chain at target file line 849:
```
// FROM:
installV40Hooks(),boot()
// TO:
installV40Hooks(),installV41Hooks(),boot()
```

Also add `installV41Hooks` and `Wy` to the IIFE scope (same as other `installVXHooks` functions), so the hook chain can call it. The Wy class should also expose `getFovScale()` (fogline.html:755) as an optional hook for renderer FOV override.

## Files to Modify

| File | Location | Change |
|---|---|---|
| Target file (V41 cosmodrome, line ~837) | After `installV40Hooks()` | Add `installV41Hooks()` function with Wy class, render wrapper, drawStatic hook, quality gating |
| Target file line 849 | hook chain | Add `installV41Hooks()` between `installV40Hooks()` and `boot()` |

## Data Flow

```
render(s) wrapper:
  1. v40CamState(pl, s.time) → camera deltas → modify pl.pitch/roll/viewBob
  2. origRender(s):
     a. Camera setup (uses pl with v40 deltas applied)
     b. drawStatic(vp, p) → sets this.lastVP, this.lastEye
     c. Scene/AO/Hero rendering → this.scene.aoColor
     d. POST_FS pass → screen (base tone mapping)
  3. v40RainFrame(this) → draws rain particles (uses lastEye, lastVP)
  4. Wy.render(this.scene.aoColor, this, s):
     a. Bloom extraction from aoColor
     b. Bloom pyramid (down/upsample)
     c. Auto-exposure (luminance histogram)
     d. COMPOSITE_FS (matA) → lens drops, CA, bloom, ACES → ldrRT
     e. FINAL_FS (matB) → grain, scope overlay, compression, signal loss → screen
```

## Validation Checklist

- [ ] `v40RainFrame` called every frame when renderer exists
- [ ] `v40LensTick` computed per-frame and passed to Wy as `uLensDirt`
- [ ] `v40CamState` deltas applied to player camera before base render
- [ ] Wy bloom pyramid renders at quality-tier resolution
- [ ] `this.lastEye` and `this.lastVP` set via drawStatic hook
- [ ] ACES tone mapping active when `settings.imaging === 'bodycam'`
- [ ] Rain intensity scales with `settings.quality` (v40Intensity)
- [ ] No regression in existing V5-V17 hooks (wrapper pattern preserves original)
- [ ] Wy `getFovScale()` accessible for optional renderer FOV override
- [ ] Wy fallbacks gracefully if GL/webgl2 unavailable
