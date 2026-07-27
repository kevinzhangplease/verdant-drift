# Verdant Drift — Twelve Sectors

A single-file, mobile-first arcade vertical scrolling shooter. Twelve sectors, each
with its own palette, enemy roster, audio mode, and hazard — bound together by one rule:

> **Legibility is not negotiable.** Every sector changes palette, but two things never
> change: your fire is always cyan-white (cool), and enemy fire is always in the
> hot magenta-to-orange band. Beauty is allowed to do anything except make the
> screen harder to read.

## Play

Open `index.html` in any modern browser, or play the deployed build. Touch, mouse,
keyboard (WASD/arrows, Shift/X focus, Space/Z bomb), or **gamepad** all work. It's a
PWA — installable and playable offline.

## Features

- **Twelve sectors**, 24 bosses, adaptive per-sector procedural music (Web Audio).
- **Chain + graze scoring** — kills build a ×8 chain; *grazing* enemy fire (flying
  close without being hit) pushes the multiplier all the way to ×16.
- **Focus / bomb / upgrades / relics**, checkpoints, three difficulties, five speeds.
- **Daily Drift** — a seeded run everyone shares that day; every run is reproducible.
- **Saved progress** — high scores, furthest sector, and all preferences persist.
- **Comfort & legibility settings** — screen-shake / flash / photosensitivity-safe
  mode, a colorblind cue, a true-hitbox marker, haptics, and adaptive quality.

## Tech

Pure HTML/CSS/JS on a single `<canvas>` — no build step, no runtime dependencies.
Fixed-timestep simulation for frame-rate-independent bullet patterns, object pooling
and swap-remove in the hot loops, a scaled bloom buffer, and adaptive quality scaling
that sheds detail to hold 60fps on low-end devices.

Files: `index.html` (the game), `manifest.webmanifest`, `sw.js` (offline cache),
`icon.svg`.

## Deploy

Static site — any host works. Connect this repo to Vercel and every push deploys live.
