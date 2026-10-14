# V8 Futuristic Enhancement Prompt for Cosmodrome Game

## Target: `last-signal-cosmodrome-v8.html`

Additive, hooks-only V8 layer for futuristic animations and architectural enhancements.

---

## 1. FUTURISTIC ANIMATIONS

### 1.1 Holographic Projector Props (V8_PROP_HOLO)
New prop type for ceiling-mounted holographic displays.

```javascript
function v8Hologram(m, x, y, z, rot, k) {
  const co = Math.cos(rot), si = Math.sin(rot);
  m.cylinder([x, y, z], [x, y + 2.5, z], 0.15, 12, [0.1, 0.4, 0.8], 12);
  m.box(x, y + 2.65, z, 0.35, 0.15, 0.35, 15, [0.2, 0.6, 1.0], [0, rot, 0]);
  for(let i = 0; i < 6; i++) {
    const a = i / 6 * Math.PI * 2;
    m.box(x + Math.cos(a) * 0.45, y + 0.2 + i * 0.35, z + Math.sin(a) * 0.45, 
          0.08, 0.1 + 0.05 * Math.sin(a * 7) * k, 0.08, 10, [0.15, 0.7, 1.0], [0, rot, 0]);
  }
}
```

### 1.2 Energy Barrier Decals
Add glowing energy barriers around breaches during rounds.

```javascript
function v8EnergyBarrier(s, breach) {
  const t = s.time * 3;
  for(let i = 0; i < 8; i++) {
    const a = i / 8 * Math.PI * 2;
    const r = 0.8 + 0.15 * Math.sin(t + i);
    projectSplat(s, 
      [breach.x + Math.cos(a) * r, breach.y + 0.5, breach.z + Math.sin(a) * r],
      [Math.sin(t * 2 + i), 0.1, Math.cos(t * 2 + i)],
      0.4 + 0.1 * Math.sin(t * 5), 
      MAT.ENERGY
    );
  }
}
```

### 1.3 Particle Pulse System
Synchronized pulse effects for machines during power-on sequence.

```javascript
function v8MachinePulse(original, s, g) {
  const out = original(s, g);
  if(s.world.power && g.type === 'machine' && s.time % 4 < 0.1) {
    for(let i = 0; i < 12; i++) {
      const a = i / 12 * Math.PI * 2;
      const r = 1.5;
      projectSplat(s,
        [g.x + Math.cos(a) * r, g.y + 0.8, g.z + Math.sin(a) * r],
        [Math.sin(s.time * 2), Math.cos(s.time), Math.cos(s.time * 2)],
        0.25 + 0.05 * (s.time % 1),
        MAT.METALLIC
      );
    }
  }
  return out;
}
```

---

## 2. ARCHITECTURAL ENHANCEMENTS

### 2.1 Reflective Surface Material (MAT.REFLECTIVE)
New PBR material for futuristic panels.

```javascript
const MAT = {
  RUBBER: 0, METALLIC: 1, STONE: 2, WOOD: 3,
  ENERGY: 4, REFLECTIVE: 5, NEON: 6, GLASS: 7
};
```

Shader addition (GLSL 300 es):
```glsl
vec3 reflectiveBSDF(vec3 N, vec3 V, vec3 L, vec3 albedo, float roughness) {
  vec3 R = reflect(-L, N);
  float m = pow(1.0 - roughness, 10.0);
  vec3 fresnel = mix(albedo, vec3(1.0), pow(1.0 - max(dot(N, V), 0.0), 5.0));
  return fresnel * m;
}
```

### 2.2 Dynamic Holographic Architecture
Procedural overlay geometry for existing structures.

```javascript
function v8HologramOverlay(rend) {
  if(!rend || !v8HoloMesh) {
    const mesh = new Mesh();
    const rng = v7Rand(0xBEEF1234);
    for(const f of WORLD.floors) {
      if(f.zone && f.zone.indexOf('CONTROL') >= 0) {
        for(let i = 0; i < 3; i++) {
          const x = f.x0 + 0.5 + rng() * (f.x1 - f.x0 - 1);
          const z = f.z0 + 0.5 + rng() * (f.z1 - f.z0 - 1);
          v8Hologram(mesh, x, f.y + 0.5, z, rng() * Math.PI * 2, rng());
        }
      }
    }
    v8HoloMesh = rend.gpu(mesh);
    rend.models.holograms = v8HoloMesh;
  }
  if(v8HoloMesh) {
    rend.draw(v8HoloMesh, identity(), SHADER.V8HOLOGRAM);
  }
}
```

