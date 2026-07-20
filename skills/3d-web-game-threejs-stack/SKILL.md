---
name: 3d-web-game-threejs-stack
description: "Build production-quality 3D web games with Vite + Svelte 5 + Three.js r180 + pmndrs/postprocessing. Covers scene architecture, camera controllers, projectile systems, wireframe/outline gotchas, cinematic locks, FPS-style controls with keyboard fallback, AABB collision vs building footprints, sticky-ground edge cases, top-landing checks, server-snap reconciliation, pointer lock + mouselook + ESC hint HUD, OS-keydown autorepeat filter, the build/verify loop, the 2D-client-to-3D feature port pattern (HUD + audio + minimap + kinds), the \"use the same logic, draw in 3D\" pattern (porting 2D parsing/analysis: spatial grid + style seeds + cell composition), the global canvas !important conflict when adding HUD mini-canvases, ResizeObserver-driven mini-canvas scaling, and the F1 debug panel for tilequery shapes. Use for any browser-based 3D game or interactive 3D scene that lives in a Vite project (standalone or monorepo)."
version: 1.4.0
author: [auto]
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags:
      - three.js
      - 3d
      - game
      - webgl
      - vite
      - svelte
      - postprocessing
      - tron
    related_skills: [cloudflare, workers-best-practices]
    category: creative
---

# 3D Web Game Stack (Three.js + Vite + Svelte 5 + pmndrs/postprocessing)

A battle-tested recipe for shipping a 3D web game that loads in <1s,
runs at 60fps on mid-range laptops, and supports both mouse and pure-keyboard
play. Validated against NEONGRID (https://github.com/4rgs/neongrid), a
3D Tron-inspired prototype with low-poly cel-shaded assets, infinite
loop levels, and a global leaderboard.

## When to use this skill

- You're building a browser-based 3D game or interactive 3D scene
- You want sub-200KB first-paint and 60fps on commodity hardware
- You need to ship both mouse-driven and pure-keyboard control
  (touch controls and gamepads are easy follow-ons)
- You're targeting Vite + a UI framework (Svelte, React, Vue) +
  Three.js + a postprocessing chain
- You're porting this stack as a 3D frontend into a monorepo that
  already has a 2D client or a NestJS/Express/Rails backend
  (see `references/client-for-existing-backend.md`)

## When NOT to use

- 3D model viewer / config viewer with no interaction loop → just
  use `<canvas>` with OrbitControls, no game loop needed
- Mobile-first casual game → consider React Native + Expo, or
  Capacitor + this stack
- Photoreal cinematic experience → reach for PlayCanvas or
  Unreal Web, this stack optimizes for stylized low-poly art

## The recipe

```toml
# package.json (relevant parts)
{
  "dependencies": {
    "three": "^0.180",
    "postprocessing": "^6.36",
    "svelte": "^5"
  },
  "devDependencies": {
    "@sveltejs/vite-plugin-svelte": "^5",
    "vite": "^5.4"
  }
}
```

```ts
// vite.config.ts
export default defineConfig({
  target: 'es2022',         // modern enough, smaller bundle
  build: {
    modulePreload: true,    // critical for fast first paint
    rollupOptions: {
      output: {
        manualChunks: {
          'three': ['three'],                       // lazy-loaded (122 KB)
          'post': ['postprocessing'],               // lazy-loaded (17 KB)
        },
      },
    },
  },
});
```

Bundle targets:
- `index.html` < 1 KB gzip
- App code 30-45 KB gzip
- `postprocessing` 17 KB gzip
- `three` 122 KB gzip (lazy)
- First paint: **~50 KB**, cold load: **~175 KB**

## Project structure (NEONGRID shape)

```
src/
  main.ts                  # boot, mount Svelte HUD, await HUD api, then construct Game
  game/
    palette.ts             # PALETTE = { cyan, orange, magenta, green, red }
    Input.ts               # keyboard Set + mouse deltas; orbit, axis, click edges
    Hero.ts                # the player; outline-per-part, AABB collision
    World.ts               # scene primitives: grid, terrain, traces
    Components.ts          # procedural decoration objects
    Enemies.ts             # 5 archetypes (patrol/turret/hunter/charger/drone)
    Boss.ts                # 3 variants (octa/spinblade/voidtank) per level
    Disc.ts                # free-flying projectile in WORLD space
    Fragments.ts           # collectibles
    Particles.ts           # fixed-size pool (no allocations in hot path)
    CameraController.ts    # WoW-style orbit, dampened yaw/pitch/distance
    Cinematic.ts           # keyframe-driven scripted camera (boss intro, level transitions)
    PostFX.ts              # bloom + chromatic aberration + scanlines + ACES
    TrailRenderer.ts       # line-strip trail behind hero
    Audio.ts               # Web Audio synth (no asset loading required)
    SaveSystem.ts          # localStorage persistence with versioning
    Game.ts                # orchestrator: scene + systems + loop + tick
    Leaderboard.ts         # global leaderboard client (submit + fetch + cache)
    ReplayRecorder.ts      # per-frame input log → upload on game-over
  ui/
    App.svelte             # HUD overlay: Svelte 5 runes
public/
  _headers                 # Cloudflare Pages cache headers
  _redirects               # SPA fallback
schema.sql                 # D1 schema (leaderboard + replays tables)
functions/                 # Cloudflare Pages Functions (server-side)
  api/scores.ts
  api/replay.ts
wrangler.toml              # Pages + D1 binding config
deploy.sh                  # one-shot deploy script
```

For the "this stack as a 3D frontend for an existing REST + WS backend"
shape, add `Backend.ts` (HTTP client), `Chunks.ts` (or similar for
your spatial endpoint), and `Players.ts` (WS client). See the
port-existing-backend reference for the full file list.

## Common pitfalls (NEONGRID's scars)

### "Wireframes appear sunken below bodies"

**Symptom**: every `LineSegments` outline in your procedural world sits
visibly lower than the body it represents. Towers, chips, capacitors all
look "broken" from the ground.

**Root cause**: in each `buildXxx()` method of a procedural builder, you
add the body mesh with `body.position.y = h * 0.5` so the body sits
half-height above the terrain. Then you `this.group.add(new LineSegments(...))`
without setting a local position — so the wireframe lives at local Y=0.
The group transforms both equally, but the wireframe ends up `h * 0.5`
metres below the body.

**Fix**: store the LineSegments in a local var, set
`edges.position.y = h * 0.5` BEFORE adding to the group. Do this for
every procedural builder in the project. Refactor candidate: a helper
`addWithOutline(geo, mat, localY)` that does both.

```ts
// Before (broken)
this.group.add(new THREE.LineSegments(
  new THREE.EdgesGeometry(body.geometry),
  new THREE.LineBasicMaterial({ color: PALETTE.cyan }),
));

// After (fixed)
const edges = new THREE.LineSegments(
  new THREE.EdgesGeometry(body.geometry),
  new THREE.LineBasicMaterial({ color: PALETTE.cyan }),
);
edges.position.y = h * 0.5;
this.group.add(edges);
```

### "Projectiles rotate with the camera"

**Symptom**: a "disc-whip" / "fireball" that should fly straight ahead
curves back to the hero when the player rotates the camera.

**Root cause**: you parented the projectile to `hero.group` so it
inherits the hero's transform. When the hero (or camera) rotates, the
projectile's "forward" vector rotates with it.

**Fix**: parent the projectile to `scene`, not to the hero. Compute the
throw direction ONCE from the camera yaw at the moment of fire, freeze it,
and let the projectile fly in world space.

```ts
// Hero class
tryAttack(cameraYaw: number) {
  const fwd = new THREE.Vector3(-Math.sin(cameraYaw), 0, -Math.cos(cameraYaw));
  this.disc.launch(this.group.position.clone().setY(1.0 + 0.02), fwd);
}
// Game.ts setup
this.scene.add(this.hero.disc.group);   // NOT this.hero.group.add(this.disc.group)
```

### "Cinematic + player movement stack on top of each other"

**Symptom**: during the boss intro cinematic, the hero keeps moving
and the boss takes damage from off-camera disc-whips.

**Fix**: early-return at the start of `tick()` when the cinematic is
running. Only the cinematic itself updates.

```ts
private tick(dt: number, t: number) {
  if (this.cine.running) {
    this.cine.update(dt);
    return;   // ← no input, no hero, no enemy AI, no damage
  }
  // ... rest of the game loop
}
```

This also fixes the "boss of new level insta-kills the hero during the
cinematic" problem — the hero's HP isn't touched while the world is
frozen.

### "Hero strafes AND the camera orbits at the same time when A is held"

**Symptom**: with A/D bound to strafing AND camera orbit, the player
gets both effects and feels sluggish.

**Root cause**: in `Input.axis()` you returned both `x` and the orbit
applied its own yaw rotation in the same frame.

**Fix** (option A — recommended for this user): separate keys entirely.
**WASD** for strafe/move, **Q/E** (or arrow keys) for camera orbit.
`Input.axis()` returns `x: 0` for A/D in the orbit-key scheme.

```ts
axis(): { x: number; z: number } {
  let x = 0, z = 0;
  if (this.keys.has('KeyW') || this.keys.has('ArrowUp'))    z -= 1;
  if (this.keys.has('KeyS') || this.keys.has('ArrowDown'))  z -= 1;
  if (this.keys.has('KeyA') || this.keys.has('ArrowLeft'))  x -= 1;
  if (this.keys.has('KeyD') || this.keys.has('ArrowRight')) x += 1;
  const len = Math.hypot(x, z);
  if (len > 0) { x /= len; z /= len; }
  return { x, z };
}
// In Game.ts: bind camera orbit to Q/E (and R/F for tilt), not A/D
const turnLeft  = this.input.held('KeyQ') || this.input.held('ArrowLeft');
const turnRight = this.input.held('KeyE') || this.input.held('ArrowRight');
```

This is the WoW/Overwatch convention and what the user wanted in the
NEONGRID sessions. If you accidentally reverse it and bind A/D to
orbit, the player will tell you twice; **take the second correction as
the final answer**, not as a contradiction — the user's preference
depends on whether they want mouse-free play (A/D orbit) or shooter-style
play (A/D strafe, Q/E orbit). See the sibling skill
`web-3d-game-control-schemes` for the full decision tree and runtime
probes.

### "Dev server boots blank, body is empty, __hud is undefined"

**Root cause**: Vite HMR breaks when **true top-level `await`** appears in a synchronously-imported module. Symptoms: blank `<body>`, `__game` and `__hud` both `undefined`, no FATAL banner.

`await` wrapped inside an `;(async () => {...})()` IIFE in `main.ts` is FINE — Vite handles that. The failure mode is awaiting at the module top level outside any function.

**Fix**:
1. Verify with `npm run build && npm run preview` — the prod build works fine.
2. For dev: avoid top-level await in `*.ts` files imported synchronously. Either use IIFE `;(async () => {...})()` or move the await into a separate bootstrap module that's dynamically imported.

This is a Vite bug, not a code bug — do NOT add workarounds that hide the error.

**Don't confuse this with** the actual blank-blank-blank failure mode observed in client-3d June 2026: stack started but `__hud = undefined` because `waitForHud` polled methods (`setTutorial`, `setBossHp`) that the placeholder App.svelte didn't expose. Different cause, similar symptom. Lesson: the App.svelte HUD MUST implement every method `waitForHud` expects, even as no-ops.

### "WASD works after `npm run dev` but stops after `npm run build && npm run preview`"

**Root cause**: `vite preview` does NOT process the `vite.config.ts` `server.proxy` block by default in Vite 5. Even when docs claim "preview.proxy defaults to server.proxy", a SPA-fallback index.html can mask proxy misroutes — `curl :4174/api/foo` returns HTML 200 with a healthy-looking response, hiding the fact that the API is unreachable.

**Fix**: explicitly define `preview: { proxy: { ... } }` in `vite.config.ts` mirroring `server.proxy`, OR develop with `vite dev` and only build/preview for production deploys.

### "Enemies pass through the hero without dealing damage"

**Root cause**: the projectile aimed at the hero was given `dir.setY(0.7)`
in `Enemies.shoot()` to make it fly at chest height, but at 14 m/s the
projectile gains `+0.7 m/s` Y velocity — so it arcs UP and over the
hero (passes 1-2m above their head) within ~1 second.

**Fix**: compute `dir` so it points at the hero's actual Y (chest
height ~0.3m above the ground), not "setY(0.7)". If the difference is
<0.5m, flatten entirely to XZ for a horizontal bullet.

