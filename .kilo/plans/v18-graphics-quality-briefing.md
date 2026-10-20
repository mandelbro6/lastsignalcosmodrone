# V18 GRAPHICS QUALITY — IMPLEMENTATION BRIEFING (for the builder agent)

**Baseline:** `/workspace/app/cosmodrome-v17.html` (3,112,675 bytes, `cosmodrome-v17-20261017-ultra-tier`)
**Source of truth for this brief:** 100% static read of the V17 document (all 2,098 original lines + all
14,794 lines of the base game expanded from minified line 20).

**Method note:** the base game is one ~3 MB minified line. Patch it the way V17 did — a count-asserted
Python script (`patch-v18.py`) doing exact string replacements with `assert src.count(old) == 1`.
Line numbers in line 20 are meaningless; **use the exact strings below**.

---

## PART 1 — NINE BUGS THAT NULLIFY THE ULTRA TIER

### B1. Ultra's 64 AO taps are silently clamped to 48
`AO_FS` in the base game:
```
int taps=clamp(uTaps,1,48)for(int i=0;i<48;i++)
```
`render()` uploads `v8Lod()===3?64:...` but the shader clamps to 48 → **Ultra AO is byte-identical to High AO.**

**Patch** (two independent replacements):
- `clamp(uTaps,1,48)` → `clamp(uTaps,1,96)`
- `i<48;i++)` → `i<96;i++)`

---

### B2. Ultra's volumetric ray strength `.69` is dead code
`render()` computes and uploads:
```
D=v8Lod()===3?.69:"high"===settings.volumetric?.46:"medium"===settings.volumetric?.23:0
...L=Math.max(O?D:0,_)...return this.f(this.post,"uRayStrength",L)
```
Then `render()` calls `this.configureBodycam(...)` **afterwards**, which overwrites the same uniform:
```
this.f(l,"uRayStrength",d)     // d = "high"===settings.volumetric?.95:"medium"===settings.volumetric?.64:0
```
**The second write wins.** So Ultra never gets `.69`; every tier gets `.95`/`.64`/`0` from
`settings.volumetric`. The "Light shafts: Off" setting *does* work — the dead value is Ultra's own boost.
V17's own header comment claims "volumetric 24 steps / .69 ray strength" — that claim is false.

**Patch** — make `configureBodycam`'s `d` tier-aware (and give Ultra a value above High's `.95`):
```
d = v8Lod()===3 ? ("high"===settings.volumetric?1.15:"medium"===settings.volumetric?.8:0)
                : ("high"===settings.volumetric?.95:"medium"===settings.volumetric?.64:0)
```
(Optionally delete the now-provably-dead `D`/`L` computation in `render()`.)

`uVolumeSteps` is **not** affected — it is written only in `configureBodycam`:
`v8Lod()===3?24:"low"===settings.quality?8:"high"===settings.quality?20:14` — Ultra's 24 works and is
exactly at the shader's ceiling `clamp(uVolumeSteps,1,24)`.

---

### B3. V9 "Render resolution" mode 1 is a dead branch (a V17 regression)
`v9Resize` only has `if (mode === 2) { ...2560 cap... }`. There is **no `mode === 1` branch**, so the
setting labelled *"True 1080p · 1920 cap"* does nothing — you get the quality cap (3840 on Ultra).
In V15 this worked because V15 temporarily set `settings.quality = "ultra"` before calling resize();
V17 removed that hack and the mode died with it. Mode 2 also caps at **2560, below Ultra's 3840**.

**Patch** — add a `mode === 1` branch (1920 cap) and raise mode 2 to 3840. Both must preserve the
`sc` restore in the `finally` block.

---

### B4. MSAA is a no-op
`getContext("webgl2",{alpha:!1,antialias:!0,depth:!0,powerPreference:"high-performance",preserveDrawingBuffer:!1})`
`antialias` applies to the **default framebuffer**. The scene renders into `this.scene.fbo` with
single-sample `DEPTH_COMPONENT24` attachments; the post pass then draws a fullscreen triangle to the
default framebuffer. **There is no `renderbufferStorageMultisample` and no `blitFramebuffer` anywhere
in the file.** MSAA smooths a fullscreen triangle's edges (there are none). All scene geometry is still
single-sampled; FXAA is the only edge AA. Net: pure cost, zero image quality.

