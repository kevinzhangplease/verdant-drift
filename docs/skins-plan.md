# Phase 6 — Skins

A skin changes **only** how the game looks and sounds. Nothing about movement, hitboxes,
damage, spawn tables, timing or scoring may differ between skins — a run played in one skin
must be mechanically identical to the same seed played in the other.

Two skins: **space** (the original look) and **picnic** (semi-3D, pastel, soft-shadowed,
kid-friendly cartoon food). In picnic the ship is a ketchup bottle, its shots are ketchup
blobs, homing missiles are mayonnaise, enemies are foods echoing the silhouette they replace
(a big round boss becomes a watermelon; small fast ones become hot dogs), and the background
is a picnic blanket scrolling down.

Decisions taken: **12 per-sector picnic variants** · **all 24 bosses reskinned, staged in
batches behind a fallback** · **a fully separate score** (its own progressions and
instrumentation) · **switchable at any time, including mid-run**.

---

## Handover — read this first

**Branch:** `claude/arcade-shooter-improvements-yaqgbx`.
**Everything is in one file:** `index.html` (~5.2k lines, single IIFE, no build step, no deps).

To resume: read this document, then `git log -12` — the commit messages carry the *reasoning*,
not just the changes, and several record why an earlier approach failed. That is the cheapest
way to avoid repeating a dead end.

### Three invariants that must not regress

1. **Determinism.** Anything reachable from `render()` must use `frnd`/`fchance`, never
   `rnd`/`rint`/`pick`/`rand` — the render path runs at whatever rate the device paints, so a
   seeded draw there feeds frame rate into the simulation. Five such leaks have been fixed
   (`noiseBuf`, `spark`, `spawnDebris`, `drawStars`, plus the cannon-clock bug below).
   **Verify determinism as a property, not a sample:** run each configuration at least three
   times and require all runs to agree. A single match between two runs is not evidence — that
   mistake produced two commits' worth of false claims that had to be retracted.
2. **Legibility.** The player band and the enemy band never overlap; each skin declares which
   hues they are. `bandOf()` exists to make this testable. Picnic deliberately inverts space's
   convention (warm player, cool enemy) and is legible for the same reason.

   Two things S4b established. **Picnic holds a hard cool band** — all five enemy keys stay in
   170–290° in every one of its twelve sectors, because its warm player band leaves no room to
   vary enemy hue the way space does; sector identity lives in the cloth instead. And **space
   does not hold the rule it states**: its `missile` is amber, inside its own enemy band, and in
   sectors 5 and 8 even `bolt` lands within 3° of the boss/elaser hue. That is pre-existing and
   deliberately untouched — but do not treat space as the reference implementation of the
   invariant, because it isn't one.
3. **Hitboxes are sacred.** A skin applies food to the *existing* geometry — same radius, same
   part count, same part positions. Never adjust gameplay shape to suit the art.

### Why staging is safe

The art layer (`ART`, `art()`) falls back to the space draw function whenever a skin has no
override. An entity with no picnic art renders its space silhouette in picnic colours, so the
game stays fully playable no matter how much of S5/S6 is finished. Batches can land
incrementally without ever leaving the build broken.

### Test harness

Not committed — rebuild it in a few lines. Append a `window.__dbg = { … }` hook inside the IIFE
of a *copy* of `index.html`, exposing whatever internals the test needs, then drive it with
Playwright (`/opt/node22/lib/node_modules/playwright`; Chromium is pre-installed — do not run
`playwright install`). Drive the simulation by calling `update(dt)` directly with
`state='paused'` so the rAF loop cannot interleave and mutate arrays mid-iteration.

Three things that repeatedly paid off, and one repeated trap:

- **Call-site diffing.** Wrap the seeded RNG to record `(frame, call-site)` per draw, run twice,
  and report the first index where they differ. This is what finally located the run-to-run
  drift, after outcome comparison and draw-count comparison had both missed it — the RNG
  *sequence* was identical and only the *frame* differed.
- **Test the mechanic, not just the code.** Several bosses are only killable if you engage their
  mechanic; a harness that ignores it reports a false failure. Simulate a player who does the
  right thing *and* one who does not, and check both outcomes are the intended ones.
- **Trap: the harness lies more often than the game does.** Calling `update()` directly freezes
  `elapsed`, so `beatTick` never fires; `hurtPlayer`/`useBomb` gate on `state==='playing'`, so a
  paused test never increments them; a dead test pilot freezes the sim. Each of these produced a
  confident, wrong bug report before the scaffolding was suspected. Confirm a finding against the
  unmodified build before believing it.

