# Fire Sale Feature Implementation Plan

## Goal
Add a Black Ops-style Fire Sale feature to cosmodrome-v10.html that discounts all purchases to 10 points for 30 seconds, triggered by collecting a fire sale crate dropped by zombies.

## Target File
`cosmodrome-v10.html` - the V10 build with audio mode features

## Implementation Location
Insert fire sale code **before `boot()` definition** and add `installFireSaleHooks()` to the hook chain at line 938.

## Key Integration Points

### 1. Hook Chain Modification (line 938)
**Current:**
```javascript
installV5Hooks(),installV6Hooks(),installV7Hooks(),installV8Hooks(),installV9Hooks(),boot();
```

**New:**
```javascript
installV5Hooks(),installV6Hooks(),installV7Hooks(),installV8Hooks(),installV9Hooks(),installFireSaleHooks(),boot();
```

### 2. Game State Dependencies
The game already has:
- `S.defense.trace` for player points
- `S.defense.pickups` array for drop collection
- `spawnDrop()` for enemy death drops
- `interact()` for purchases (`spend()` call)
- `interactable()` for price display
- `updateHUD()` for powerups display
- `audio.tone()` and `audio.burst()` for procedural synth
- `$` helper for DOM element access

### 3. Fire Sale Code Insertion Point
Insert before `boot();` definition (around line 937 in current file). The code will:
- Define fire sale constants and state variables
- Provide procedural synth audio placeholder (`fsBarStep`, `fsStartSting`, `fsEndSting`)
- Wrap `tick` for music scheduling
- Wrap `defenseStep` for pickup collection
- Wrap `spawnDrop` to spawn fire sale crates
- Wrap `interact` for price escrow logic
- Wrap `interactable` for price display
- Wrap `updateHUD` for countdown display

## Audio Implementation Options

### Option A: Procedural Synth (Default, No External Assets)
Uses existing `audio.tone()` and `audio.burst()` primitives. No file changes needed.

### Option B: Embedded Licensed Audio
If a licensed MP3/OGG is available:
```javascript
const AUDIO_SRC = `data:audio/mpeg;base64,PASTE_BASE64_HERE`;
const fireSaleMusic = new Audio(AUDIO_SRC);
fireSaleMusic.loop = true;
fireSaleMusic.volume = 0.4;
fireSaleMusic.preload = "auto";
```

**Note:** Cannot use YouTube URL directly - must be embedded as data URI or local file.

## Fire Sale Crate Visualization
The crate will render as the gold bonus sphere (using existing `circuitPieces` → `dropBonus` rendering path). For visual distinction from regular drops, consider adding a dedicated mesh in `createModels`.

## Testing Checklist
1. [ ] Kill zombies → fire sale crate appears every ~32 kills
2. [ ] Collect crate → "FIRE SALE! EVERYTHING 10 POINTS!" message
3. [ ] HUD shows "FIRE SALE 30s · ALL BUYS 10"
4. [ ] All purchases (doors, weapons, perks, mystery box) cost 10 pts
5. [ ] Music plays during sale (synth or embedded)
6. [ ] After 30s → sale ends, music stops, prices revert
7. [ ] No crate spawn while sale is active
8. [ ] Pause/resume correctly pauses/stops music

## Validation Steps
1. Verify syntax with browser developer tools console
2. Test in browser with WebGL support
3. Confirm no external network requests are made
4. Verify audio context is not blocked by autoplay policies (requires user gesture)

## References
- Guidance file with complete hook implementation: lines 1-627 of the provided text
- Game hook pattern: `installV7Hooks`, `installV8Hooks`, `installV9Hooks` (lines 138-936)
- Hook chain: line 938