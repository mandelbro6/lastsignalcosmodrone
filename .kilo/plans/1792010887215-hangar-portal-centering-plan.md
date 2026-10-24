# Hangar Portal Centering Plan

## Context

The hangar spans world X:15-65 (center: 40), Z:-80.5 to -20.5 (back wall at Z:-80.5).
The aircraft ("hangar aircraft" collider) sits at X:26.22-53.78, Z:-51.33 to -25.77, Y:0-8.27 — its nose (tip) points toward the back wall at Z:-51.33.
V44_HANG_X0=40, V44_HANG_Z0=-48 (local→world offset for hangar geometry).

## Decisions

### 1. Portal centered on back wall
- **Current:** `V47_TELE_SITES` hangar entry at `{x:52.5, z:-78.6, yaw:0, s:3.0, base:0}` — too far right.
- **Target:** `{x:40, z:-78.6, yaw:0, s:3.0, base:0}` — center of back wall (X midpoint of 15-65 = 40).
  - `z:-78.6` keeps the ring 0.1m forward of the wall (z:-80.5) so its near edge aligns with the wall face.
  - `base:0` keeps the bottom of the ring at floor level (matches the inverted/mirrored orientation).
  - `yaw:0` faces into the hangar (mirror of portal room's `yaw:Math.PI`).

### 2. Links (teleport triggers) updated
- Update `V47_LINKS` `from` X coords from 52.5 → 40 for both directions.
- Keep `z:-78.9` as the trigger center (midpoint between crate platforms at Z:-77.3 and Z:-80.5).

### 3. Gun box at plane tip
- **Current:** Box at `(40, ~1.3, -50)` with racks at `(41.3, 1.58, -50)` and `(38.58, 1.58, -50)`.
- **Target:** Move the box to the plane's nose tip area. The aircraft collider spans Z:-51.33 to -25.77, so place the box just in front (north side) of the nose at Z:-53:
  - Box center: `(40, 0.7, -53)` — crate platform 3×0.5×2 at this position.
  - Thunder Gun rack: `(41.3, 1.58, -53)`, facing "east" (`u:[0,0,1]`).
  - Portal Gun rack: `(38.58, 1.58, -53)`, facing "east" (`u:[0,0,1]`).
  - Sign positions updated to match.
  - This is in the walkable floor area and clear of the aircraft collider.

### 4. Remove invisible walls and obstacles inside the hangar
The following functions add wall colliders or visual barriers inside/near the hangar:

| Function | Line | What it does | Action |
|---|---|---|---|
| `v47SealHallway` | 2250 | Builds wall colliders at x:-7.85/-4.15, z:35.9-39.7 (portal room area) | **Comment out** (line 2688) |
| `v47HangarBand` | 2362 | Visual mesh walls at y:7.4-20.6 (rim band, no colliders) | **Comment out** (line 2689) |
| `v47HangarSouth` | 2405 | Wall + colliders at z:-20.3 (hangar mouth) | **Comment out** (line 2690) |
| `v46SealMap` | 2136 | Previously disabled map-sealer (already commented out line 2687) | No change needed |
| `v47StepSeal` | 2288 | Not called anywhere; exists only as dead code | No change needed |

**Keep:**
- Basic hangar floor (line 188) — `W.floors` at X:15-65, Z:-80.5 to -20.5
- Hangar perimeter walls (lines 176-179) — these define the hangar boundary
- Hangar pillars (lines 170-172) — structural
- Aircraft collider (line 180) — the plane itself

## Files
- `/workspace/app/cosmodrome-v47.html` (single-file mod, 2833 lines)

## Steps

1. **Edit `V47_TELE_SITES`** (line 247): change hangar `x:52.5` → `x:40`
2. **Edit `V47_LINKS`** (line 291): change both `x:52.5` → `x:40`
3. **Edit `v47GunAdd`** (lines 306-327): change `BOX_X`/`BOX_Z` constants from `(40, -50)` to `(40, -53)`; update all derived positions (racks, signs) to use the new `BOX_Z`
4. **Comment out wall-building calls** (lines 2688-2690): make `v47SealHallway`, `v47HangarBand`, `v47HangarSouth` no-ops by commenting their try/catch blocks
5. **Run syntax check**: `node --check` on extracted script

## Validation
- `node --check` on the `<script>` content must pass.
- Portal ring (s:3.0) centered at X:40, Z:-78.6 — floor bounds of ring: X:37-43, Z:-80.5 to -76.7. No collision with aircraft (X:26.22-53.78 starts at Z:-51.33 — well clear in Z).
- Gun box at Z:-53: clear of aircraft (aircraft starts at Z:-51.33). Racks at X:41.3 and X:38.58, both inside aircraft X range (26.22-53.78) but at Z:-53 which is outside the aircraft Z range (-51.33 to -25.77). No collision.
- Trigger center at (40, -78.9): midpoint between crate platforms at Z:-77.3 and Z:-80.5 is (-77.3 + -80.5)/2 = -78.9. ✓