**Patch — see PART 2 (real MSAA).** If real MSAA is not implemented, revert `antialias:!0` → `antialias:!1`
(free win, zero visual loss).

---

### B5. Shadow maps never reallocate on a runtime quality change
`Renderer` constructor:
```
this.shadow=this.shadowBuffer(v8Lod()===3?2048:"low"===settings.quality?256:"balanced"===settings.quality?512:1024)
this.keyShadow=this.shadowBuffer(v8Lod()===3?2048:"low"===settings.quality?512:"balanced"===settings.quality?768:1536)
```
`resize()` recreates the scene/ao/hero FBOs and textures but **not** `this.shadow`/`this.keyShadow`.
Changing quality in the menu therefore leaves shadow maps at the boot-tier size. Combined with B7,
existing players get 512/768 maps forever while the dropdown says "Ultra".

**Patch** — add `Renderer.prototype.setQualityTier()` that disposes and recreates both shadow buffers
at the new size, and call it from the quality `oninput` handler. Mirror the existing `shadowBuffer()`
teardown (delete framebuffer + depth texture) — see `resize()` for the exact disposal pattern.

---

### B6. Screen atlas desyncs on runtime quality change
Three sites disagree about tile size:
| Site | Reads LOD | When |
|---|---|---|
| `v8ScreenInit()` — `const tp = v8Lod() === 3 ? 256 : 128;` | at init | **once** (guarded `if (v8ScreenTex) return;`) |
| `v8DrawScreens()` — `const tp = v8Lod() === 3 ? 256 : 128, x = ...` | live | every redraw |
| `v8ScreenTick()` — `gl.texSubImage2D(..., (i % 4) * (v8Lod() === 3 ? 256 : 128), Math.floor(i / 4) * (v8Lod() === 3 ? 256 : 128), ...)` | live | every upload |

Boot on 'high' (512² atlas / 128px tiles) then pick 'ultra' → draw code targets 256px offsets in a 512px
atlas, scales content 2× into a 128px tile canvas (clipped), and uploads a 128px tile at offset 256
(wrong quadrant). All in-world screens — power status, facility schematic, round counter, sign boards —
visibly corrupt.

**Patch — make the atlas tier-independent (recommended, kills the bug permanently).** One 1024² texture is
negligible memory. Three replacements in the V8 module:
- `const tp = v8Lod() === 3 ? 256 : 128;` → `const tp = 256;`
- `const tp = v8Lod() === 3 ? 256 : 128, x =` → `const tp = 256, x =`
- `(v8Lod() === 3 ? 256 : 128)` → `256`  (×2 on the texSubImage2D line)

*Alternative:* rebuild the atlas in `v8ScreenTick()` whenever `v8Lod()` changes. More code, more risk.

---

### B7. Particle pool never resizes (third boot-frozen resource)
`v8PartInit()`:
```
const gl = rend.gl, n = v8Lod() === 3 ? 1024 : v8Lod() === 0 ? 96 : v8Lod() === 2 ? 512 : 256
...
v8PartData = new Float32Array(n * 8)
gl.bufferData(gl.ARRAY_BUFFER, v8PartData, gl.DYNAMIC_DRAW)   // sized once, guarded by v8PartProgram
```
Runtime quality change leaves the pool at the boot-tier size. (V9's own comment at the enemy-FX site
confirms the shared pool: *"pool caps stay V8's (96/256/512)"* — that comment is now stale, Ultra is 1024.)

**Patch** — on quality change, tear down and rebuild the particle pool (VAO + buffer + program) at the
new `n`. Do it in the same `setQualityTier()` as B5.

---

### B8. "Default is Ultra" is false for every returning player
`installV17Hooks()` runs before `boot()`:
```
delete settings.quality
settings.quality = V17_DEFAULT_QUALITY      // "ultra"
```
Then `boot()` merges the saved settings over it:
```
const e = JSON.parse(localStorage.getItem("last-signal-cosmodrome-v5-settings") || "null")
if (e) for (const t of Object.keys(settings)) t in e && (settings[t] = e[t])
```
`wireControls()` writes the **whole** settings blob to localStorage on any slider change, so `quality`
is present in every existing save → **the merge overwrites Ultra**. Ultra reaches fresh installs only.