```ts
const dir = playerPos.clone().sub(this.group.position);
const dy = dir.y;
dir.y = 0;
dir.normalize();
if (Math.abs(dy) < 0.5) {
  // horizontal bullet at hero's chest height
  const start = playerPos.clone().setY(0.3);
} else {
  // angled bullet, scale velocity slightly to compensate for gravity-less arc
  const start = this.group.position.clone().setY(this.group.position.y + 0.7);
}
```

Also: collision radius. With a projectile at 14 m/s and a 60fps tick,
each frame advances 0.23m. If your collision radius is 1.0m, you get
tunneling. **Use 1.4m minimum for fast bullets.**

### "Enemy disc-projectile appears below the ground"

**Symptom**: the bullet fires but you can't see it; it appears as a
bright dot somewhere near the ground far behind the hero.

**Root cause**: bullet geometry has a fixed local Y (0.7m), but the
parent group sits at world Y=0 with a terrain heightmap; the bullet
renders at world Y=0.7m while the hero is at world Y=terrain. The
bullet is below the terrain for hero-on-hill, above terrain for hero-
in-valley.

**Fix**: spawn the bullet at `playerPos.y + 0.3` (chest) and make the
bullet's Y trajectory match the hero's Y as the bullet travels. Easiest
implementation: store the initial `playerPos.y` at launch and keep the
bullet at that Y for its entire flight (flat trajectory, no arc).

### "Boss takes 1 hit and dies"

**Symptom**: the boss has `hp = 10` and the disc deals `damage = 1`,
so 10 hits = 1 frame of contact.

**Fix**:
- Add `invincibilityFrames` after each hit (typical: 0.18s ≈ 11 frames).
  During those frames, hits are ignored.
- Scale HP per level: `hp = 30 + 20 * (level - 1)` (NEONGRID formula).
- Scale damage per level: `damage = 1 + Math.floor(level / 2)`.
- Tie attack patterns to HP percent: at <30% HP go "enrage" (faster,
  multi-shot, back-dash).
- Add a stagger visual: flash white for 0.05s, push the boss back 0.3m.

### "Finalization sequence when the player collects the last fragment"

**Symptom**: collecting the last fragment just spawns the boss with no
warning. The player misses the moment of victory and the boss appears
out of nowhere.

**Fix**: trigger a dedicated sequence between "last fragment collected"
and "boss spawned". This makes the transition feel earned:

```ts
private finalizeFragmentRun() {
  if (this.bossSpawned) return;
  this.bossSpawned = true;   // claim immediately to prevent double-spawn

  // 1. White flash overlay (HUD-side, 0.9s, fade-out keyframe)
  this.hud.setFlash('white', 0.9);
  // 2. Screen shake
  this.shake(0.6, 1.0);
  // 3. Lore banner
  this.hud.setLore('// ALL FRAGMENTS COLLECTED // the master process awakens');
  // 4. Audio
  this.audio.warning();
  // 5. Cyan particles burst at hero position
  this.particles.burst(this.hero.group.position.clone().setY(1.5), PALETTE.cyan, 120, 8, 1.2);
  // 6. Spawn the boss after a delay so the player can register the flash
  setTimeout(() => this.spawnBoss(), 1600);
}
```

The HUD `setFlash` method injects a full-screen overlay with a CSS
keyframe animation. Two presets work well:

```css
.flash-white { background: rgba(255,255,255,0.85); }
.flash-red   { background: rgba(255, 34, 85, 0.7); }   /* game over */
@keyframes flash-fade-white { 0% {opacity:0.95} 100% {opacity:0} }
```

Use the same `setFlash('red', 0.7)` for game over, with a 0.7s delay
before the name-entry modal opens so the death registers first.

Pair this with an **extended 5s boss reveal cinematic** (3 keyframes
becoming 5), not 3.6s. The player needs time to take in the new boss
model, the level-up effects, and the audio warning before combat
begins.

### "Boss contact damage feels toothless"

**Symptom**: touching the boss does `hero.hurt(1)` and the player only
loses 1 HP per contact, so they can casually walk through the boss and
survive multiple hits.

**Fix**: bosses should feel dangerous. Make contact an instakill with
strong feedback:

```ts
if (this.boss.alive && this.boss.group.position.distanceTo(this.hero.group.position) < 4.5) {
  this.damageThisLevel++;
  const dead = this.hero.hurt(999);          // instakill
  this.hud.setHp(this.hero.hp, this.hero.maxHp);
  this.audio.hurt();
  this.shake(0.6, 0.7);
  // Massive red particle burst so the player understands what hit them
  this.particles.burst(this.hero.group.position.clone().setY(1), PALETTE.red, 200, 12, 1.4);
  if (dead) this.startGameOver();
}
```

Contact radius 4.5m (was 2.5m) — generous because the boss is wide. Pair
this with a 0.7s red flash on the HUD + delayed name modal so the
death registers before the player is asked for their handle.

**Workflow note**: when the user reports "boss doesn't feel dangerous
enough" or asks to soften/tune boss contact damage, **the default
direction is instakill, not a soft cooldown**. NEONGRID bounced between
`hurt(1)+0.7s` (too soft — projectiles flew through unnoticed and the
fight felt toothless) and `hurt(999)` instakill (the right answer).
The user's preference is binary — they want the boss to feel like a
*threat you avoid*, not a wall you scrape. Don't iterate through
intermediate damage values; go straight to instakill with strong
feedback. If you ever soften it, make sure the boss's OTHER damage
sources (projectiles) are working first — otherwise the fight becomes
"safe to ignore".

### "Boss projectiles fly through the hero without dealing damage"

**Symptom**: the boss shoots cyan/magenta spheres at the hero, the
hero's HP doesn't drop, the projectiles sail past untouched. Looks
like the boss's attacks are visual-only.

**Two root causes, both can ship together**:

**Root cause 1 — unchecked projectile pool**: the projectile-collision
loop in `tick()` iterates
`for (const e of this.enemies) { for (const p of e.projectiles) { ... } }`
— i.e. only enemy projectiles get collision-tested against the hero.
Boss projectiles live in a separate pool (`this.boss.projectiles`,
created by `Boss.spawnProjectile`, updated inside `boss.update()`)
and **nothing checks them against the hero**. They drift off-world
until their `life` expires.

This bug is invisible when boss body contact is instakill (the hero
dies on touch before any projectile can land). The moment you soften
body contact to per-tick damage, the projectiles become the only
real threat and the missing collision becomes obvious.

**Root cause 2 — 3D sphere hitbox misses every high shot**: even with
the loop wired correctly, the standard
`p.mesh.position.distanceTo(hero.position) < 2.0` check is a
**sphere** around the hero, not a cylinder. Boss projectiles are
spawned at `group.position + (0, 2.5, 0)` — 2.5m above terrain — and
travel near-horizontally. The hero's pivot is at Y=0, body at Y=0..2.
A projectile that passes "directly over" the hero at 2.5m altitude
has a 3D distance of ≥ 2.5m from the hero's pivot — **above** the
2.0m sphere. The shot sails through the hitbox and the player sees
"the boss's shots go through me."

**Combined fix** (apply both — the loop without the cylinder still
misses, the cylinder without the loop never fires):

```ts
// Inside the boss block in tick() — mirror the enemy projectile loop
for (const p of this.boss.projectiles) {
  const dx = p.mesh.position.x - this.hero.group.position.x;
  const dz = p.mesh.position.z - this.hero.group.position.z;
  const distXZ = Math.hypot(dx, dz);
  if (distXZ < 2.0) {   // ← cylinder, not sphere
    if (this.shopDamagePaused) { p.life = 0; continue; }
    p.life = 0;
    this.particles.burst(p.mesh.position.clone(), PALETTE.cyan, 18, 4, 0.5);
    const dead = this.hero.hurt(p.damage);
    this.hud.setHp(this.hero.hp, this.hero.maxHp);
    this.audio.hurt();
    this.shake(0.25, 0.3);
    if (dead) this.startGameOver();
  }
}
```

Use `p.damage` rather than a hardcoded value — boss projectile damage
scales with level (`1 + floor((level-1)/2)` in NEONGRID) and you want
the collision check to respect that. Sphere vs cylinder:

| Projectile behavior                         | Use         |
|--------------------------------------------|-------------|
| Travels at a fixed Y (boss shots, most NPC bullets) | **XZ cylinder** |
| Arcs through the air (grenade, gravity-affected) | 3D sphere  |
| Player-fired disc that may go up/down      | 3D sphere  |
| Hero's own weaponized attack                | 3D sphere  |

**Diagnostic recipe** (when "boss projectiles don't hit" is reported):
1. Spawn a real boss via the game's spawn path, or stub one with
   `projectiles: []` populated with a mesh at known position.
2. Drop a projectile at `hero.position + (1.5, 5, 0)` (XZ inside the
   hitbox, Y well above the hero so a sphere check would miss).
