# Ripping Packs — Game Design Document

A single-file browser idler about cracking trading-card booster packs, sorting the dump pile, and selling completed sets on a live market. Built as one HTML file with vanilla JS + CSS. All gameplay state persists in `localStorage`.

---

## 1. Pitch

> *Open packs. Watch cards rain into a pile of colored sand. Click the sand (or type the right keys) to sort. Spot the rare and foil grains and crit your sort. Craft full sets. List them on a live drift market with a 10-slot cap. Buy sorting machines and hire "Homies" to grind for you while you chase the next legendary.*

The fantasy is the **pack-opening loop**, dramatized: pulls drop physical-feeling grains into a sand pile, sorting reads as a tactile act (click into a grain mix and time it right for a crit), and selling is a tiny live market with drifting prices and a finite shelf.

---

## 2. Core Loop

```
BUY → RIP → SORT → CRAFT → LIST → SELL → BUY ...
```

1. **Buy** packs, sealed boxes, or cases from the shop.
2. **Rip** a pack — either by clicking the booster or by typing the on-screen prompt word.
3. **Sort** the unsorted pile by clicking the sand canvas. Each click sorts 3–7 cards (a *crit* near a non-standard grain triples it).
4. **Craft** Standard / Rare / Foil Standard sets when you've collected one of every card in that rarity.
5. **List** crafted sets (and unique mythic / non-standard foil pulls) on the marketplace, max 10 active listings.
6. **Sell** — each listing resolves on its own cooldown for the current market price.
7. **Reinvest** profits into more packs, Sorters ($1k each, permanent), or Homies ($20 + 1 box, temporary).

---

## 3. Cards & Rarities

Each booster pack contains **10 cards**: 9 standards + 1 slot that upgrades through a rarity chain (Standard → Rare → Epic → Legendary → Mythic). Every card slot independently rolls a foil chance and an additional upgrade chance.

### Set sizes (unique card IDs per rarity)

| Rarity     | Set size | Grid layout      |
|------------|----------|------------------|
| Standard   | 36       | 2 rows × 18      |
| Rare       | 18       | 1 row            |
| Epic       | 9        | 1 row            |
| Legendary  | 6        | 1 row            |
| Mythic     | 3        | 1 row            |

Foil variants exist for every rarity (separate card identities).

### Inventory model — *priced* vs *bulk*

- **Bulk (sortable):** Standard, Rare, and Foil Standard cards. These never have per-card market prices. They flow through the unsorted pile → sand-sort → per-id buffer, and only liquidate by crafting a full Set into a market item.
- **Priced (per-id market):** Epic, Legendary, Mythic, and foil variants of Rare/Epic/Legendary/Mythic. These go straight into a per-id inventory and can be listed individually by clicking their cell in the collection grid.

---

## 4. Sand Pile + Sorting

Newly pulled bulk cards drop into the sand canvas as **colored grains** — green for standard, blue for rare, rainbow-shimmer for foil standard. Each grain represents one card stacked physically in the pile.

### Click-to-sort

Clicking the canvas removes 3–7 grains and moves those cards into your collection (assigning random IDs against `SET_SIZES[rarity]`). The collection cell for each landed card flashes.

### Click-position crit

The old random-crit roll was replaced with a positional system: **if any non-standard grain (rare or foil-standard) is within a 2-cell radius of your click, the sort crits and sorts ×3**. Pure standard fields stay non-crit. This rewards reading the pile and clicking into the colored specks.

### Unsorted cap

Max 2000 cards unsorted. The "Supply" / pack-opening flow refuses to rip when the cap is hit; the typing mini-game silently drops keys.

### Dump

A `DUMP PILE` button under the sand cashes out the entire unsorted pile at 50% of `SELL_PRICE × FOIL_MULTIPLIER`. Used to clear a stuck pile or trade volume for a small cash injection.

---

## 5. Crafting Sets

When `bulkInvById[key]` holds at least 1 of every card ID in a rarity, a glowing **CRAFT … SET** button appears in the vault's tracking row for that rarity. Crafting:

- Consumes 1 of every card ID (per-rarity set completion drops to 0)
- Decrements `state.stock[rarity]` (or `state.stockFoil.standard` for foil track)
- Adds 1 to the matching Set item key (`standardSet` / `rareSet` / `foilStandardSet`)

### Set baselines (market items)

| Set                | Baseline $ | Notes                          |
|--------------------|-----------:|--------------------------------|
| Standard Set       | 30         | 36 unique standards            |
| Rare Set           | 200        | 18 unique rares                |
| Foil Standard Set  | 120        | 36 unique foil standards       |

Set items drift on the same volatile market as other priced inventory (mean-reverting fundamental + tick volatility, configurable in the Game Design panel).

---

## 6. Marketplace

A 10-slot, single-tier market. Each slot holds one listing — either a priced single (mythic, non-standard foil) or a crafted Set.

### Listing tiles

Listings render as mini-cards matching the collection cell silhouette. They show:
- Rarity-tinted background + border + glow
- Glyph and card ID (or `SET`)
- Gold askPrice strip along the bottom
- Gold countdown badge in the top-right

Set listings get a stacked-cards illusion via offset shadows. Foil listings shimmer. Empty slots render as dashed `+` placeholders.

### Pricing & cooldown

Every listing posts at **1× current market price**. Resolve cooldown is uniform in `[LISTING_COOLDOWN_MIN, LISTING_COOLDOWN_MAX]` (default 20–40s). Every listing eventually sells.

Click a listing tile to **cancel and return** the card/set to inventory.

### Cap behavior

When 10 slots are full:
- The collection-grid click-to-list path shows "All 10 listing slots full" and refuses
- The vault's `LIST` buttons swap to `SLOTS FULL` text and disable

---

## 7. The Sand Pile — Visualization

`spawnSandFromPulls()` creates falling grains for each newly opened bulk card, with staggered Y offsets so they read as a sand-fall stream. Grains physics-fall down a column until they hit the existing pile height. `stepSandAnim` runs on `requestAnimationFrame` whenever there's motion or a foil-standard grain is in the pile (which keeps its color shimmer animating).

Foil standard grains rotate through HSL hues every ~24ms — they visibly sparkle in the pile.

---

## 8. Typing Mini-Game

A typing zone under the pack opener shows a 3–7 letter word from a curated thematic list: `rip, tear, pack, open, more, another, cards, rare, epic, legendary, mythic, foil, sealed, booster, holo, shiny, gem, hit, chase, pull, bust, crack, slab, mint, vault, set, collect, score, pile, drop, stack, deck, binder, graded, case, box, spread, lucky, hot, fresh, shred, sort, homie, sorter, go`.

- The next-letter slot pulses white with a gold underline
- Each correct keystroke calls `ripOne()` — if a pack actually opens, the letter flashes (gold → white scale-up → gold) and advances
- Wrong keys are ignored silently
- If supply is empty or the unsorted pile is full, the press is dropped (no advance, no pack consumed) — the game self-throttles
- Word completes → 280ms beat → new word picked

The handler is suppressed when focus is on an input/textarea or any modifier is held, so the Game Design panel and other text inputs still work normally.

---

## 9. Helpers — Homies & Sorters

### Hire Homie — temporary auto-ripper

- **Cost:** $20 + 1 sealed box
- **Effect:** spawns an animated 🧢 sprite next to the pack opener with a live progress bar; auto-rips one pack every 3s from its personal 36-pack pool
- **Lifetime:** until the box is empty (~108s total, faster if you bought multiple homies — they tile to the right)
- Homie rips trigger the same pull / sand-pile / particle animations as manual rips, but don't drain `state.packSupply`

### Buy Sorter — permanent auto-sorter

