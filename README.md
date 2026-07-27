# Verdant Drift — Six Sectors

A single-file, mobile-first arcade vertical scrolling shooter. Six sectors, each with
its own palette, enemy roster, audio mode, and hazard — bound together by one rule:

> **Legibility is not negotiable.** Every sector changes palette, but two things never
> change: your fire is always cyan-white (cool), and enemy fire is always in the
> hot magenta-to-orange band. Beauty is allowed to do anything except make the
> screen harder to read.

## Play

Open `index.html` in any modern browser, or play the deployed build. Touch or mouse
to steer; on-screen pads for **Bomb** and **Focus**. Desktop keyboard is supported.

## Tech

Pure HTML/CSS/JS on a single `<canvas>` — no build step, no dependencies. Procedural
audio via the Web Audio API, a scaled-down bloom buffer for glow, and a parallax
starfield/nebula background.

## Deploy

Static site — any host works. This build is deployed on Vercel.