**Patch** — one-time migration inside `installV17Hooks()` (which runs before `boot()`): read the
`"last-signal-cosmodrome-v5-settings"` blob; if it has no `v17QualityChosen` flag, set
`blob.quality = "ultra"` and write it back. When the user picks a tier, set `v17QualityChosen = true`
so the default never re-applies. (Bump the storage key to `...-v18-settings` instead if you prefer a
clean break — but then migrate the old blob for the other keys.)

---

### B9. Particle additive blending double-multiplies alpha
`V8_PART_FS` outputs **premultiplied** alpha:
```
outColor = vec4(tint * alpha * s, alpha * s)
```
but the draw call uses:
```
gl.blendFunc(gl.SRC_ALPHA, gl.ONE)
```
`SRC_ALPHA` multiplies the already-premultiplied source by alpha a second time → particles render at
roughly **alpha²** brightness. Steam, sparks, embers and thruster plumes are all dimmer than authored.

**Patch** — in the V8 particle draw path only:
```
gl.blendFunc(gl.SRC_ALPHA, gl.ONE)   →   gl.blendFunc(gl.ONE, gl.ONE)
```
⚠️ There are three `blendFunc` calls in the file. **Only the particle one (the 2nd, `SRC_ALPHA, ONE`)
changes.** The dust pass and the glass pass use `SRC_ALPHA, ONE_MINUS_SRC_ALPHA` and must be left alone.

---

## PART 2 — MAKE MSAA REAL (biggest single visual win available)

Currently the scene is single-sampled in `this.scene.fbo` and the post pass runs on the default
framebuffer. To get true MSAA:

1. **Create a multisampled scene FBO** in `resize()` (alongside the existing `scene` FBO):
   - colour: `gl.renderbufferStorageMultisample(gl.RENDERBUFFER, samples, gl.RGBA16F, w, h)`
   - depth: `gl.renderbufferStorageMultisample(gl.RENDERBUFFER, samples, gl.DEPTH_COMPONENT24, w, h)`
   - `samples = Math.min(4, gl.getParameter(gl.MAX_SAMPLES))`
   - if `EXT_color_buffer_float` is unavailable, fall back to `gl.RGBA8`
2. **Render the scene into `sceneMS.fbo`** instead of `scene.fbo`.
3. **Resolve** with:
   ```
   gl.bindFramebuffer(gl.READ_FRAMEBUFFER, sceneMS.fbo)
   gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, scene.fbo)
   gl.blitFramebuffer(0,0,w,h, 0,0,w,h, gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT, gl.NEAREST)
   ```
   Depth blit requires identical formats on both sides — keep `DEPTH_COMPONENT24` on both.
4. The **AO pass** reads `this.scene.color` + `this.scene.depth` — unchanged, it now reads the resolved
   textures. The **post pass** reads `aoColor` — unchanged.
5. **Soften or disable FXAA** when MSAA is on (double-AA mushes edges). `POST_FS` has an `fxaa()`
   function; gate it on a uniform or a settings flag.
6. If any of this proves unreliable, **revert `antialias:!0` → `antialias:!1`** and note in the
   changelog that MSAA was deferred. That is strictly better than the current state.

---

## PART 3 — RAISE THE TIER-INVARIANT CAPS (the real fidelity ceiling)

None of these are reachable by any quality setting. They are what actually cap the image.

