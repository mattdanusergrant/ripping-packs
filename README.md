# Ripping Packs

A small incremental clicker about buying and ripping booster packs from a fictional cryptid TCG.

**Play:** https://mattdanusergrant.github.io/ripping-packs/

## Run locally

Open `index.html` in any modern browser. No build step, no dependencies.

## Mechanics

- Buy packs (1), boxes (36, save 11%), or cases (216, save 17%) — bulk locks in today's price
- Each pack: 3 commons + 1 uncommon + 1 rare slot (80% uncommon / 18% rare / 2% mythic)
- Foils — 5% per card, 3× sell value, rainbow shimmer
- Sell by rarity (foils protected by default) or dump the whole collection
- Save: localStorage per browser

## Status

MVP v0.6 — works end to end. Next candidates: per-rarity meta effects, prestige loop, a second set, sound design.