3. Sample `hero.hp` and `proj.life` after one tick: if `proj.life=0`
   and `hero.hp` dropped, the loop works. If `proj.life` is still
   positive, the loop is either gated wrong or not running at all.

Also check the boss aim for a `setY(0.7)` / `setY(1.5)` shortcut
that **overwrites** the Y component of the direction vector
instead of the target. Pair that with the sphere check and the
projectile is simultaneously above the hero AND outside the 2m
radius. Fix: use the actual 3D direction (no `setY` hack) and let
the fire-origin height be the only Y adjustment.

See `web-3d-game-control-schemes` §12D/E for the deeper
explanation, worked code, and runtime probes.

## The Vite + Svelte 5 boot pattern

```ts
// main.ts
const appEl = document.getElementById('app')!;

(async () => {
  const hud = mount(App, { target: appEl });
  const api = await waitForHud();   // polls window.__hud until Svelte $effect populates it
  const game = new Game(appEl, api);
  game.start();
  (globalThis as any).__game = game;
})();
```

```svelte
<!-- App.svelte -->
<script lang="ts">
  $effect(() => {
    (window as any).__hud = {
      setScore: (n) => score = n,
      setHp: (n, max) => { hp = n; hpMax = max; },
      // ... every setState method exposed by the HUD
    };
  });
</script>
```

The HUD always exists; the game waits for `__hud` to be populated before
constructing. This avoids the classic "Game constructs before HUD
exists, throws on `hud.setScore()`" race.

## HUD api separation: setFlag vs triggerFetch

When a HUD overlay has both a visibility flag AND a data fetch (e.g.
"open leaderboard page" → flip `showLeaderboardPage = true` AND call
`fetchTop(...)` to populate it), the temptation is to combine them
in one HUD method. That breaks because:

```ts
// ❌ INFINITE LOOP — both paths flip the flag and call each other
setLeaderboardPage: (v: boolean) => {
  showLeaderboardPage = v;
  if (v) (window as any).__game.openLeaderboardPage('all');
}
// ...and inside openLeaderboardPage:
async openLeaderboardPage(period) {
  this.hud.setLeaderboardPage(true);   // ← recursion back into the same method
  const top = await fetchTop(20, period);
  // ...
}
```

The page crashes with `RangeError: Maximum call stack size exceeded`.
Fix: split by intent:

| Method                          | Purpose                              | Side effects                |
|---------------------------------|--------------------------------------|-----------------------------|
| `hud.setLeaderboardPage(v)`     | Just flip the flag (open/close UI)   | None                        |
| `game.openLeaderboardPage(p)`   | Fetch data + flip the flag            | fetch + `hud.setLeaderboardPage(true)` |

Button handlers call the data-fetching method (which also flips the
flag). External code that only wants to flip the flag (backdrop click
to close) calls the flag-only method. The Game method always flips the
flag for you; the flag-only method never triggers the fetch.

Same pattern applies to `setShop`, `setSettings`, `setLeaderboardPage`,
`askForName` — if a HUD API *both* changes HUD state *and* triggers
Game-side work, split it into two methods. See the sibling skill
`web-3d-game-control-schemes` §3 for the full worked example.

## Camera controller pattern (WoW-style)

```ts
applyOrbit(dx: number, dy: number) {
  // dx positive → yaw decreases → world turns RIGHT
  this.yaw -= dx * this.YAW_SPEED;
  this.pitch = clamp(this.pitch + dy * this.PITCH_SPEED, MIN_PITCH, MAX_PITCH);
}
update(dt: number, target: THREE.Vector3) {
  // Smoothed follow
  this.position.lerp(desired, 1 - Math.exp(-dt * this.SMOOTH));
  this.lookAt(target);
}
```

- **Yaw inverted** (drag-right = world-right): every mouse delta goes
  in negative. This is the "natural" feel users expect.
- **Pitch clamped** between 0.1 and 1.4 radians. Beyond that the
  camera clips or flips.
- **Smoothing**: `1 - exp(-dt * k)` is frame-rate independent. `k = 8`
  is comfortable for follow cam, `k = 4` for cinematic.

### Camera target Y must follow the player (don't use a fixed Y)

**Symptom**: with a `setTarget(x, 1.4, z)` style orbit cam, the
hero's Y is fixed at 1.4 (offset to the torso when the player
is on the ground). When the player jumps to `y = 3.5` and the
user has zoomed out (`distance = 30`), the camera still aims at
`y = 1.4` — 2.1m below the player. The player flies out of the
frame at the top because the lookAt point is below them.

**Fix**: the target Y must follow the player's actual visual Y.
Use the offset pattern: `targetY = GROUND_OFFSET + hero.group.position.y`,
where `GROUND_OFFSET` is the constant distance from the
player's feet to the lookAt point (e.g. 1.4m to aim at the
torso of a ~1.7m avatar).

```ts
// In Game.ts tick():
const heroWorld = lngLatToWorldMeters(this.hero.lngLat);
const heroVisualY = this.hero.group.position.y;
this.cameraCtl.setTarget(heroWorld.x, 1.4 + heroVisualY, heroWorld.z);
this.cameraCtl.place(this.camera);
```

**Caveats**:
- The offset is avatar-proportions-specific. If the avatar is
  2m tall instead of 1.7m, bump the offset from 1.4 to 1.6
  so the camera still aims at the torso.
- For top-landing on colliders: `hero.group.position.y` is
  already the visual Y (which includes the collider height).
  The camera will track 1.4m above the rooftop. Same math.
- This applies to any orbital cam with variable zoom + a
  player that changes Y (jumping, ladders, elevators, etc.).
  Not just WoW-style — also applies to fixed-tilt side-scrollers
  if the camera Y is computed per-frame.

## Cinematic keyframes (boss intro / level transitions)

```ts
type CineKeyframe = {
  focus: THREE.Vector3;     // where the camera looks
  yaw: number;              // absolute yaw (radians)
  pitch: number;            // absolute pitch
  distance: number;         // meters from focus
  t: number;                // seconds into the cinematic
};

cine.play([
  { focus: V(0, 0, -45), yaw: Math.PI,    pitch: 0.45, distance: 22, t: 0    },
  { focus: V(0, 0,   0), yaw: this.camCtl.yaw, pitch: 0.6,  distance: 28, t: 1.4  },
]);
```

Interpolate linearly between keyframes; call `cinematic.update(dt)`
every frame; flag `running` when t exceeds last keyframe.

## PostFX chain (cheap, looks expensive)

```ts
import { EffectComposer, RenderPass, EffectPass,
         BloomEffect, ChromaticAberrationEffect,
         ScanlineEffect, ToneMappingEffect } from 'postprocessing';

composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new EffectPass(camera,
  new BloomEffect({ intensity: 0.4, luminanceThreshold: 0.5 }),
  new ChromaticAberrationEffect({ offset: new Vector2(0.0004, 0.0004) }),
  new ScanlineEffect({ density: 1.5, opacity: 0.15 }),
  new ToneMappingEffect({ mode: ToneMappingMode.ACES_FILMIC }),
));
```

Keep chromatic aberration under 0.001 — past that the screen becomes
unreadable on small displays. Bloom luminance threshold 0.5+ keeps the
look Tron-like (only highlights bloom).

## Render-on-demand (battery friend)

Pause the render loop on `visibilitychange === 'hidden'` and on
`document.hidden === true`. Most laptops idle CPU when the tab is
hidden, but Three.js doesn't pause automatically.

```ts
document.addEventListener('visibilitychange', () => {
  this.paused = document.hidden;
});
private tick(dt: number, t: number) {
  if (this.paused) return;
  // ...rest
}
```

## Procedural generation seeds

Re-seed `Math.random` with mulberry32 between level transitions so each
level gets a different layout but is reproducible given the seed:

```ts
function seedRandom(seed: number) {
  let s = seed >>> 0;
  Math.random = () => {
    s |= 0; s = (s + 0x6D2B79F5) | 0;
    let t = Math.imul(s ^ (s >>> 15), 1 | s);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

Call `seedRandom(level * 9173 + 42)` at the start of `startNextLevel()`.

## Performance checklist

- [ ] Particles use a **fixed pool** (no `new` in hot path)
- [ ] TrailRenderer uses `BufferGeometry.setDrawRange()` instead of
      creating new geometries each frame
- [ ] Boss + enemies update only if the player is within ~80m
      (`cullDistance = 80`)
- [ ] Audio uses Web Audio synth (no asset downloads)
- [ ] Three.js + postprocessing are in **manual chunks** (lazy)
- [ ] Terrain heightmap is a single function `terrainHeight(x, z)` —
      do NOT sample a stored 2D array; that's a cache-miss generator
- [ ] `requestAnimationFrame` is your loop, not `setInterval`
- [ ] First frame shows the tutorial overlay (don't render empty world)

## Build / verify loop

For any visual change, run BOTH:

```bash
npm run build && npm run preview    # production
```

Not `npm run dev` alone — Vite's HMR breaks with top-level await in
sync imports (see `web-3d-game-control-schemes` §5).

For headless / CI verification:

```bash
npm run build && (npm run preview -- --port 4174 &) && \
  curl -s http://127.0.0.1:4174/ | head -5