| Cap | Current | Target | Where |
|---|---|---|---|
| Dynamic lights | **12 of 25** | 16–24 | `uniform vec4 uLPos[12],uLColor[12],uLPower[12],uLRadius[12]` in the scene FS + the fixed `for(i<12)` loop + the render loop's light selection (`R=[E[0], ...E.slice(1).sort(...).slice(0,11)]` then padded to 12 with dummy lights) + the dummy-light padding count. **All four must move together** or the uniform arrays and the loop disagree. |
| Shadow filtering | **9-tap PCF (3×3)** | 16-tap Vogel disc; **PCSS** on the key light | `shadowPCF()` in the scene FS (uses `sampler2DShadow`, hardware compare) |
| Key shadow frustum | **132° cone, 52 units far** | 3–4 cascades around the camera | `render()`: `this.keyVP = mm(perspective(2.3, 1, .12, 52), lookAt(...))` — this is a *point-light* cone for the brightest lamp, not a sun. A 2048² map spread over 132° is very coarse. **Cascades are the single biggest shadow-crispness win.** |
| Flashlight shadow | re-rendered **every frame**, no architecture in it | add architecture to the `'fill'` map | `render()`: `(k>0||this.shadowDirty)&&this.shadowPass(this.shadow,P,e)`. The `'fill'` map is per-frame at 2048² on Ultra. `v8DrawArch` only draws into the `'key'` map (`this.v8Pass === 'key' && v8Lod() >= 1`). |
| Material atlas | **512²**, 12 layers, **4× aniso** | 1024², 8–16× aniso | `createProceduralPBR(512, 84417)`; `Math.min(4, MAX_ANISOTROPY_EXT)` |
| Texture noise | **4-octave value noise** (4/16/64/128) | gradient/simplex, ≥6 octaves | inside `createProceduralPBR`: `for (let t of [4,16,64,128])` — finest octave = 4-texel features at 512². Blobby. |
| Normal maps | **none** (height only) | real tangent-space normal array | `uRelief` carries R=height, G=metal, B=cavity, A=scatter. Add a third texture array; the shader already has `derivativeRelief` for the fallback. |
| Bump fade | **off beyond 38 m** | remove / extend | scene FS: `micro = 1.-smoothstep(16.,38.,length(uEye-vP))` |
| Ambient / IBL | **none** — 2-colour hemisphere, ambient .15 | 32² irradiance + 128² prefiltered cube | scene FS: `hemi = mix(vec3(.19,.235,.28), uAmbientColor, N.y*.5+.5)`, `ambient = .15` |
| Gloss reflection | **fake 2-colour analytic** | cubemap / SSR | scene FS: `v8env = mix(vec3(.012,.018,.026), vec3(.09,.12,.16), smoothstep(-.15,.55,v8R.y))` |
| Sky | **2-colour vertical gradient** | cubemap / stars / clouds | scene FS: `mix(vec3(.035,.050,.068), vec3(.008,.018,.033), horizon)` where `horizon = clamp((vP.y+9.)/180.,0.,1.)` |
| Tri-planar cost | **6 fetches/pixel** (3 albedo + 3 relief) | keep, but budget-aware | scene FS tri-planar block — this is why Ultra is expensive |
| "Bloom" | **4 taps, radius 6px, threshold .85, weight .025** | real chain: bright-pass → 4–5 downsample/blur → upsample | `POST_FS` — the in-pass 4-tap version. V12–V15 added a LOT of emissive content (RGB strobe lamps, screen panels, cyan/amber strips, rocket fire) that a real bloom would transform. |
| Fog | clamped 0.72 | allow denser + height fog | scene FS: `zfog = clamp(1.-exp(-length(uEye-vP)*uFog*.65), 0., .72)` |
| Far plane | 1600, **hardcoded 3× in post** | make it a uniform | `POST_FS`: `z = .035*1600./max(.00001,1600.-depth*(1600.-.035))` (and two more copies). Breaks if the far plane changes. |
| Dynamic res floor | 0.64, **only ever decreases** | tiered floor + restore upward | frame loop: `e>8e3 && renderer.fps<30 && renderer.scale>.64 && (renderer.scale=Math.max(.64,renderer.scale-.12), renderer.fps=50)` — once a stutter drops you to .64 you stay there for the session, on every tier. |
| Base FOV | **88°** | 78° + user slider | `render()`: `perspective(mix(b,66,o.aim)*Math.PI/180, this.w/this.h, .035, 1600)` |
| HDR | RGBA16F only if `EXT_color_buffer_float`; **zombie path always clamps to LDR** | keep HDR for zombies | scene FS zombie path: `outColor = vec4(clamp(zc0,0.,1.),1.)` — the main path writes unclamped HDR (emit 2.0–3.7) but zombies are clipped, so they never bloom. |
| Gore budget | maxSplats **72**, maxDebris 26, maxSpray 190 | 200 / 40 / 400 | `GORE_TUNING` |
| Dust | **56/floor, not tier-gated** | 32/56/96/160 | `for (let t = 0; t < 56; t++)` in the Renderer constructor |
| Un-scaled geometry | pipes 8 segs, trays 2–5 brackets, small cylinders 6–8, hazard rings 24/32 | scale on `lod > 1` | V8 architecture: `v8Pipe(..., 8, 3)`, `v8CableTray(..., 5)`, `v8HazardRing(..., 24)` / `32`, `m.cylinder(..., 8)`, `m.cylinder(..., 6)` |