- **Cost:** $1,000 each, max **3** per save
- **Behavior:** Each tick (250ms) the sorter checks its internal interval. When elapsed, it pulls one card from the unsorted pile (weighted by key counts) into its own per-key buffer and removes the grain from the sand canvas.
- **Buffer cap:** 20 cards at Lv 1 (+10 per level). When full, the sorter pauses until you collect.
- **Manual collect:** Click the gold-glowing **COLLECT** button on the sorter card to flush its buffer into the collection — all cards land at once with `trackBulkCard` flash animations + set-progress updates.

### Sorter upgrades

| Level | Interval | Buffer | Upgrade cost |
|------:|---------:|-------:|-------------:|
| 1     | 2000 ms  | 20     | —            |
| 2     | 1000 ms  | 30     | $500         |
| 3     |  667 ms  | 40     | $2,000       |
| 4     |  500 ms  | 50     | $4,500       |
| 5     |  400 ms  | 60     | $8,000       |

Floor on interval is 250ms; `SORTER_LEVEL_MAX = 5`. Cost grows as `SORTER_UPGRADE_BASE × level²`.

---

## 10. Boxes — Sealed Inventory

Boxes are an **owned resource** (`state.boxesOwned`), distinct from loose packs (`state.packSupply`).

- Buying a Box: `+1 boxesOwned` (no longer +36 loose packs)
- Buying a Case: `+6 boxesOwned`
- Clicking the pack auto-cracks a sealed box into 36 loose packs when supply is dry
- Hiring a Homie consumes 1 box and gives the homie a personal 36-pack pool (independent of `packSupply`)
- Supply readout: `Supply: N · 📦 M`

This lets the player choose between immediate manual ripping (crack a box → 36 in supply) and delegated ripping (give a box to a homie).

---

## 11. Economy & Default Pricing

Pack/box/case costs:
- **Pack:** $3
- **Box:** $100 (36 packs worth, ~7% discount)
- **Case:** $500 (6 boxes worth, ~23% discount)

Bulk dump prices (`SELL_PRICE`, ×`FOIL_MULTIPLIER` for foils):
- Standard: $0.01
- Rare: $0.05
- Foil Standard: $0.10
- Foil Rare bulk: not applicable (foil rares are per-id priced)

Market baselines (drift around these, configurable):
- Epic: $5, Legendary: $25, Mythic: $100
- Foil Rare: $1.5, Foil Epic: $50, Foil Legendary: $250, Foil Mythic: $1000
- Standard Set: $30, Rare Set: $200, Foil Standard Set: $120

---

## 12. Market Simulation

A simple mean-reverting random walk per market key:

- Each `MARKET_TICK_MS` (7s default), every priced card ID has its `state.market[key][id]` price adjusted toward its `state.marketFund[key][id]` fundamental.
- Tick volatility: `TICK_VOLATILITY` (default 0.05) — proportional shock per tick
- Mean reversion strength: `TICK_REVERT` (default 0.07)
- Floor / ceiling clamped to `PRICE_FLOOR_RATIO × baseline` / `PRICE_CEIL_RATIO × baseline`
- Fundamentals drift slowly via `TICK_FUND_DRIFT`

Price drift is visible: list a card now or wait for a high tick. Average inventory value is shown in the vault rows ("~$2.85 avg") when you hold multiple of a priced card.

---

## 13. UI Layout (3-column at desktop)

| Column         | Contains                                                          |
|----------------|-------------------------------------------------------------------|
| **Shop** (left)| Buy buttons (Pack/Box/Case), Hire Homie / Buy Sorter, set-progress meters, the pack opener, typing zone, hint line, sand canvas, dump button, sorter cards |
| **Vault** (mid)| Header (coins, packs opened, foils), marketplace listings panel, collection grid (cells per rarity row, foil row beneath each), then the three bulk tracking rows (Rare / Foil Standard / Standard) with set craft + list buttons |
| **Footer**     | One-line flavor copy                                              |

Mobile collapses to single column. Cells shrink to 28px and the standard grid stays 18-wide.

---

## 14. State Schema (high level)