### 2.3 Power Grid Visualization
Wire-like connections between machines when power is on.

```javascript
function v8PowerGrid(original, vp, program) {
  original(vp, program);
  if(!s.world.power) return;
  
  const machines = WORLD.machines;
  for(let i = 0; i < Math.min(machines.length, 6); i++) {
    for(let j = i + 1; j < machines.length; j++) {
      const a = machines[i], b = machines[j];
      if(dist2(a, b) < 8 && Math.abs(a.y - b.y) < 0.5) {
        const t = s.time * 3;
        const wire = new Mesh();
        wire.line([a.x, a.y + 0.3, a.z], [b.x, b.y + 0.3, b.z], 
                  0.05 + 0.02 * Math.sin(t), [0, 1, 0.8], 8 + 4 * Math.sin(t));
        rend.draw(wire, identity(), SHADER.V8WIRE);
      }
    }
  }
}
```

---

## 3. INTEGRATION HOOKS

### 3.1 V8 Tick Enhancement
```javascript
function v8Tick(original, s, dt, controls) {
  v8Lamps(s.time);
  v8DrawHolograms(s);
  return original(s, dt, controls);
}
```

### 3.2 V8 Draw Enhancement
```javascript
const v8Static = Renderer.prototype.drawStatic;
Renderer.prototype.drawStatic = function(vp, program) {
  v8Static.call(this, vp, program);
  v8DrawProps(this, program);
  v8HologramOverlay(this);
  v8PowerGrid.call(this, vp, program);
};
```

### 3.3 V8 Explosion Enhancement
```javascript
function v8Explode(original, s, g) {
  const out = original(s, g);
  if(s.world.power) v8EnergyBarrier(s, g);
  return out;
}
```

---

## 4. SHADER DEFINITIONS

Add to shader array:

```javascript
SHADER.V8HOLOGRAM = `#version 300 es
precision highp float;
uniform mat4 uProjection, uView;
uniform float uTime;
in vec3 aPosition;
in vec2 aUV;
out vec2 vUV;
void main() {
  vUV = aUV;
  gl_Position = uProjection * uView * vec4(aPosition, 1.0);
}`;

SHADER.FRAGMENT = `precision highp float;
in vec2 vUV;
uniform float uTime;
uniform vec3 uColor;
void main() {
  float d = distance(vUV, vec2(0.5));
  float alpha = 0.3 + 0.4 * sin(uTime * 5.0);
  float edge = smoothstep(0.45, 0.5, d);
  vec3 col = uColor + 0.1 * sin(uTime * 3.0 + vec3(0.0, 2.0, 4.0));
  gl_FragColor = vec4(col, alpha * (1.0 - edge));
}`;
```

---

## 5. WORLD RNG SEEDING

```javascript
function v8Rand(seed) {
  return mulberry32((seed >>> 0) || 0x5EED7013);
}
```

---

## 6. INSTALLATION HOOKS

```javascript
function installV8Hooks() {
  tick = v8Tick.bind(null, tick);
  explodeGrenade = v8Explode.bind(null, explodeGrenade);
  Renderer.prototype.drawStatic = (function(orig) {
    return function(vp, program) {
      orig.call(this, vp, program);
      v8DrawProps(this, program);
      v8HologramOverlay(this);
    };
  })(Renderer.prototype.drawStatic);
}

installV7Hooks(), installV8Hooks(), boot();
```

---

## 7. DETERMINISTIC BEHAVIOR REQUIREMENTS

- All random choices must use `v7Rand(0x5EED7013)` for byte-identical reproduction
- Props never collide (physics unchanged)
- Effects are purely visual
- No modifications to core economy, routes, or world build

---

## 8. QUALITY TIER INTEGRATION

Use existing quality settings for effect intensity:

```javascript
const v8Intensity = {'low': 0.3, 'balanced': 0.6, 'high': 1.0}[settings.quality];
```

---

## Summary

V8 adds:
1. **Holographic projector props** - ceiling displays with animated geometry
2. **Energy barrier visual effects** - glowing defenses around breaches
3. **Power grid visualization** - wire connections when power is active
4. **Pulse particle effects** - synchronized machine energy pulses
5. **Reflective/Neon materials** - shader extensions for futuristic surfaces

All hook into existing V7 pipeline via `installV8Hooks()` for seamless integration.