### An open aesthetic question

The picnic palette and the ketchup bottle are guesses at "pastel, kid-friendly". Everything from
S5 onward inherits them. If they are not right, redirect **before** ~45 more draw functions are
authored against them — that is by far the cheapest moment.

---

## The two things that make this non-trivial

**1. It is not a palette swap — it is a second art layer.** A ketchup bottle is not a recoloured
arrow; a watermelon is not a recoloured hexagon. There are **65 bespoke draw functions** (21
`dw*` enemy, 24 boss `draw`, 20 `drawPart`) plus the ship, drones, missiles, bullets, pickups,
particles, floats, lasers, shocks, tells, background and 12 hazards.

**2. The legibility rule has to be generalised, not broken.** The README calls it
non-negotiable: player fire cool/cyan, enemy fire hot magenta→orange. It was also enforced
*structurally* — `applyPalette` copied only the eight enemy/background keys, so the six player
keys (`hull, bolt, missile, drone, crystal, pickup`) were physically unreachable by a palette
swap. Picnic deliberately **inverts** it: warm ketchup shots, cool enemy fire.

The rule is therefore now **band-based**: *the player band and the enemy band never overlap, and
each skin declares which hues they are.* That preserves the actual purpose — knowing instantly
whose projectile is whose — while allowing the inversion. It is an explicit, tested invariant,
not a convention, or the rule quietly rots. (Phase 5 spent a whole commit fixing eight sites
where a player colour had leaked into enemy art; that must not regress.)

---

## Status

| Stage | State |
|---|---|
| S1 foundations | shipped `c269bf2` — band-based palette, `SKIN`, `bandOf`, 100 literals routed |
| S2 selection | shipped `73cede5` — title + pause rows, persistence, live switching |
| S3 player art | shipped `e87b368` — art dispatch + fallback, sprite cache, semi-3D helpers, ketchup bottle |
| S4a blanket | shipped `952eb2e` — gingham background, crumbs/ants |
| S0 determinism | shipped `e11bf77` — root cause found and fixed; see below |
| S4b spreads & hazards | shipped `36c81bd` — 12 picnic sectors, 10 hazards, hard cool band |
| S5 enemies | shipped `33e7dbc` — all 19 types as food, cool-band roster |
| S6 bosses | shipped `23e9772`, `9ef46b8` — all 24 as food |
| S7 | **next** — audio: sound bank split and the 12-sector picnic score |

### S0 — the determinism hunt (resolved)

**Retraction, on the record.** S1 and S2 each claimed the same seed produced byte-identical
simulation across skins. That was a single-sample comparison and it was wrong: the same seed run
three times in the *same* skin gave three different scores (5034 / 5467 / 5737 on a 40s run). The
guarantee this whole feature rests on was, at that point, unproven.

The cause was not a seeded-RNG leak at all, which is why two earlier hunts missed it. Call-site
diffing showed the RNG *sequence* was identical between runs — only the *frame* each draw landed
on differed (a kill at frame 198 in one run, 194 in the next). `resetRun` reset the player's
relics, overdrive, bulwark, morph, drone form and grace timers, but never `P.fireT` or
`P.missileT`. A new run therefore inherited whatever phase of the fire cycle the previous run
ended on: the first shot landed up to a full fire-interval early or late, and every kill after it
shifted with it. Same seed, same RNG sequence, different timing.

Fixed in `e11bf77` (along with `drawStars` recycling stars via seeded `rnd()` on the render path).
Verified as a property: three runs each in space and picnic, all six identical at
`5034,6,10,3` — so the cross-skin claim can now be made honestly, and Daily Drift is genuinely
reproducible for the first time.

---

## Architecture

### S1. Skin registry and band-based colour

`SKIN = { space:{…}, picnic:{…} }`, each declaring: a base palette for **all 14 `PAL` keys**
(not just the eight), 12 sector variants, an art table, a background renderer, and a sound bank.

`applySkin(key)` sets the base palette and rebuilds derived state; `applyPalette(sectorIdx)` then
layers the sector variant on top. The existing `PAL` key **names** are kept, so the ~214 existing
`rgba(PAL.x, …)` call sites needed no edit — only their values change.

**Prerequisite, done first:** route the ~101 hardcoded colour literals through the palette. Hull
fills like `'rgba(8,30,38,.96)'` and `'#fff'` were scattered through the draw functions and would
render space-coloured under picnic. Unglamorous, and it had to come first or every later commit
would fight it.

**The invariant, made testable:** `bandOf(colour)` returns `player | enemy | neutral`, and a
verification pass asserts no enemy-owned draw call uses a player-band colour or vice versa, in
both skins.