```
state = {
  coins, packsOpened, packSupply, boxesOwned,
  opened: { [rarity]: lifetime pull count, non-foil },
  openedFoil: { [rarity]: lifetime foil pull count },
  stock:     { [rarity]: current non-foil inventory total },
  stockFoil: { [rarity]: current foil inventory total },
  owned:     { [rarity]: array of unique IDs collected },
  ownedFoil: { [rarity]: array of unique foil IDs collected },
  unsorted:  { standard, rare, foilStandard },
  bulkInvById: { standard: {id:count}, rare: {id:count}, foilStandard: {id:count} },
  invById: { [marketKey]: {id:count} },   // priced inventory only
  market:  { [marketKey]: {id:price} },   // live prices
  marketFund: { [marketKey]: {id:fund} }, // fundamentals
  listings: [{ uid, rarity, cardId, foil, askPrice, resolveAt, willResolve }],
  pile: [[grainKey,...], ...],            // 2D column-stack of sand grains
  homies: [{ uid, packsRemaining, lastRipAt }],
  sorters: [{ uid, level, buffer:{standard,rare,foilStandard}, lastSortAt }],
  // plus: counters, set-completion flags, version stamps for migration
}
```

A versioned migration runs on load (`MARKET_VERSION`, `SET_VERSION`) to backfill new fields and reshape removed ones — e.g. legacy per-id foil-standard inventory gets folded into `bulkInvById.foilStandard`; retired `state.friends` is dropped.

---

## 15. Configurability — Game Design Panel

Press the **DESIGN** button to open a modal exposing every tunable as a numeric input, grouped: Economy, Pack Odds, Sorting, Marketplace, Staff, Market Baselines. Values persist in `localStorage` per-browser. **Apply & Reload** restarts the game with the new constants. A **RESET** button at the bottom wipes the save entirely.

This makes the whole game a live design sandbox — every drop-rate, every baseline, every cooldown, every cost is hot-swappable for tuning.

---

## 16. Out-of-Scope (current build) / Roadmap Hooks

- **Multiplayer / trades:** none.
- **Cross-series:** only one card series; expansion would require a series picker + per-series state namespacing.
- **Cosmetics:** no custom card art or alternate sleeve themes yet.
- **Achievements / quests:** none.
- **Mobile-first polish:** layout adapts but interaction is desktop-pointed (the canvas crit is generous enough for touch, but the typing mini-game is desktop-only).
- **Sorter upgrades beyond Lv 5:** capped for now; future levels could specialize (e.g. foil-only sorter, rare-priority sorter).
- **Homie variants:** currently one type. Future: rare-finder homies, fast-rip homies, sorting homies.

---

## 17. Pacing — One Player's Day

| Time-in     | Coins range | Activity                                                           |
|------------:|-------------|--------------------------------------------------------------------|
| 0–5 min     | $0–$50      | Crack starter box. Click-rip + type-rip first 36 packs. Hand-sort. |
| 5–20 min    | $50–$500    | Hire first homie. Buy boxes. Spot-list a first mythic / foil rare. |
| 20–60 min   | $500–$3k    | Buy first sorter. Start crafting Standard Sets reliably.           |
| 1–3 hrs     | $3k–$20k    | Multiple sorters, Lv 2–3. Working on Foil Standard Set.            |
| 3+ hrs      | $20k+       | Max sorter levels. Rare Set completions. Chasing foil mythic.      |

---

## 18. Look & Feel

Dark plum / indigo panel palette, gold accents for cash and CTAs. Pixel-bold display font (`Press Start 2P` if loaded, else system mono). Color codes by rarity:

- Standard: green `#5ad06a`
- Rare: blue `#5aa8ff`
- Epic: purple `#c060ff`
- Legendary: gold `#ffd84d`
- Mythic: orange `#ff7a3d`
- Foils: rainbow shimmer gradient

CSS animations carry the game's tactility: card-pop flash for new pulls, sand-pile bobbing for foil grains, gold pulse for available craft/collect buttons, scale-up for typed letters that hit, stacked-card shadow on Set listings.

---

*End of design doc.*
