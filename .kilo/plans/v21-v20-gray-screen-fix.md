# V20 → V21 FIX LIST (the gray-screen regression)

**Target:** `/tmp/.../c3979c9d-cf2b-48ca-8475-3305ada8cc9d.txt` (`cosmodrome-v20-20261020-direct-render`, 2877 lines)
**Baseline:** `/workspace/app/cosmodrome-v17.html` (V17, staged)

## What you are looking at right now

V20 is a **diagnostic fallback ladder**, not the renderer. Line 2574:

> `// V20 "DIRECT RENDER": eliminate the black frame stage by stage`
> `// V18.1 hardened NaN/MSAA; V19 added a safe-shader tier. The world was still black…`
> `//   1 compat  2 scene-safe  3 ao-post-safe  4 ldr  5 direct`

The ladder is `compat → scene-safe → ao-post-safe → ldr → direct`. It advanced because the **real
renderer is genuinely black**. What you see is the fallback's deliberately flat placeholder shader
(line 2597-2611): no PBR, no GGX specular, no shadows, no AO, no tri-planar relief, no wear, no
curvature, no IBL — just `base*(amb + .40*key + .18*fill)`. The "lights" you can see are the emissive
strips (vM 9/10/11/18/19), which the fallback still emits.

**The real renderer went black when V18 added real MSAA.**

---

## ROOT CAUSE — production blits MSAA differently than the probe tests

**Production resolve** (line 2404):
```js
gl.blitFramebuffer(0,0,this.w,this.h, 0,0,this.w,this.h, gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT, gl.NEAREST);
```

**The probe that "proves" MSAA works** (line 2153):
```js
gl.blitFramebuffer(0,0,4,4, 0,0,4,4, gl.COLOR_BUFFER_BIT, gl.NEAREST);
```

A **multisample depth resolve is not supported in WebGL2** on many drivers, and when
`blitFramebuffer` errors the **entire call is discarded — including the color**. So:

1. `this.scene.msFbo` is created (line 2198) and the base render is patched to draw into it
   (`t.bindFramebuffer(t.FRAMEBUFFER, this.scene.msFbo||this.scene.fbo)`).
2. `v18Resolve()` (line 2404) tries to blit MS→`scene.fbo`; the call is rejected → **`scene.fbo` is
   never written**.
3. The AO pass (line ~2404+, immediately after) reads `this.scene.color` → black.
4. The post pass samples `aoColor` → black.
5. World renders black behind a perfectly working DOM HUD.

**Why the auto-test never caught it:** `v18MsaaResolveTest` probes with `COLOR_BUFFER_BIT` **only**,
so it *passes* on a driver where the combined production call fails. It validates a code path
production never uses. The verdict is also cached per context (`r.v18MsaaOk`), so once `true` it is
trusted for the session.

Patched base render order is otherwise correct — resolve happens before the AO pass:
```js
... this.v18Resolve() && this.v18Resolve(),
t.bindFramebuffer(t.FRAMEBUFFER,this.scene.aoFbo), ... t.bindTexture(t.TEXTURE_2D,this.scene.color) ...
```

---

## FIX A — split the resolve so a depth failure cannot void the color (line 2398-2408)

```js
Renderer.prototype.v18Resolve = function () {
  try {
    const gl = this.gl, ms = this.scene && this.scene.msFbo;
    if (!ms) return;
    gl.bindFramebuffer(gl.READ_FRAMEBUFFER, ms);
    gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, this.scene.fbo);
    // COLOR must be its own call: a rejected blit discards the WHOLE call.
    gl.blitFramebuffer(0,0,this.w,this.h, 0,0,this.w,this.h, gl.COLOR_BUFFER_BIT, gl.NEAREST);
    const colorErr = gl.getError();
    // DEPTH resolve is best-effort: a multisample depth resolve is unsupported on many drivers.
    gl.blitFramebuffer(0,0,this.w,this.h, 0,0,this.w,this.h, gl.DEPTH_BUFFER_BIT, gl.NEAREST);
    const depthErr = gl.getError();
    while (gl.getError() !== gl.NO_ERROR) {}
    if (depthErr !== gl.NO_ERROR) this.v18NoDepthResolve = true;   // AO depth stale
    if (colorErr !== gl.NO_ERROR) { this.scene.msFbo = null; this.resize(); }
    gl.bindFramebuffer(gl.FRAMEBUFFER, null);
    v18Stats.resolves++;
  } catch (e) { v18Errors.push('resolve:' + e.message); }
};
```
Then in the AO pass, if `this.v18NoDepthResolve`, set `uAmount = 0` (skip SSAO) — a missing AO is far
better than a missing world.

## FIX B — make the probe test the production call, and verify the real target

Do not trust a 4×4 mock. After the **first real resolve**, probe the actual texture:
```js
// one-shot, in v18Resolve after the color blit
if (!this.v18MsaaVerified) {
  this.v18MsaaVerified = true;
  const m = v18ProbeMean(this, this.scene.color);      // NOT aoColor
  if (m >= 0 && m < .2) {                              // scene.color is black -> resolve did not land
    this.v18MsaaOk = false; this.scene.msFbo = null; this.resize();
    V18_DIAG.autoFixes.push('MSAA resolve produced a black scene - disabled.');
  }
}
```
Also reset `r.v18MsaaOk = null` on `webglcontextrestored` (line 2544 currently only calls `v20DiagAuto`).