---

## PART 4 — NEW CAPABILITY (schedule after Parts 1–3)

- **TAA** — highest AA quality, resolution-independent, biggest effort. Needs: Halton-jittered
  projection matrix, history buffer, velocity buffer (or depth + previous VP reprojection), and a
  history blend in the existing fullscreen post pass. The renderer already has `uTime` and a
  fullscreen post pass, so the infrastructure is close. **This is the biggest long-term upgrade.**
- **SSR** on floors (roughness-thresholded).
- **Cascaded volumetric light shafts** — the raymarch hardcodes `1600.` three times.
- **User-facing rows** the renderer already supports internally:
  - "Resolution" (0.75× / 1× / 1.5× / 2× DPR) — independent of quality; right now 3840 is only
    reachable through Ultra's internal cap and 1440p users cannot ask for 1.5× supersampling.
  - "Shadow quality" (256/512/1024/2048/4096)
  - "Max lights" (4–24)
- **FOV slider**, and an **Ultra preset** that sets quality + resolution + volumetric high + shadow 2048.

---

## PART 5 — ORDER OF WORK

**Phase A — correctness first (cheap, immediately visible).** B1, B2, B3, B4-revert, B5, B6, B7, B8, B9.
Every one of these is a small, count-asserted string replacement. They are what currently make Ultra
a no-op. Do them before any feature work — otherwise you will be debugging visual regressions against
a broken baseline.

**Phase B — real MSAA.** (Or revert to `antialias:!1` and defer.)

**Phase C — raise the caps.** Pick the highest-impact-per-risk first:
C1 cascaded key shadows → C2 16-tap/PCSS → C3 lights 12→24 → C4 material atlas 1024² + aniso 8–16× →
C5 real normal maps + better noise → C6 IBL cubemap → C7 real sky → C8 multi-pass bloom →
C9 the remaining table rows.

**Phase D — TAA and the new user-facing rows.**

---

## PART 6 — REGRESSION GUARDS (V17's plan required "no regression" for low/balanced/high)

V17 broke the hooks-only discipline: it used source edits (`patch-v17.py`) because `v8Lod` is a `const`
and cannot be rebased at runtime. Continue that approach for anything inside the base engine
(Renderer, shaders, `createProceduralPBR`) and keep hooks for anything reachable via prototype
wrapping. Whatever you do:

1. **low / balanced / high must behave exactly as in V17.** Only Ultra gains. Add a count-asserted
   check that the non-ultra branches of every edited expression are byte-identical.
2. **The strobe system (V15) is out of scope.** Do not touch `V15_SOFT_HZ`, `V15_SOFT_FLOOR`,
   `v15PaintSoft`, `v15PaintAtmSoft`, `v15OffPaint`, or the V15 `strobeLevel` UI row.
3. **V12/V13/V14 (RGB spectacle / strobe / RGB atmosphere) are out of scope.** Do not change
   `V13_STROBE_HZ`, `V13_STROBE_DUTY`, `V13_STROBE_FLOOR`, `V13_SPEED`, `V14_MUSIC_GAIN`,
   `V14_SKY_A_BASE`, `V14_SKY_B_BASE`, `V14_FOG_BASE`, `V14_HEMI_BASE`, or the
   `V14_FS_MARK`/`V14_SKY_OLD`/`V14_SKY_NEW` shader-patch strings (the count-asserted
   `v14PatchFS` will fail its `c(x) !== 1` / `c(x) !== 2` checks if you alter them).
4. **Protected gameplay numbers stay byte-identical** — prices, geometry tables, `PACE`, `WEAPONS`,
   `MAT`, `GORE_TUNING` tuning constants, `DIRECTOR_TUNING`.
5. **No new RNG streams.** V8/V9/V10 each have their own; don't touch `S.rng` or the global random.
6. **V9's `v9res` mode 1/2 must not break the `finally { settings.quality = q; this.scale = sc; }`
   restore** — that block is what keeps the quality tier and the scale consistent.

---

## PART 7 — VALIDATION CHECKLIST

Run these in a browser console / one-frame diff and record the numbers:

| # | Check | Expected |
|---|---|---|
| 1 | `settings.quality` on a **fresh** profile | `"ultra"` |
| 2 | `settings.quality` on a profile with an **old** localStorage blob | `"ultra"` (B8 migration works) |
| 3 | `v8Lod()` | 3 for ultra, 2 high, 1 balanced, 0 low |
| 4 | AO uniform + `AO_FS` clamp | `uTaps` = 64 uploaded, `taps` in shader = 64 (**not 48**) |
| 5 | `uRayStrength` value on Ultra + volumetric High | **1.15**, not .69 and not .95 (B2) |
| 6 | `uVolumeSteps` on Ultra | 24 |
| 7 | `renderer.shadow.size` / `keyShadow.size` on Ultra | 2048 / 2048, **and still 2048 after a runtime quality change** (B5) |
| 8 | `renderer.w`/`renderer.h` on Ultra, 4K display | ≤ 3840 wide (B3) |
| 9 | Screen atlas canvas size on Ultra | 1024×1024, **and still 1024×1024 after a runtime quality change** (B6) |
| 10 | Particle pool count on Ultra | 1024, **and still 1024 after a runtime quality change** (B7) |
| 11 | In-world screen tile text legibility | sharp, no wrong-quadrant content (B6) |
| 12 | Steam / spark / ember brightness | visibly brighter than V17 (B9) |
| 13 | `gl.getParameter(gl.MAX_SAMPLES)` and the scene FBO | multisample renderbuffers present + blit resolve (Part 2) |
| 14 | **Switch quality at runtime 4× (ultra→low→ultra→high) and watch the screens, shadows, particles** | no corruption, no stale atlas, no resync (B5/B6/B7) |
| 15 | Run low / balanced / high and diff against V17 | **pixel-identical** except the intended Ultra-only branches |
| 16 | Fire Sale show (V12–V15) still strobes, and the Strobe Full/Soft/Off row still works | unchanged |
| 17 | `v8Stats` / `v9Stats` / `v10Stats` / `v13Stats` / `v14Stats` / `v15Stats` error counters | all zero |

---

## PART 8 — WHAT I DID *NOT* FIND

To save you time, these are **not** problems in V17 (I checked specifically):

- The `v8Tile()` UV insets (`.005`–`.245`) are **correct** for both 128px and 256px tiles — the inset is
  normalised to the 4×4 atlas, so changing the tile size does not need a UV change.
- The V14 shader-patch mechanism (`v14PatchFS`, `V14_FS_MARK`, `V14_SKY_OLD`/`NEW`) is **working** —
  the sky/fog/hemi uniforms are correctly promoted at runtime. Don't touch it.
- The V16 quality lock is **successfully removed** by V17's `delete settings.quality` (the accessor was
  `configurable: true`), and low/balanced/high are real choices again. The V17 comment about "v8Lod is a
  const and cannot be rebased at runtime" is accurate — that is why source edits were used.
- HDR *is* real for the world geometry (RGBA16F + emit values 2.0–3.7 + ACES tone mapping in the post
  pass). Only the **zombie** path clamps to LDR. The `EXT_color_buffer_float` fallback to RGBA8 on
  devices without float colour buffers is the real risk, not the pipeline design.
- The base FOV of 88° is a deliberate design choice, not a bug — but it is very wide and worth a slider.
- The `wireControls()` string-vs-number handling (`["quality","imaging","volumetric"].includes(e)`) is
  correct; `quality` is read as a string everywhere and `v8Lod()` compares against strings.

---

## PART 9 — ONE-PARAGRAPH SUMMARY

Ultra exists and is wired, but nine defects nullify it: AO taps clamp at 48, the volumetric ray
strength is overwritten before it reaches the GPU, the V9 resolution mode 1 is a dead branch, MSAA
lands on the wrong framebuffer, and shadow maps / the screen atlas / the particle pool are all frozen
at boot while localStorage overwrites the Ultra default. Fix those nine first — they are all small
count-asserted string replacements. Then attack the tier-invariant caps, which are what actually cap
fidelity: 12 lights of 25, a 132° point-light shadow cone with no cascades, 9-tap PCF, 512² 4-octave
value-noise materials with 4× aniso and no normal maps, no IBL, a fake 2-colour gloss reflection, a
2-colour sky gradient, a 4-tap in-pass "bloom", an 88° FOV, and a dynamic-resolution floor of 0.64
that only ever decreases. Those caps, not the quality ladder, are the ceiling.