### S2. Art dispatch with fallback

```
ART[skin].enemy[type] · ART[skin].boss[key] · ART[skin].ship / bullet / pickup / …
```

`drawEnemies` already dispatched through `TYPES[e.type].draw` and `drawBoss` through
`def.draw`/`def.drawPart`/`def.drawExtra` — both single choke points. Resolve an override first,
fall back to the existing function when a skin has no art for that entity yet. That fallback is
what makes "staged in batches" possible.

### S3. Semi-3D, cheaply — pre-rendered sprites

"Semi-3D with soft shadows" on canvas 2D means radial gradients for volume, a consistent top-left
light, a rim highlight and an offset soft shadow. **Doing that per entity per frame will not hold
60fps on mobile** — 30+ entities each building gradients every frame is exactly the cost the
adaptive quality scaler exists to shed.

So: render each picnic entity **once** into an offscreen canvas at first use, cache by
`type+size`, and blit thereafter. Rotation and flash tint stay per-frame (cheap); the shading is
baked. Shared helpers — `blob()`, `softShadow()`, `gloss()` — so 65 draw functions share one look
instead of each inventing it. Respect `qLevel`: at low quality drop the shadow pass and the rim
highlight, never the silhouette.

One trap worth remembering: several draw paths leave the composite mode set to `'lighter'`, so
`blitSprite` must force `source-over` itself or blobs blow out to white.

### S4. Live switching

Because switching is allowed mid-run, no skin state may be baked at boot. `applySkin` rebuilds
`buildNebula`/`buildStars`/the blanket pattern, re-applies the current sector palette, swaps the
sound bank without stopping the music, and clears the sprite cache. It is safe to call from the
title screen, the pause menu, and mid-fight.

Determinism is safe here because cosmetic randomness draws from `_fxRng`, a separate stream from
the seeded simulation. Skin switching therefore cannot desync a run — but verification must prove
it, since that is the whole promise.

### S5. Audio

`SOUNDBANK = { space:{…}, picnic:{…} }` exposing the same method names (`shot, boom, pickup,
chime, hurt, bombFx, alarm, fanfare, tick`) plus a 12-entry `MODES` table, so the ~40 existing
call sites are untouched. Picnic: softer waveforms (sine/triangle over square/saw), major-mode
progressions, squelches and pops and bottle-squeezes in place of zaps and booms.

**Fix while here:** `MODES` had only 6 entries for 12 sectors, so sectors 7–12 threw every 25ms
and had no music at all. Phase 5 patched it by wrapping (`i % MODES.length`); picnic should ship
12 from the start, and space's six should be extended to a real twelve rather than left wrapping.

### S6. Chrome

The HUD, veils and buttons are DOM/CSS driven by custom properties (`--jade`, `--ink`, …). A skin
sets those on `:root`, so menus, boss bar, HUD chips and clear screens follow the theme without
restructuring markup. Skin selection is an `optionRow` on the title screen — that helper already
takes `(host,label,items,current,onPick)` — plus a pause-menu entry, persisted in the save
alongside difficulty and speed with a defensive default.

---

## Picnic content

**Player** — ketchup bottle (squeeze-deform on fire), ketchup-blob shots, mayonnaise homing bits,
mustard drones, a gingham reticle, crumbs for the thruster trail.

**Enemies** *(shipped — this list is the one that landed, not the original)*. The first roster
here was cherry tomato, prawn, lobster, lemon slice and so on: written before picnic committed
to a hard cool band, and every one of those foods sits in the *player's* band. Recolouring them
blue would fight the subject, so the roster was rechosen around food that is naturally cool:

drifter → blueberry · weaver → blue corn chip · lancer → purple carrot · turret → blueberry
muffin · sentry → red cabbage (its arcs were always concentric) · hauler → cool-box · scavver →
bluebottle · cinder → iced star biscuit · ember → loose blueberry · blinker → jelly cube ·
prism → blue cheese (the rind is the shield) · shoal → grape · angler → mussel with a pearl
lure · wisp → thistledown · polyp → artichoke · chorister → red onion (rings, for a thing that
sings in rings) · warp → moth · pulsar → blueberry macaron · mote → poppy seed.

**Rule for S6:** pick the food to fit the band, never the band to fit the food.