## FIX C — immediate mitigation, zero risk

MSAA on the **default** framebuffer (`antialias:!0`) does nothing anyway: the scene renders into an
FBO, and the post pass draws a fullscreen triangle. Set `settings.v18msaa = 0` by default and revert
the context flag `antialias:!0` → `antialias:!1`. Removes the failure mode at no visual cost.

**Your instant fix, right now, no code change:** open **Settings → "MSAA (V18)" → "Off"**.
`v18MakeTargets` then allocates no MS FBO, the base render binds `scene.fbo` directly, and the real
renderer comes back. Or append **`?safe=1`** to the URL (forces compat: no MSAA/TAA/bloom/IBL, real
renderer). `?simple=1` and `?direct=1` also exist.

---

## Still broken after the MSAA fix

### 1. `v8CableTray` — brackets placed off the end of the tray (line 214-215)
```js
const nb18 = v8Lod() > 1 ? brackets * 2 : brackets;
for (let i = 0; i < nb18; i++) { const f = (i + .5) / brackets; ... }
```
The count doubles but the divisor does not. On high/ultra `f` runs to ~2.0, so **half the brackets sit
twice down the length of the cable run**, floating in mid-air.
**Fix:** `const f = (i + .5) / nb18;`

### 2. `v8HazardRing` — the ring is traced twice (line 236-242)
```js
const sg18 = v8Lod() > 1 ? segs * 2 : segs;
for (let i = 0; i < sg18; i++) { const a0 = i / segs * Math.PI * 2, ... }
```
`sg18` doubles but the angle step stays `i/segs` → the second lap overlaps the first exactly:
doubled geometry and a visible seam.
**Fix:** `const a0 = i / sg18 * Math.PI * 2, a1 = (i + .5) / sg18 * Math.PI * 2, am = (a0 + a1) / 2;`

### 3. `uRayStrength` is still overwritten (base, `configureBodycam`)
The base writes `uRayStrength` twice: `render()` uses `.46/.23/0`, then `configureBodycam()` overwrites
it with `.95/.64/0`. Ultra's `.69` never reaches the GPU (V18's `f` wrap only adds `uFxaa`/`uFar`/
`uBloomAmt` in the same branch). Minor, but V17's header still claims "volumetric … .69 ray strength".
**Fix:** make `configureBodycam`'s `d` tier-aware, or delete the dead `render()` write.

### 4. V20 ladder step 3 ('ldr') is a one-way trap (line 2667-2672)
It monkey-patches `gl.getExtension` to permanently hide `EXT_color_buffer_float`, then calls
`resize()` — so float targets are gone for the rest of the session even if the user changes
settings. **Fix:** restore the original `getExtension` when the step is undone.

### 5. V20 ladder step 4 ('direct') has no way back (line 2674-2685)
It nulls `r.scene.fbo` and swaps in the discard program. Only a reload restores the real renderer.
**Fix:** add a "restore full renderer" path that rebuilds the scene FBO and restores `r.program`,
`r.aoProgram`, `r.post`.

### 6. `v18-quality-chosen` is only set by the preset button (line 2367)
The localStorage migration (line 2113-2117) forces `quality='ultra'` for any player who has not set
that flag. But the flag is set **only** by "Apply Ultra preset" — a manual change in the Graphics
dropdown does not set it, so the next load re-forces Ultra over the player's choice.
**Fix:** set the flag from the quality select's `oninput` (or in `wireControls`).

---

## Already fixed since the V15 analysis (confirmed in V20)

| Item | V20 state |
|---|---|
| B1 AO taps clamp 48 | ✅ `clamp(uTaps,1,96)` + loop `i<96` |
| B2 uRayStrength overwrite | ❌ still overwritten (see above) |
| B3 V9 `v9res` mode 1 dead branch | ✅ modes 1 **and** 2 implemented (line 914) |
| B4 MSAA | ❌ **the black screen** |
| B5 shadow maps never realloc | ✅ `setQualityTier()` (line 2374) |
| B6 screen atlas desync | ✅ tier-independent `tp = 256` |
| B7 particle pool never resizes | ✅ rebuilt in `setQualityTier()` |
| B8 "default is Ultra" false | ✅ V18 localStorage migration (line 2113) |
| B9 particle blend double-alpha | ✅ `blendFunc(gl.ONE, gl.ONE)` (line 476) |

Also improved since V15/V17: the sky shader now has stars and 3-octave cloud noise
(`V14_SKY_OLD`/`V14_SKY_NEW`, lines 1834-1835) — the 2-colour gradient is gone. Light arrays were
raised to 24 consistently (`uLPos[24]`, loop `i<24`, upload padded to 24, `settings.v18lights`).

---

## Priority order

1. **Turn MSAA off by default** + revert `antialias` to `!1` → the real renderer returns immediately
2. **Split the resolve + probe the real `scene.color`** → MSAA can come back safely
3. `v8CableTray` / `v8HazardRing` divisors → two one-line geometry fixes
4. Make ladder steps 3 and 4 reversible
5. `v18-quality-chosen` flag from the dropdown
6. Only then resume the V18/V20 feature work (bloom, TAA, IBL) — none of it has been visually
   validated yet, because the scene has been black the whole time