```

For visual verification inside an agent session, use the browser
tools (`browser_navigate`, `browser_vision`) on the preview URL.

**Exception — local backend bridging**: when the project talks to a
real backend (NestJS/Express/Rails), `vite preview` cannot reliably
proxy `/api/*` and `/ws` for you. Use `vite dev` and rely on
`server.proxy`. See pitfalls below.

## Pitfalls when bridging to a real backend

These come from porting this stack as a 3D frontend into a monorepo
that already has a 2D client and a NestJS/Express backend. The
NEONGRID-default "static leaderboard" doesn't trigger them; they
appear the moment you `fetch('/api/...')` from the browser.

### "Login fails with `TypeError: Failed to fetch` even though curl works"

**Root cause**: production backends pin `CORS_ORIGIN` to the prod
frontend host (`https://app.example.com` etc.) and ignore localhost.
The browser preflight fails because the dev origin
(`http://127.0.0.1:5174`) isn't allow-listed.

**Fix — proxy through Vite**: keep the browser same-origin, let Vite
tunnel `/api/*` and `/ws` to the backend. Add the proxy block in BOTH
`server` and `preview`:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    host: '127.0.0.1', port: 5174,
    proxy: {
      '/api': { target: 'http://127.0.0.1:3030', changeOrigin: false, secure: false },
      '/ws':  { target: 'ws://127.0.0.1:3030',  ws: true, changeOrigin: false, secure: false },
    },
  },
  preview: {
    host: '127.0.0.1', port: 4174,
    // Repeat explicitly — Vite 5 documents that preview.proxy
    // defaults to server.proxy, but a SPA-fallback index.html can
    // *mask* proxy misroutes (curl gets the HTML and the 200 looks
    // healthy while the API is unreachable). Explicit is safer.
    proxy: {
      '/api': { target: 'http://127.0.0.1:3030', changeOrigin: false, secure: false },
      '/ws':  { target: 'ws://127.0.0.1:3030',  ws: true, changeOrigin: false, secure: false },
    },
  },
});
```

Then point the client at same-origin:

```ts
const API_BASE = (import.meta as any).env?.VITE_API_BASE ?? '';   // '' = same-origin
```

…with WS resolving the same way when `API_BASE` is empty:

```ts
const wsBase = apiBase && apiBase.length > 0
  ? apiBase.replace(/^http/, 'ws')
  : `${location.protocol === 'https:' ? 'wss' : 'ws'}://${location.host}`;
```

**For production-style neutrality in `main`** (no localhost/port
hardcoded), use the opt-in env-var pattern: see
`references/dev-proxy-pattern.md`. That approach keeps the committed
`vite.config.ts` and `Backend.ts` clean; the developer copies
`.env.example` → `.env.local` and sets `CLIENTE_3D_DEV_PROXY=1` to
activate the proxy. Symmetry test recipe is included.

### "npm install only adds 2 packages, build fails on missing `vite`/`svelte`"

**Root cause**: `NODE_ENV=production` is set in the shell (Hermes TUI
inherits it from the session). npm respects `NODE_ENV` like the
matching config flag — `production` silently applies `--omit=dev` and
skips devDependencies (Vite, Svelte, TypeScript, esbuild rollup
plumbing). Exit code is 0; only "added 2 packages, audited 3 packages
in 1s" appears. The devDependencies never get installed.

**Fix**: prefix every npm command with `NODE_ENV=development`:

```bash
NODE_ENV=development npm install
NODE_ENV=development npm run build
NODE_ENV=development npm run dev
NODE_ENV=development npm run preview
```

Or `unset NODE_ENV` once per session (only affects the current shell;
new processes inherit again).

**Diagnostic one-liner** when "the install failed but didn't say so":

```bash
npm install && (ls node_modules/ | wc -l; npm ls --all 2>&1 | head -10)
# If you see ~30+ dirs in node_modules and npm ls shows every dev dep,
# you're fine. If you see <10 dirs and npm ls lists only "three" +
# "postprocessing", the NODE_ENV trap just bit you.
```

**Trap extension — subagents**: subagents dispatched with "run
npm install" inherit the parent's `NODE_ENV=production`. They will
silently install a sparse `node_modules/`, report success, and the
parent only discovers it when the next build fails on a missing
module. Either include `NODE_ENV=development npm install` literally
in the subagent brief, or fix the shell init for the session.

### "`document.cookie` is empty after login — did the cookie fail?"

**Root cause**: `Set-Cookie` from the backend carries
`HttpOnly; SameSite=Lax; Secure`. `HttpOnly` blocks JS from reading
it (`document.cookie === ''`), but the browser still sends it on
`fetch(..., { credentials: 'include' })`. This is correct behavior,
not a bug.

**Fix / verification**: hit an auth-required endpoint after login
(`/api/auth/me`, profile, etc.). If it returns the user data, the
cookie is working. Never trust `document.cookie` for auth check.

### "Backend route returns 404 even though controller exists and app.module registers it"

**Root cause**: the NestJS controller is in `app.module.ts` controllers
list and has `@Controller('api/foo')` but the container you ran
(`docker compose up`) was built before the controller file existed,
or the workspace's `dist/` is stale. Re-build and re-up the container:
`docker compose up -d --build backend`. Or run without Docker and let
`nest start:dev` watch.

Verify with `docker logs <container> --tail 50` — search for the
`[Nest] Mapped {...}` lines for the route you expected. If they're
absent, the running container doesn't have the controller compiled in.

### "The brief assumed endpoints that don't exist"

**Rule**: before writing or delegating any HTTP/WS bridge code, **read
the actual controllers** in the backend repo and verify:

- route path + method (e.g. `POST /api/auth/login`, not the
  guessed `/api/players/login`)
- request body shape (batch wrapper? snake_case or camelCase?)
- response envelope (plain object? `{ data: ... }`? array of items?)
- cookie attributes (`HttpOnly`? `Secure`? `SameSite`?)
- WS event shape (envelope like `{event, data}`? raw JSON?)

See `references/contract-discovery-before-bridging.md` for the
5-probe recipe (read app.module.ts, scan each controller, run a curl
probe with empty/valid body, compare headers). The classic
session-killer shape: brief assumes `GET /api/worlds/:id/chunks/:cx/:cz`
when the real route is `POST /api/spatial/chunks` with body
`{ requests: [{provider, key, centerLngLat, ...}] }`. Always verify.

### "Subagent dispatched for copy-literal work timed out without producing output"

**Rule**: do NOT delegate mechanical work (`cp`, `mkdir`,
straightforward file editing) to a subagent if you can do it inline.
Subagents loaded with a long brief about a complex plan will spend
their entire timeout budget READING before EDITING, even when the
user said "literal copy" and 90% of the work is mechanical. They
frequently return with `(no summary — status=timeout)` and zero
artifacts, leaving you with nothing to verify.

**Tripwires that mean "do not delegate"**:
- More than 80% of the work is `cp`, `mkdir`, or `write_file` with
  content already provided
- The brief contains "copia literal" or "do a literal copy"
- The decision tree is short (single verification probe at the end)

**What to delegate instead**: a fork with the backend contract
written into `Backend.ts`/`Players.ts`, the WS surface area mapped,
the CORS solution chosen, and the verification steps enumerated.
That's reasoning-heavy enough for delegation to add value.

### "`npm run dev` boots but `npm run build` fails on top-level await in `main.ts`"

Already documented above ("Dev server boots blank"). The reverse
case — `main.ts` has `await foo()` inside `(async () => {...})()`
which is fine for dev but fails for build — is the same root cause
and the same fix (move await into a dynamic import or use plain
.then).

## Cloudflare Pages

`_headers` (in `public/`) sets cache rules:

```
/assets/*
  Cache-Control: public, max-age=31536000, immutable
/favicon.svg
  Cache-Control: public, max-age=86400
```

`_redirects` enables SPA fallback (only needed if you ever add
client-side routing):

```
/*    /index.html   200
```

`wrangler.toml` binds the D1 database and any secrets:

```toml
name = "neongrid"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "neongrid-leaderboard"
database_id = "<from wrangler d1 create>"

[vars]
# (only non-secret config; secrets go via `wrangler pages secret put`)
```

One-shot deploy:

```bash
wrangler d1 execute neongrid-leaderboard --file=schema.sql
wrangler pages secret put SCORE_SECRET
npm run deploy
```

See `cloudflare` and `wrangler` skills for the full workflow.

### "Porting a 2D client's features to a 3D stack — feature port pattern"

**Symptom**: a 3D frontend lives in a monorepo with a mature 2D client
(PixiJS / Phaser / DOM-based). The 2D client has features the 3D
client doesn't: music streaming with playlist menu, world-data
dispatch (buildings, POIs, NPCs, items, roads, water, trees),
minimapa, combat (disc-whip, raycast, area-of-effect), items
inventory, skills cooldowns, menus. Each port requires deciding how
to draw the feature in 3D and how to keep the architecture
consistent with the 3D stack (Svelte 5 runes, Three.js r180, Vite
config, `window.__<feature>` globals for HUD coupling).

**Root cause**: there's no template for the 2D→3D port. The
developer re-discovers each time that:
- The 2D client's color palette (`KIND_RENDER` map) needs porting
  to a `Record<kind, { base, emissive }>` in the 3D World renderer.
- The 2D client's `useEffect` + `useRef` Canvas2D pattern needs
  porting to Svelte 5 runes (`$state` + `$effect` + `setInterval`).
- The 2D client's `useState` + `setState` for HUD state is replaced
  by Svelte runes plus the `window.__<feature>` global pattern
  (see `references/hud-from-gamestate-via-window-globals.md`).
- The 2D client's component-per-feature React tree is collapsed
  into one `App.svelte` with `$state` per concern (because adding
  React/Svelte components in a 3D Vite project is more overhead
  than the feature warrants).

**Fix — 5-step port template**:

1. **Read the 2D source first**. Open the 2D client's `*.jsx` /
   `*.js` for the feature. Identify the data shape (props, state,
   effects, side-effects), the colors / materials, and the
   interaction surface (clicks, keybinds, polls).

2. **Identify the 3D primitive**. Each 2D feature maps to a 3D
   rendering pattern:
   - `Canvas2D minimap` → `THREE.Mesh` radar OR a separate
     `HTMLCanvasElement` (cheaper for 2D drawings like circles +
     text). Most minimaps are 2D-rendered canvases overlaid on the
     HUD, NOT 3D meshes.
   - `PixiJS sprite` → `THREE.Sprite` with `CanvasTexture` (text
     labels, badges) OR `THREE.Mesh` with custom geometry
     (buildings, NPCs, items).
   - `React modal` → Svelte panel with `position: fixed` + glass
     `backdrop-filter: blur()` + a global state rune for
     `open/closed`.

3. **Port the data shape, not the component tree**. The 2D
   client's `KIND_RENDER = { building: '#ff6a00', water: '#355b77', ...}`
   becomes a `Record<kind, { base: number; emissive: number }>` in
   the 3D World. The 2D client's `MUSIC_TRACKS = [{ id, label, src }, ...]`
   becomes a `readonly MusicTrack[]` in the 3D audio module. Keep
   the same id semantics so backend contracts carry over unchanged.

4. **Wire via `window.__<feature>`**. Each ported feature gets
   a global in `Game.ts` setup (see the reference file). The HUD
   reads/writes it via `(window as any).__<feature>`. The Game
   never imports the HUD and vice versa.

5. **Verify in runtime**. After each port, test the feature
   end-to-end:
   - The 3D feature renders correctly (3D primitive matches the
     2D visual semantics).
   - The HUD's polling latency is acceptable (200ms for state
     that changes every frame, 500ms for state that changes
     every few seconds).
   - The cross-feature integration works (e.g. clicking a track
     in the playlist triggers the audio bus, which the
     minimapa doesn't care about — but the audio control in
     the HUD reflects the new `currentTrackId` within the next
     poll cycle).

Worked examples in 4world's 4world/cliente-3d (jun-26):

| 2D source | 3D target | Key file |
|---|---|---|
| `cliente/src/audio/musicTracks.js` | `src/audio/musicTracks.ts` | readonly `MusicTrack[]` (10 sonatina tracks) |
| Neongrid `src/game/Audio.ts` (2 tracks) | `src/audio/Audio.ts` (N tracks) | `AudioBus` class with N-layer round-robin crossfade |
| `cliente/src/ui/MiniMap.jsx` (Canvas2D) | `src/ui/App.svelte` `$effect` + `HTMLCanvasElement` | North-up 180x180 minimapa with buildings + remotes + player |
| `cliente/src/map/renderingConstants.js` `KIND_RENDER` | `src/game/World.ts` `KIND_COLOR` + `spawnFeature(opts)` dispatch | Per-kind geometries (box, sphere, cylinder, cone, plane) + `MeshStandardMaterial` per kind |
| `cliente/src/ui/OptionsMenu.jsx` | `src/ui/App.svelte` audio control | Mute button + volume slider + playlist toggle |

The pattern repeats: for each new feature (skills, items, menus,
combat), the question is "what 2D primitive does this map to?"
plus "what window global bridges the Game and the HUD?". The
template is small enough to apply in a single iteration per
feature.

### "Use the same logic, draw in 3D" — porting 2D parsing/analysis to 3D

When the 2D client already has logic for **interpreting** map
data (spatial grid, cell analysis, style seeds, composition),
the 3D client should reuse the same logic verbatim and only
swap the renderer. The temptation is to rewrite the logic for
the 3D stack — that's wasted effort and risks subtle divergence
between clients.

For the **scale alignment** that follows the port (two-scale
cellSize: logic=1m + visual=4m + the `BLOCK_SIZE = FOOTPRINT_METERS / 2`
rule, plus the placeGround performance trade-off from 30ms to
756ms and the optimization routes), see the
`cliente-3d-frontend-traps` skill Traps 82/83/84.
| `buildingStyleSeed.js` (`resolveBuildingFeatureStyleSeed`, `composeBuildingRunStyleSeed`) | Same hash function (`hashText` / FNV-1a), same inputs (feature.id → properties.id → osm_id → wikidata → name) | Deterministic — same input = same seed in both clients. Use it for `hashFootprint`, `hashTint`, anything per-feature visual. |
| `worldComposition.js` (`analyzeWorldCell`, `buildWorldComposition`, `resolveOverlay`) | Same logic, same zone names (`warm`, `cool`, `neutral`, `courtyard`, `meadow`, `grove`, etc.) | The zones drive terrain colors in the 3D canvas texture (`placeGround`). The 2D client paints them in PixiJS; the 3D client paints the equivalent colors in `CanvasTexture` cells. |
| `tilequery` HTTP response shape (`features[]` with `geometry.coordinates` Point + `properties.{type, kind, height, ...}`) | Same parser, same kind mapping | If the backend emits `Point` for both clients, you only get centroids. If the backend emits `Polygon`, the 3D client can use Polygon points for footprint corners; if `Point`, fall back to hash-based footprint from `buildingStyleSeed`. |

**Pattern**:

1. **Read the 2D module first**. Open `cliente/src/.../buildingStyleSeed.js`,
   `worldComposition.js`, etc. Read the whole file — the
   zones, the hash function, the kind classification rules,
   the function signatures. Most of these are pure functions
   (no DOM, no PixiJS), so the port is mechanical.

2. **Keep the same data shape**. `analyzeWorldCell` in the 2D
   client takes `{ cell, getKind }`; the 3D port takes the
   same. `resolveBuildingFeatureStyleSeed` in the 2D client
   returns a `uint32` from the feature id; the 3D port
   returns the same `uint32`. Same input → same output →
   same visual look across clients.

3. **Wrap with a 3D-specific renderer**. The 2D client calls
   `zoneColor(zone)` and paints with PixiJS Graphics. The
   3D port calls `zoneColor(zone, variantSeed)` and paints
   the same color into a `CanvasTexture` cell. Same color
   palette, different paint target.

4. **The 3D World dispatches by kind via `spawnFeature`**. Each
   cell gets a `kind` (from the tilequery feature at that lng/lat)
   or `'terrain'` (default). The `placeGround` loop calls
   `analyzeWorldCell({ cell, getKind })` and paints the zone's
   color in the canvas backing. Adjacent cells get warmer
   (`courtyard` near buildings) or cooler (`neutral` far
   from buildings) colors based on `buildingNeighbors` from
   the analysis. This produces a neighborhood-aware terrain
   without any external assets.

**Worked example (4world, jun-26)**:

- `cliente/src/map/buildingStyleSeed.js` (40 LOC) → `cliente-3d/src/game/buildingStyleSeed.ts`
  - Same `hashText` (FNV-1a), same `resolveBuildingFeatureStyleSeed`,
    same `composeBuildingRunStyleSeed`.
  - Added `hashFootprint(styleSeed, defaultFootprint)`:
    `w/d = default * (0.7 + ((styleSeed >> n) & 0xff) / 255 * 0.6)`
    — deterministic 70-130% of default per feature.
  - Added `hashTint(styleSeed, baseColor)` — small luminosity
    variance around the base color. **Not applied per-mesh in
    this iteration** because Three.js `MeshStandardMaterial` is
    shared per kind; per-mesh tint requires a custom
    `ShaderMaterial` with a `uTint` uniform. Document the
    constraint, don't work around it.

- `cliente/src/map/worldComposition.js` (264 LOC) →
  `cliente-3d/src/game/worldComposition.ts` (verbatim logic for
  `hashSeed`, `resolveOverlay`, `analyzeWorldCell`,
  `buildWorldComposition`). Same constants: `PATCH_SCALE = 6`,
  same `VEGETATION_KINDS`/`STRUCTURED_GROUND_KINDS` sets, same
  zone names. The 3D port adds `zoneColor(zone, variantSeed)`
  that maps zones to RGB strings consistent with the HUD
  palette (cyan / orange / magenta).

- `World.ts` integration:
  - `spawnBuilding(x, z, height, isPart, featureId)`:
    `styleSeed = resolveBuildingFeatureStyleSeed({ id: featureId, properties: null }) ?? 0`,
    `footprint = hashFootprint(styleSeed, FOOTPRINT_METERS)`,
    `mesh.scale.set(footprint.w, height, footprint.d)`.
    Save `mesh.userData.styleSeed` and `mesh.userData.footprint`
    for downstream consumers (colliders, minimap, hover).
  - `placeGround(center, radius)`:
    `cellKinds = featuresToCellKinds(features)`,
    `getKind = (cell) => cellKinds.get(key) ?? 'terrain'`,
    iterate the cells, `analyzeWorldCell({ cell, getKind })`,
    `ctx.fillStyle = zoneColor(analysis.zone, analysis.variantSeed)`.

- Verified runtime: 8 sampled buildings had footprints
  6.26×6.47 to 10.06×9.33 (within 70-130% range). The
  terrain visibly shifted from brown (`courtyard` near
  buildings) to blue-grey (`neutral`/`cool` far from
  buildings).

**Trade-off** the user accepts: the 2D client's tilequery
emits `Polygon` geometry (real corners); the 3D client's
tilequery emits `Point` geometry (centroid only — the
backend has `tilequery.geometry: "polygon"` in `properties`
but doesn't return the coords). Hash-based footprint is
the 3D fallback until the backend emits Polygon. Same
issue with the 2D client's `firstCollider` — it's based on
real polygon, but the 3D client approximates with
`FOOTPRINT_METERS × factor`.

### "HUD mini-canvas stretches across the screen because of a global `canvas { width: 100vw }` rule"

**Symptom**: a `<canvas>` element nested inside an HUD panel
(minimapapa, debug overlay, audio visualizer) renders as a
fullscreen element instead of staying inside its 180×180
container. The HUD panel itself looks fine (correct
positioning via `position: fixed; bottom/right: 14px`), but
the inner `<canvas>` is 1280×633 px (the viewport size)
and covers the entire screen, hiding the rest of the HUD.

**Root cause**: the project's `index.html` ships a global
`canvas` rule with `!important`:

```css
canvas {
  display: block;
  width: 100vw !important;
  height: 100vh !important;
}
```

The rule predates any HUD canvases (it assumed only the
Three.js renderer canvas). The selector `canvas` is more
general than `.minimap canvas`, and `!important` wins any
specificity battle. The mini-canvas inside the HUD gets
stretched to viewport size.

**Fix**: use `width: 100% !important; height: 100% !important`
on the HUD canvas, where `100%` means 100% of the parent
container (not the viewport). Pair with an explicit
`width` / `height` on the parent container (also `!important`
or otherwise specificity-safe):

```css
.minimap {
  bottom: 14px;
  right: 14px;
  width: 180px;
  height: 180px;
  padding: 0;
  border: 1px solid rgba(0, 240, 255, 0.4);
  border-radius: 4px;
  background: transparent;
  overflow: hidden;
}
.minimap canvas {
  display: block !important;
  width: 100% !important;
  height: 100% !important;
}
```

The `100%` (vs `180px`) is the key: it lets the canvas fill
whatever the container becomes (see "ResizeObserver
mini-canvas" below).

### "ResizeObserver mini-canvas — adapt drawing to the container"

**Symptom**: you want a HUD mini-canvas (minimapapa, debug
overlay, audio visualizer) to scale fluidly with its
container: change the CSS from 180×180 to 240×240 and the
contents should re-render at the new size without pixelation.

**Fix pattern**:

1. **In the draw loop, read `clientWidth`/`clientHeight`** of
   the canvas (not a hardcoded constant). If the size
   changed since last frame, update the backing store
   (`canvas.width = w; canvas.height = h`) so the browser
   doesn't stretch the previous backing.

   ```ts
   let lastW = 0, lastH = 0;
   const draw = () => {
     const w = canvas.clientWidth || 180;
     const h = canvas.clientHeight || 180;
     if (w !== lastW || h !== lastH) {
       canvas.width = w;
       canvas.height = h;
       lastW = w;
       lastH = h;
     }
     const SIZE = w;  // recompute every frame
     // ... draw with SIZE
   };
   ```

2. **Attach a `ResizeObserver` to the canvas** to fire
   `draw()` immediately when the container resizes
   (browser window resize, dev tools open, programmatic CSS
   change), instead of waiting for the next `setInterval`
   tick:

   ```ts
   const ro = new ResizeObserver(() => draw());
   ro.observe(canvas);
   return () => {
     clearInterval(id);
     ro.disconnect();
   };
   ```

3. **Scale all sizes proportionally to `SIZE`**, not a
   constant. The player radius, arrow length, dot radius,
   label font — everything that previously was a fixed
   pixel count becomes `max(min, SIZE * factor)`:

   ```ts
   const playerRadius = Math.max(3, SIZE * 0.028);
   const arrowLen    = Math.max(6, SIZE * 0.07);
   const remoteRadius = Math.max(2, SIZE * 0.018);
   const buildingBase = Math.max(3, SIZE * 0.04);
   ctx.font = `bold ${Math.max(7, SIZE * 0.05)}px monospace`;
   ```

4. **Edge case — `box-sizing: border-box`**: when the parent
   has a `border`, the inner canvas's `clientWidth` is the
   parent width **minus the border** (e.g. 180×180 container
   with 1px border = 178×178 inner). Either use
   `box-sizing: border-box` (clean) or accept the 2px
   difference (visible only if you set
   `canvas.width === container.width` exactly).

**Worked example (4world, jun-26)**: minimapa went from
fixed 180×180 (resize = pixelated stretch) to dynamic
100% of container (resize = clean re-render at 240×240,
120×120, etc.). All HUD elements (player, arrow, font,
dots, buildings) scaled proportionally. The
ResizeObserver fires a draw on every resize, so the
canvas never lags behind the container.

### "Disable pointer lock temporarily without removing the code"

**Symptom**: pointer lock is great for FPS-style mouselook
in gameplay, but it blocks all interaction with HUD
elements (buttons, sliders, dropdowns, modals). The user
clicks on the canvas once to lock the pointer, then tries
to click on the audio playlist or volume slider in the
HUD, and the click is captured by the canvas (or
prevented by the lock). The HUD becomes effectively
unreachable until ESC is pressed.

**Fix — comment out the pointer lock request, leave the
infrastructure in place**:

```ts
// MouseOrbitInput.ts
const onPointerDown = (e: PointerEvent) => {
  // ... existing setup ...
  if (e.button === 2) e.preventDefault();

  // Pointer lock DESACTIVADO temporalmente (jun-26).
  // The lock blocks HUD interactions (playlist, sliders, login).
  // To restore: uncomment below + remove the ESC hint HUD.
  //
  // if (!this.pointerLocked) {
  //   try { (this.canvas as any).requestPointerLock?.(); } catch {}
  // }
  this._onUserGesture();
};
```

**Don't actually delete the lock code** — the user wants
it back when a "fullscreen gameplay" mode is added later
(distinct from the current "menu + gameplay" mode). Leaving
the code commented with a clear "how to restore" comment
saves a re-derivation cycle.

**The `onUserGesture` callback stays live** — it fires on
every click regardless of lock state. This is what wakes
up the `AudioContext.resume()` for music. If you remove
the callback when commenting out the lock, audio
playback breaks for guest users (no successful login →
no `bootstrapMultiplayer` → no audio init).

### "Debug panel for tilequery responses — F1 toggle"

**Symptom**: you need to debug what shape the backend
tilequery response has (which property keys, which
types, which heights) before writing the renderer.
Iterating blind produces code that breaks when the
backend later adds a new field.

**Fix**: ship a debug panel from day one. Hotkey F1 to
toggle, R to refresh. Shows:

- `responseKeys` — top-level keys (`features`, `points`,
  `center`, `radius`, `ts`, `source`)
- `featureCount` — number of features
- `source` — which provider (`mapbox`, `overpass`,
  `nestjs-tilequery`)
- `sampleFeature` — first full feature (geometry +
  properties, prettified JSON)
- `allPropertyKeys` — union of all `properties.*` keys
  across features (sorted, deduped)
- `uniqueTypes` — unique `properties.type` values
  (`['building', 'building:part']` if the backend only
  emits buildings)
- `heightStats` — min/max/avg/median for
  `properties.height`
- `distanceStats` — same for
  `properties.tilequery.distance`

This is the diagnostic the user reads BEFORE implementing
the 3D port. Saves a "fix wrong assumption about the
response shape" round trip.

**Pattern**:

```svelte
<!-- App.svelte -->
<script lang="ts">
  let debugOpen = $state<boolean>(false);
  let debugText = $state<string>('F1 to load tilequery...');
  async function refreshDebug() {
    const { tilequery } = await import('../game/backendClient');
    const snap = (window as any).__minimap?.getSnapshot();
    const json = await tilequery(snap.player.lat, snap.player.lng, 150);
    debugText = JSON.stringify(summaryOf(json), null, 2);
  }
  $effect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.code === 'F1' && !e.repeat) {
        e.preventDefault();
        debugOpen = !debugOpen;
        if (debugOpen) refreshDebug();
      } else if (e.code === 'Escape' && debugOpen) {
        debugOpen = false;
      } else if (e.code === 'KeyR' && debugOpen && !e.repeat) {
        refreshDebug();
      }
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  });
</script>

{#if debugOpen}
  <div class="hud-panel debug-panel" role="dialog">
    <div class="debug-header">
      <span class="debug-title">DEBUG · tilequery</span>
      <button class="close-btn" onclick={() => debugOpen = false}>×</button>
    </div>
    <pre class="debug-body">{debugText}</pre>
  </div>
{/if}
```

CSS: glassmorphism panel with a debug-distinctive color
(yellow/amber) so it's visually obvious it's not a
game-state panel. Width 480px, left-center, `z-index: 9999`.

**Critical fetch path — use the project's `tilequery`
wrapper, not `fetch` directly**. The wrapper handles
`VITE_API_BASE`, the Vite proxy, the cookie credentials,
and the timeouts. Calling `fetch(url)` from the HUD
often fails with `TypeError: Failed to fetch` because
the URL hits the wrong origin (the project talks to
`/api/...` via proxy, not `http://localhost:3030`).

## Sub-skill references

- **`references/audio-music-streaming-and-sfx.md`** — Web Audio synth
  + music streaming recipe (AudioBus class, 2-layer round-robin
  crossfade, SFX oscillators for pickup/slash/hit/boom/hurt/warning/
  victory, `onUserGesture` callback for browser autoplay policy,
  HUD mute/volume control). Use when adding audio to any Three.js
  + Vite project. Pairs with the `mouseOrbit.onUserGesture` pattern
  documented in `references/controls-and-camera.md`.
- **`references/responsive-mobile-hud.md`** — Svelte 5 + Three.js
  mobile HUD: tap-to-expand minimap, compact corner cards,
  controls banner at the bottom, `body.mobile` class pattern
  for QA with `?mobile=1`, the Svelte `:global()` scope trap,
  and the separate `controlsMessage` vs `lore` channels
- **`references/threejs-wireframe-and-outline.md`** — the LineSegments /
  EdgesGeometry / BackSide outline pattern, including the
  wireframe-offset fix
- **`references/hud-from-gamestate-via-window-globals.md`** — exposing
  game state to the Svelte HUD via per-feature `window.__<feature>`
  globals (audio bus, mouse lock, minimapa snapshot, etc.). Use
  when the HUD and the game are decoupled and a reactive store
  would drag the Svelte runtime into pure-TS game code.
- **`references/threejs-part-pivot-rigging.md`** — rotating humanoid
  parts (arms, legs) from the shoulder/hip, not the geometric center,
  via a per-part `pivotY` offset. Pairs with the outline-mesh pattern
  from `references/threejs-wireframe-and-outline.md`.
- **`references/controls-and-camera.md`** — input → camera mapping,
  A/D vs Q/E convention, cinematic lock
- **`references/combat-and-projectiles.md`** — collision radii,
  invincibility frames, boss scaling
- **`references/climbable-geometry.md`** — the topCollider vs firstCollider
  pattern for letting the hero stand on top of decorative props
- **`references/build-and-deploy.md`** — Vite config, _headers /
  _redirects, wrangler.toml, deploy.sh, the "Outdated Optimize Dep"
  cache-fix workflow, favicon setup
- **`references/shop-and-overlay-locks.md`** — semantic gating for
  level-transition overlays (shop, level-up screen): `input.frozen`
  + `shopDamagePaused` so the world keeps running while the player
  decides what to upgrade, plus the `restartRun()` pattern that
  replaces `location.reload()` after game-over
- **`references/boss-spawn-cinematic.md`** — two-phase boss reveal
  cinematic: 360° spiral around the player (wind-up) + dolly-in
  close-up of the boss (reveal) with full keyframe table and tuning
  bounds
- **`references/end-of-encounter-cinematics.md`** — the OTHER end of
  the encounter: kill cinematic (slow-zoom out from the boss body)
  and death cinematic (smash-zoom into the hero's body). Covers the
  `onDone`-chaining pattern that replaces `setTimeout` racing the
  cinematic, the `.clone()` requirement for the body position, the
  timing constraint (cine duration ≥ existing `setTimeout` delay to
  avoid races), and the **gameOver-vs-cine-update trap**: when a
  cinematic has an `onDone` callback that needs to fire during
  gameOver, the loop must update the cine independently of the
  `!gameOver` gate on the main `tick()` or the callback never runs
- **`references/dev-proxy-pattern.md`** — opt-in dev-proxy pattern
  for POCs that need to talk to a backend locked to a production
  CORS_ORIGIN. Committed files stay neutral (`vite.config.ts`
  without proxy, `Backend.ts` reads `VITE_API_BASE` only); debug
  config lives in `.env.local` (gitignored) gated on
  `CLIENTE_3D_DEV_PROXY=1`. Includes the symmetry-test recipe
  (verify proxy is on WITH `.env.local`, off WITHOUT) and why
  editing `backend/.env` is the wrong answer.
- **`references/client-for-existing-backend.md`** — same engine,
  different purpose: using this stack as a 3D frontend for a
  backend you already have (REST + WS + cookie auth). Bootstrap
  by literal-copy of render-only files, fork only files that need
  the backend contract. Includes `Backend.ts`/`Chunks.ts`/`Players.ts`
  templates, frustum-culling policy, the cross-stack profile-routing
  workaround for `delegate_task`, a per-step verification matrix,
  and a tail section "Bootstrap gotchas" capturing NODE_ENV
  install traps, the "invented endpoints" rule with the actual
  NestJS contracts, CORS collision with a coexisting 2D client,
  and the subagent-mechanical-work timeout pattern. Use when
  porting this stack into a monorepo with a 2D client already in
  production.
- **`references/contract-discovery-before-bridging.md`** — 5-probe
  recipe for verifying a production backend's auth/data/WS
  contracts (route paths, body shapes, cookie flags, CORS origin)
  BEFORE writing or delegating the bridge code. Reads controllers,
  cross-checks `app.module.ts` registrations, validates live with
  curl. Prevents briefs that assume invented endpoints like
  `GET /api/worlds/:id/chunks/:cx/:cz` when the real one is
  `POST /api/spatial/chunks` with a batch body.
- **`references/cliente-3d-4world-workspace-conventions.md`** —
  workspace-specific conventions for the `4rgs/4world/cliente-3d`
  project: NODE_ENV=production trap (npm omits devDependencies),
  backend port 3030, the `worldId` requirement in `joinWorld`,
  the "kill zombie vite" recipe, and the **subagent timeout
  pattern**: 3 subagents in a row died by timeout doing the 95%
  of work before commit. Includes a copy-paste-ready brief
  template for surgical re-delegation. Use when working on the
  4world 3D client specifically; the rest of this skill applies
  to any 3D web game.

## Game-loop reset pattern (auto-restart after game-over)

When the user wants a "play again" flow that doesn't require a manual
button click, schedule an automatic restart inside the score-submission
handler. Persist the things that should survive runs (best score,
achievements, avatar, heroName); reset everything else to defaults.

```ts
submitWithName(name: string) {
  // ... persist score, upload replay, fetch leaderboard ...
  // Auto-restart IMMEDIATELY (synchronous restartRun in the same call).
  // A "restart in 3 seconds" countdown reads as a bug — the player
  // just confirmed their name, they want the new run NOW.
  this.restartRun();
}

restartRun() {
  this.gameOver = false;
  this.hud.setGameOver(false);
  // Keep: save.bestScore, save.run.avatar, save.run.heroName
  // Reset: upgrades, currentLevel, achievements, score, fragments
  this.save.run.upgrades = { ...DEFAULT_UPGRADES };
  this.hero.hp = this.hero.maxHp;
  this.level = 1;
  this.score = 0;
  // Regenerate world with seedRandom(this.level * 9173 + 42)
  // Replay: this.replay.clear(); this.replayStartedAt = performance.now();
  // Cinematic reveal of the new level
}
```

Don't reload the page (`location.reload()`) — that nukes the Svelte HUD
state, the AudioContext, and any in-memory caches. A clean `restartRun()`
preserves everything and feels instant.

## More pitfalls (jun-26, 4world/cliente-3d)

These came up when porting the same engine to a 3D frontend
(`4rgs/4world/cliente-3d`) that lives in a monorepo with a 2D
client and a NestJS backend. Several map directly to standalone
browser 3D games; a few are 4world-specific but the patterns transfer.

### "Outlines stay behind when an animated body part rotates"

**Symptom**: in a humanoid avatar built from `BoxGeometry` parts
(torso, head, arms, legs), each part is given a BackSide outline
sibling mesh scaled by 1.06–1.08. When you add a walk cycle
(`armL.rotation.x = sin(t*10) * 0.6`), the arm rotates but the
outline mesh — added as a sibling in the same root group — stays
frozen in place. The result: the arm swings with no outline, while
a static "ghost" outline hovers at the original rest position.

**Root cause**: the outline mesh and the body mesh are siblings in
the same `THREE.Group`, not children of a shared part group. A
`rotation.x` on `armL` only affects the arm mesh; the outline
inherits from the root group, which doesn't rotate per-part.

**Fix**: each `addPart()` returns a `THREE.Group` containing the body
mesh + outline mesh. Animations apply to the part group, so the
outline follows automatically.

```ts
// Before (broken)
const mesh = new THREE.Mesh(geo, bodyMat);
group.add(mesh);
const outline = new THREE.Mesh(geo, outlineMat);
outline.scale.setScalar(1.06);
group.add(outline);   // outline is sibling of mesh; rotations on mesh don't carry outline

// After (fixed) — outline is child of the part group
const part = new THREE.Group();
part.position.set(x, y, z);
const mesh = new THREE.Mesh(geo, bodyMat);
part.add(mesh);
const outline = new THREE.Mesh(geo, outlineMat);
outline.scale.setScalar(1.06);
part.add(outline);
group.add(part);
// Now part.rotation.x rotates BOTH mesh and outline together.
```

If you have many parts and a single model is used for both the
local hero and remote players, factor the construction into a
`buildPlayerModel({ color, seamColor })` factory that returns
`{ group, bodyMat, refs: { torso, head, armL, armR, legL, legR } }`.
The local hero and each remote player get their own instance
(material cloned per call so each can be tinted independently
from `state.appearance.color`).

### "Player jumps through the roof of a building"

**Symptom**: low buildings with `BoxGeometry` extrusions. Hero
walks into the side, jumps, the jump arcs over the top of the
building, and the hero lands on the other side. Visually the
player flies through the roof at the apex of the jump.

**Root cause**: the top-landing check requires
`newY >= top - 0.3 && vy <= 0` — i.e. "falling AND already in the
landing range". While the player is still rising (`vy > 0`),
`newY` might already be over the top, but the check fails because
`vy > 0`. The integrate loop applies `newY` directly to
`position.y`, the player passes through the roof, and the next
frame the player is on the other side.

**Fix**: drop the `vy <= 0` condition. `onTop` should only check
that `newY` is in the top range; the apply block clamps to `top`
if `newY > top` and resets `vy` to 0 (or the velocity reflected).
This is the pattern Neongrid uses for jump-and-stand-on-top
chaining: a single jump arcs up, hits the cap's top, plants
mid-arc, and the player can chain another jump.

```ts
// Before (broken)
const onTop = newY >= top - 0.3 && this.vy <= 0;

// After (fixed)
const onTop = newY >= top - 0.3;
// In the apply block: if (newY > standOnY + 0.001) clamp down.
```

The same shape applies to boss-fight jump pads, jump-through
platforms, and any "stand on top of arbitrary geometry" feature.

### "Sticky ground leaves the player floating in mid-air forever"

**Symptom**: hero stands on a building. Walks to the edge, takes
one step off the footprint. The hero stays at the building's
`top` Y level, hovering in the air, with `vy` accelerating to
-45 m/s in the background, until 2-3 seconds later a single
frame's `newY` finally crosses the sticky threshold and the
hero falls like a stone.

**Root cause**: a sticky-ground feature designed to handle "step
just off the edge of a collider without falling into the void"
keeps applying frame after frame as long as
`oldY >= lastStandOnY - 0.4`. But `oldY` is reset to `lastStandOnY`
every frame (because the sticky also resets `position.y` to
`lastStandOnY`), so the threshold is satisfied forever. The
hero stays at `lastStandOnY` while gravity secretly accumulates
`vy`.

**Fix**: when sticky is applied, do NOT persist `lastStandOnY`
to the same value. Set it to `null` so the next frame starts
fresh:

```ts
if (stickyApplied) {
  this.lastStandOnY = null;  // ← critical: don't persist
} else {
  this.lastStandOnY = standOnY > 0 ? standOnY : null;
}
```

This makes the sticky truly transient: it bridges one frame after
the player leaves the footprint, and from the next frame on
the integrate loop finds no collider under the new XZ, falls
through to `standOnY = groundY = 0`, and the hero drops
naturally under gravity. Pair this with the `vy <= 0` removal
from the top-landing check above and the player gets smooth
walking off edges + free falling.

A secondary fix: when sticky IS applied, the `apply` block should
NOT mark `onGround = true` or reset `jumpsLeft`. Otherwise, pressing
Space while sticky-ing off the edge of a building would count as
a ground jump (because the sticky planted the player at the
building's top with `onGround = true`).

### "Salto se resetea manteniendo Space presionado"

**Symptom**: hero jumps once, lands a moment later. If the user
keeps Space pressed (or the OS autorepeat kicks in), the
`onGround` flag combined with new keydown edges fires a fresh
ground-jump every ~500 ms. Visually the player bounces in
place. Identical shape: jump, land, jump, land.

**Root cause**: the `keydown` handler was guarded with
`if (this.keys.has(k)) return;` only. That blocks the second
consecutive keydown the browser fires for the same held key
under normal conditions — but if the `keys` Set ever clears
(window blur, login modal stealing focus, programmatic injection
via `pressKey`), the next keydown from the OS autorepeat bypasses
the guard and sets a new edge. Every 500 ms a new edge → new
ground-jump.

**Fix**: filter on the standard `e.repeat` flag from the browser
itself, which is `true` for any keydown that is part of an OS
autorepeat sequence. This works without depending on internal
state and survives `keys.clear()` cycles:

```ts
target.addEventListener('keydown', (e: KeyboardEvent) => {
  if (e.repeat) return;          // ← add this FIRST
  if (this.keys.has(k)) return;  // existing guard, keep
  this.keys.add(k);
  if (k === 'Space') this._jumpEdge = true;
  // ...
});
```

Also: if the project uses `press(code)` for programmatic
input-injection (unit tests, AI bots), `press()` bypasses the
`e.repeat` guard. Either make `press()` set a
`_programmaticEdge` flag and consume it explicitly per call, or
document that `press()` is meant to fire one edge per call and
callers should debounce.

### "Backend snap resets the player to y=0 mid-jump"

**Symptom**: hero is mid-jump at y=2.5. The server sends a
`stateDelta` or `playerUpdate` (soft snap ≤ 0.5m or hard snap
≥ 2m). On the next frame the player teleports to y=0, the
ground-jump triggers from `onGround = true`, and a fresh jump
starts — visually breaking the original jump's arc.

**Root cause**: `setAuthoritativePosition(lat, lng)` for the
soft/hard snap paths did
`this.group.position.set(w.x, 0, w.z); this.yPos = 0;` —
overwriting the current Y to 0 every time the server
"confirmed" the position. The next integrate frame starts from
y=0 with `vy ≈ 12.5` and `onGround = false` (set by the apply
block, not the snap), but the ground-jump from section 8 sees
`onGround = true` (because the apply block's branch for `if
(vy <= 0 && newY >= standOnY - 0.3 && !stickyApplied)` flipped it
to `true` on a different frame). The ground-jump triggers,
`vy = 12.5`, fresh jump starts.

**Fix**: rule of thumb is "preserve the higher Y between client
and server". The server 2D doesn't send Y, so preserve client Y
when the client is mid-jump.

```ts
const currentY = this.group.position.y;
const syncedY = currentY;  // server doesn't send Y; preserve

if (distMeters <= SOFT_THRESHOLD_M) {
  this.group.position.set(w.x, syncedY, w.z);
  this.yPos = syncedY;
  return;
}
if (distMeters >= HARD_THRESHOLD_M) {
  // Hard snap teleports XZ to a different place. If the new XZ
  // doesn't have a collider under it, preserving Y leaves the
  // hero floating in mid-air at y > 0. Reset Y to 0 on hard snap
  // (ground is the only logical position) UNLESS the player was
  // already near the ground (currentY < 0.5).
  const hardY = currentY < 0.5 ? currentY : 0;
  this.group.position.set(w.x, hardY, w.z);
  this.yPos = hardY;
  this.vy = 0;
  this.onGround = hardY === 0;
  this.lastStandOnY = null;  // also clear sticky
  return;
}
```

The dual-write invariant: `group.position.x/z` is the visual
source of truth (animated each frame by the integrate loop);
`localLngLat` is the wire-protocol source of truth (sent to the
server). `setAuthoritativePosition` must keep both consistent.
If the snap also lerps (`startSnap` path), the lerp block
(line 1 of `update()`) must only write `x` and `z`; the integrate
loop in step 7 decides `y`.

**Critical sub-trap — hard snap resets Y to 0 (don't preserve on
lateral teleport)**: the rule "preserve Y" applies ONLY to the
soft snap (`dist <= 0.5m`, same XZ). For the hard snap
(`dist >= 2m`, XZ differs from the current XZ), preserving
`currentY` leaves the hero floating in the air at `y > 0` over
an XZ that has no collider under it. The hard snap is a lateral
teleport, not a position confirmation; reset Y to 0 (assume
ground-level arrival), `vy = 0`, `onGround = (hardY === 0)`,
`lastStandOnY = null` (clear sticky). Exception: if
`currentY < 0.5` (the player was already almost on the
ground), preserve Y to avoid pops when the server sends a
minor correction that classifies as hard.

```ts
if (distMeters >= HARD_THRESHOLD_M) {
  const hardY = currentY < 0.5 ? currentY : 0;
  this.group.position.set(w.x, hardY, w.z);
  this.yPos = hardY;
  this.vy = 0;
  this.onGround = hardY === 0;
  this.lastStandOnY = null;
  return;
}
```

The three snap types each have their own Y policy. Confusing
them produces "salto interrumpido" (soft snap that resets Y)
or "flotar en el aire" (hard snap that preserves Y):

| Snap type | dist | XZ change | Y policy |
|---|---|---|---|
| Soft (confirm) | ≤ 0.5m | none | **preserve Y** (server confirms our pos) |
| Hard (teleport) | ≥ 2m | lateral | **reset to 0** (lateral teleport) |
| Mid (lerp) | 0.5 < d < 2m | small | **lerp XZ, Y from integrate** (next frame decides) |

### "Pointer lock + mouselook: rotate camera without holding a button"

**Symptom**: in a 3D world with mouse-driven camera orbit, the
user wants the standard FPS behavior: click on the canvas once,
the cursor disappears, moving the mouse rotates the camera
without needing to keep the button held. ESC releases the
cursor. A small HUD hint shows "ESC to release cursor" while
locked.

**Fix pattern**:

1. **Request lock on first pointerdown** on the canvas:

   ```ts
   const onPointerDown = (e: PointerEvent) => {
     // ... existing drag setup ...
     if (!this.pointerLocked) {
       try { (this.canvas as any).requestPointerLock?.(); } catch {}
     }
     // user gesture: also wake up the AudioContext if not yet
     this._onUserGesture();
   };
   ```

2. **Track lock state via `pointerlockchange`** on `document`,
   not `window`:

   ```ts
   const onPointerLockChange = () => {
     this.pointerLocked = (document as any).pointerLockElement === this.canvas;
     this._onPointerLockChange(this.pointerLocked);
   };
   document.addEventListener('pointerlockchange', onPointerLockChange);
   document.addEventListener('pointerlockerror', (e) => {
     console.warn('requestPointerLock rejected', e);
   });
   ```

3. **Use `movementX/Y` (delta) when locked, `clientX/Y`
   (absolute) when not:

   ```ts
   if (this.pointerLocked) {
     dx = e.movementX; dy = e.movementY;
   } else {
     dx = e.clientX - this.lastX;
     dy = e.clientY - this.lastY;
     this.lastX = e.clientX; this.lastY = e.clientY;
   }
   ```

4. **Skip the `dragging` check when locked** (so movement
   without a held button still rotates the camera):

   ```ts
   if (!this.dragging && !this.pointerLocked) return;
   ```

5. **HUD hint visible only while locked** (Svelte 5 + glass
   panel at top-center):

   ```svelte
   <div class="hud-panel pointer-lock-hint"
        class:visible={mouseLocked}
        aria-label="Pointer lock hint">
     <span class="esc-key">ESC</span>
     <span class="hint-text">presiona para recuperar el cursor</span>
   </div>
   ```

   ```ts
   let mouseLocked = $state<boolean>(false);
   $effect(() => {
     const id = setInterval(() => {
       const v = !!(window as any).__mouseLocked;
       if (v !== mouseLocked) mouseLocked = v;
     }, 100);
     return () => clearInterval(id);
   });
   ```

   Polling at 100ms is fine for a boolean flag; if the user
   wants push-based updates, expose a callback that mutates
   `$state` from outside Svelte.

**Critical sub-trap — pointer lock ≡ button held (FPS-style)**:
the mouselook is a different interaction model from drag-to-orbit.
When the pointer is locked, the user is in FPS mode: they expect
to move the mouse without holding a button and have the camera
rotate. This means:

1. **Skip the `dragging` check when locked**. Otherwise the
   `onPointerMove` filter (`if (!this.dragging) return;`) blocks
   all mouse movement until the user clicks.

   ```ts
   if (!this.dragging && !this.pointerLocked) return;
   ```

2. **When locked, prefer `movementX/Y` (delta) over `clientX/Y`
   (absolute)**: the browser reports delta movement while
   locked (the cursor is "frozen" but mouse movement is
   delivered as delta). `clientX/Y` is anchored at the lock
   position and doesn't update.

   ```ts
   if (this.pointerLocked) {
     dx = e.movementX; dy = e.movementY;
   } else {
     dx = e.clientX - this.lastX;
     dy = e.clientY - this.lastY;
     this.lastX = e.clientX; this.lastY = e.clientY;
   }
   ```

3. **Once locked, ESC is the only way out**: the browser hides
   the cursor; there is no visual "drag handle" the user can
   release. Pair with the HUD hint so the user knows how to
   recover the cursor.

Caveat: `requestPointerLock` requires the page to be in focus
and (per spec) historically required HTTPS. Modern browsers
allow it on `http://localhost` and `http://127.0.0.1`. If the
project is served from a LAN IP without HTTPS, lock is denied
and the project falls back to standard click-drag with no
visible feedback. Surface a `pointerlockerror` log so the
developer sees this in the console.

### "Audio + pointer lock share the same user gesture"

`AudioContext.resume()` (and music `playTrack`) require a
"user gesture" (click, keypress) to start. The browser rejects
silent autoplay. The cleanest implementation in a Vite +
Three.js + Svelte 5 stack: **wake up the audio inside the same
`pointerdown` handler that requests the pointer lock** — the
click on the canvas is the first user gesture either way, and
you don't need to wait for the actual lock to succeed.

Pattern: expose an `onUserGesture` callback on the
`MouseOrbitInput` (or the equivalent input class). Game.ts
subscribes once and runs the audio init inside it:

```ts
// MouseOrbitInput.ts
private _onUserGesture: () => void = () => {};
onUserGesture(cb: () => void) { this._onUserGesture = cb; }
const onPointerDown = (e: PointerEvent) => {
  // ... existing setup ...
  if (!this.pointerLocked) (this.canvas as any).requestPointerLock?.();
  this._onUserGesture();   // ← always fire (idempotent on the audio side)
};

// Game.ts
this.mouseOrbit.onUserGesture(() => {
  if (this.audioBus.isReady()) return;   // idempotent: no-op on subsequent gestures
  this.audioBus.ensure();
  this.audioBus.preloadMusic(2).then(() =>
    this.audioBus.playTrack(DEFAULT_TRACK_ID, 0.8)
  );
});
```

**Don't tie audio to `bootstrapMultiplayer` only** — that only
runs after a successful login. If the user is playing in "guest
mode" (login failed or not yet attempted), the bootstrap is
skipped and the audio never wakes up. A click on the canvas is
a user gesture the browser counts; the audio init there is
robust against any login state.

### "Vite zombies serve stale bundle after hot-reload"

**Recipe (the 4world iteration loop)**: after a refactor
(e.g. changing the default export signature of
`buildPlayerModel`), the dev server's HMR sometimes does NOT
update the bundle served on the existing port. The browser
shows the old behavior; new commits aren't visible.

1. `lsof -nP -iTCP:5180 -sTCP:LISTEN` — find the dev server PID.
2. `kill -9 <pid>` — kill zombies.
3. `cd cliente-3d && NODE_ENV=development npm run dev -- --port 5180 > /tmp/vite-5180.log 2>&1 &`
4. `sleep 5 && curl -sI http://127.0.0.1:5180/ | head -1` — verify HTTP 200.
5. Test the change in a fresh browser tab.

If `kill -9` doesn't free the port (some macOS versions hold it
in TIME_WAIT), use `pkill -f "vite --port 5180"` or
`lsof -ti:5180 | xargs kill -9`.

This pattern is reused in EVERY iteration of a 4world task;
without it, 50% of "the fix didn't work" reports are zombie
bundles, not bugs. (Related root cause: top-level `await` in
sync-imported modules — see "Dev server boots blank" above.)

### "Hero walks through buildings"

**Symptom**: the 2D client (PixiJS) had a `firstCollider` /
`topCollider` AABB system; the 3D client (Three.js) had
nothing. Hero walks into a building and clips through it. If
you add a Three.js `mesh` with `userData.kind = 'building'`, the
visual exists but no collision body is registered.

**Fix**: factor the building footprint into a `BuildingCollider`
interface (`{ x, z, halfX, halfZ, height }`), expose it from
`World.getColliders()`, and consume it in the hero's integrate
loop. A single AABB slide (X then Z) handles the wall
collision; a top-landing check (see "Player jumps through the
roof" above) handles stand-on-top.

```ts
// World.ts
getColliders(): ReadonlyArray<BuildingCollider> {
  return this.buildings.map(b => {
    const w = lngLatToWorldMeters(b.lngLat);
    return {
      x: w.x, z: w.z,
      halfX: FOOTPRINT_METERS / 2, halfZ: FOOTPRINT_METERS / 2,
      height: b.height,
    };
  });
}

// Hero.ts
setColliders(colliders: ReadonlyArray<BuildingCollider>) {
  this.colliders = colliders;
}

// Game.ts
this.hero.setColliders(this.world.getColliders());

// in integrateMovementWithColliders:
//   1. compute newX/newZ from velocity
//   2. for each collider, if AABB overlap in X (with oldZ), slide:
//      newX = c.x - sign(velX) * (halfX + radius + epsilon)
//   3. re-check overlap in Z (with finalX) for slide
//   4. apply finalX, finalZ
//   5. gravity: newY = oldY + vy*dt
//   6. for each collider under our XZ, if (newY >= top - 0.3),
//      standOnY = top
//   7. apply newY, with sticky-ground cleanup
```

The single most important detail: in step 6, the `newY >= top
- 0.3` check does NOT require `vy <= 0`. See "Player jumps
through the roof" above for why.

## Sub-skill references