**Bosses** *(shipped — all 24)*. warden → blueberry jam jar (its lid is the shield plate) ·
bloom → flower cake · scrapjaw → nutcracker · gravedigger → blackberry shedding drupelets ·
forgetwin → salt-and-pepper mills · solaris → a cut fig · sovereign → sugar bowl and cubes ·
architect → chequered sandwich cake · lantern → punched tin lantern · leviathan → caterpillar ·
gauntlet → scone / star biscuit / sandwich by form · quietstar → the grand jelly mould ·
harrow → cake fork · cairn → layer cake whose tiers you knock off · bract → platter of cucumber
slices · medusa → jellyfish-mould jelly · cantor → sandwich triangles · basilica → cupcakes on a
stand · loom → lattice pie between two spools · reflection → your own bottle in the enemy band ·
valve → mangosteen · heart → beetroot · keeper → preserving jar with six clamps · aperture → the
basket lid opening.

Each keeps its exact hitbox, part count and part positions — food is applied to the existing
geometry, never the other way round.

**Two traps this stage hit, worth carrying into S7 and beyond.** First: three live readouts
(medusa's bell, aperture's iris, heart's beat scale) were rewritten from memory instead of read
from source and came out subtly wrong — always diff a reskinned expression against the original.
Second: the space boss table leaks player-band keys into enemy draws (`PAL.crystal` ×26,
`PAL.bolt` ×17). Copying a space draw as the basis for picnic art copies the leak with it; every
picnic boss has been cleaned, the space table deliberately has not.

**World** — gingham blanket scrolling on `scrollY`, crumbs and ants as the star layers, cutlery
and paper plates as debris, the ground plane becomes grass. The 12 hazards become spilled drinks,
wasp swarms, sun glare, dropped jelly, and so on.

**Pickups & FX** — crystals are sprinkles; mods are condiment sachets; shields are a cloche; bombs
are a shaken fizzy-drink can. Explosions are splats, not fireballs; the boss death sequence
becomes a cake collapse; the graze aura is a sugar shimmer.

---

## Staging

Each stage is independently shippable and leaves the game playable.

1. **Foundations** — route the hardcoded literals through the palette; add `SKIN`, `applySkin`,
   band-based `PAL`, the `bandOf` invariant + test. No visual change yet.
2. **Selection & chrome** — title row, pause-menu entry, persistence, CSS custom properties.
   Picnic is selectable and recolours the world, still using space silhouettes via fallback.
3. **Sprite cache + shared semi-3D helpers**, then the **player**, bullets, pickups and particles.
4. **Background & hazards** — blanket, crumbs, grass, the 12 sector variants.
5. **Common enemies** — all of them. Note the count is **19**, not the 21 this plan said:
   `TYPES` holds 13 campaign-one types and 6 campaign-two. `dwBunker` and `dwPylon` are boss
   *parts*, which is where the extra two came from.
6. **Bosses** — 24 in four batches of six, campaign one first (it is the more distinctive art).
7. **Audio** — sound bank split, picnic SFX, then the 12-sector picnic score.

---

## Verification

**Test the property, not a sample.** Every determinism assertion must run each configuration at
least three times and require all of them to agree — a single match between two runs is not
evidence.

1. **Mechanical identity — the core promise.** Run the same seed in both skins with identical
   scripted input; assert identical score, enemy count, bullet count, positions and lives at every
   sampled frame. Any divergence means a skin touched gameplay.
2. **Mid-run switching** — switch skin every few seconds during a fight; assert no console errors,
   no leaked state, and that the run signature still matches an unswitched control run.
3. **Legibility bands** — for each skin, assert every player-owned draw uses only player-band
   colours and every enemy-owned draw only enemy-band, via `bandOf`. Screenshot a dense fight in
   each skin to confirm by eye.
4. **Fallback coverage** — with picnic selected, spawn every enemy type and all 24 bosses and
   assert each renders (its own art or the fallback) with no errors.
5. **Performance** — check at `qLevel 0` that shadows drop and the silhouette survives.

   **The skin-to-skin ratio is not currently answerable in this environment.** S4b tried: with
   medians of five interleaved samples, untouched sectors still swung between −11% and +19%
   across runs, so inter-run drift in the headless software rasteriser exceeds the effect being
   measured. Large-area fills — which is exactly what picnic's cloth is — are the workload a
   software rasteriser punishes most and a real GPU handles best, so a bad number here is weak
   evidence of a real problem. Profile *layers against each other in the same run* instead
   (stub `ART.picnic.background` / `.stars` / a hazard and difference the timings); that is
   stable and it is what found the two real costs S4b fixed. Settle the ratio on a device.
6. **Persistence** — skin survives reload; a save written before this feature loads with a
   defensive default.
7. **Regression** — the Phase 5 boss sweep still passes in both skins.
