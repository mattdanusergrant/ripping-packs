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
3. **Sort** the unsorted pile by clicking the sand canvas — every click detonates the nearest grain plus a 4-orthogonal AoE, and any non-standard caught in the blast chain-explodes (rares trigger another 4-ortho blast, foils trigger an 8-cell "+" reaching 2 cells along each axis).
4. **Craft** Standard / Rare / Foil Standard sets when you've collected one of every card in that rarity.
5. **List** crafted sets (and unique mythic / non-standard foil pulls) on the marketplace, max 10 active listings.
6. **Sell** — each listing resolves on its own cooldown for the current market price.
7. **Reinvest** profits into more packs, Sorters ($500 each, permanent), or Homies ($20 + 1 box, temporary).

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

### Click-to-detonate

Sorting is purely positional — there's no "pull random cards from the top." Every sort is a click that detonates a grain in the pile.

1. Click → search a 4-cell (`SORT_CLICK_RADIUS_CELLS`) Manhattan diamond around the click for the nearest grain of any type. If none, the click does nothing.
2. **Wave 0 (click AoE):** the seed grain plus its 4 orthogonal neighbors are destroyed. Every non-standard cell destroyed here is queued as a wave-1 detonator.
3. **Wave N+1 (chain):** every queued detonator triggers its own type's blast:
   - **Rare** → 4-orthogonal blast (same radius as the click AoE).
   - **Foil Standard** → 8-cell "+" pattern: 1- and 2-cell reach along each cardinal axis (N/S/E/W). Foils explode bigger.
4. Standards in any blast are consumed but don't propagate the chain.
5. Each destruction spawns an **expanding pop** in the rarity's color on the canvas. Waves are spaced by `CHAIN_STEP_MS` (90ms) so each pop reads.
6. **Every destroyed grain is fully sorted** — set progress, collection flash, market-key tracking, same as the old hand-sort.
7. After the chain finishes, the pile gravity-settles: every surviving grain above a destroyed cell becomes a falling grain at its previous visual Y and physics-falls down to fill the gap.

Outcomes:
- Click on a pure-standard area: just **5 cards** sorted (seed + 4 orthos).
- Click on a lone rare: **5 cards**, no further chain.
- Click on a lone foil: **9 cards** — the 5-cell click AoE plus 4 more cells the foil reaches at distance 2 along the axes.
- Click into a chain of orthogonally-connected rares: ripples outward 5 cells per rare, no diagonals.
- Click on a mixed cluster: the foil's "+" reach can pull in another non-standard several cells away that wouldn't be reachable from a rare alone.

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

Every rarity (bulk and priced, foil and non-foil) has its own craftable Set item — 10 tracks total. Bulk Sets consume cards from `bulkInvById`; priced Sets consume from the per-id `invById`. Each Set item is market-priced and listed via the marketplace just like any other priced item.

Bulk-set baselines (Standard / Rare / Foil Standard) are explicit — those rarities have no per-card market price for a multiplier to act on.

Priced-set baselines are **derived** at boot via `SET_CRAFT_MULTIPLIER` (default `1.15`):

```
BASELINE[setKey] = SET_SIZES[baseRarity] × BASELINE[cardKey] × SET_CRAFT_MULTIPLIER
```

| Set                  | Set size | Formula            | Baseline $ |
|----------------------|---------:|--------------------|-----------:|
| Standard Set         | 36       | explicit           | $30        |
| Rare Set             | 18       | explicit           | $200       |
| Foil Standard Set    | 36       | explicit           | $120       |
| Epic Set             | 9        | 9 × $5 × 1.15      | $51.75     |
| Legendary Set        | 6        | 6 × $25 × 1.15     | $172.50    |
| Mythic Set           | 3        | 3 × $100 × 1.15    | $345       |
| Foil Rare Set        | 18       | 18 × $30 × 1.15    | $621       |
| Foil Epic Set        | 9        | 9 × $250 × 1.15    | $2,587.50  |
| Foil Legendary Set   | 6        | 6 × $1,000 × 1.15  | $6,900     |
| Foil Mythic Set      | 3        | 3 × $4,000 × 1.15  | $13,800    |

So crafting a priced set sells for **+15% over the sum of singles**. Small premium → encourages set completion without making crafting strictly better than spot-selling. Bump `SET_CRAFT_MULTIPLIER` in the design panel to change the slope. Bumping a per-card baseline automatically bumps the set baseline that depends on it.

---

## 6. Marketplace

A 10-slot, single-tier market. Each slot holds one listing — either a priced single (mythic, non-standard foil) or a crafted Set.

### Listing tiles

Listings render as 30px miniature trading cards:
- Rarity-tinted background + border + glow
- Small rarity glyph in the top-left corner
- **Big centered price** (compact format — `$42`, `$1.5k`, `$25k`)
- Tiny card ID along the bottom (or `SET` for crafted sets)
- Gold countdown badge in the top-right
- Full unabbreviated price + rarity label in the `title` attribute for hover

Set listings get a stacked-cards illusion via offset shadows. Foil listings shimmer with a rainbow gradient. Empty slots render as dashed `+` placeholders so the slot cap is visible.

### Pricing & cooldown

Every listing posts at **1× current market price**. Resolve cooldown is uniform in `[LISTING_COOLDOWN_MIN, LISTING_COOLDOWN_MAX]` (default 10–20s). Every listing eventually sells.

Click a listing tile to **cancel and return** the card/set to inventory.

### Cap behavior

When 10 slots are full:
- The collection-grid click-to-list path shows "All 10 listing slots full" and refuses
- The vault's `LIST` buttons swap to `SLOTS FULL` text and disable

---

## 6b. Collection cells

The collection grid is now a pure visual progression tracker — 18px-wide cells that display only ownership, nothing else. No card numbers, no prices, no count badges, no listed stars. Owned cells fill with the rarity color (and shimmer for foils); unowned cells stay as faint outlines. Newly-collected cells animate via the existing card-pop flash.

**Listing singles is gone.** The only way to monetize cards is to complete a rarity's Set and craft + list that Set via the marketplace. So mythics, foil rares, foil epics, etc. all funnel through Set crafting now — no more impulse-listing a lucky pull.

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

## 8b. CLOUT — vault sets for XP

Every crafted set can either be **listed** for cash or **vaulted** for CLOUT. Vaulting is a one-way trade — the set is consumed and you can never list it later — but it earns progression XP that ratchets up your effective caps.

**CLOUT per set:** `ceil(sqrt(BASELINE[setKey]))`. Stays in sync with the set's market baseline, so retuning prices reshapes the curve automatically.

| Set                  | Baseline | CLOUT |
|----------------------|---------:|------:|
| Standard Set         | $15      | 4     |
| Rare Set             | $200     | 15    |
| Foil Standard Set    | $60      | 8     |
| Epic Set             | $52      | 8     |
| Legendary Set        | $173     | 14    |
| Mythic Set           | $345     | 19    |
| Foil Rare Set        | $621     | 25    |
| Foil Epic Set        | $2,588   | 51    |
| Foil Legendary Set   | $6,900   | 84    |
| Foil Mythic Set      | $13,800  | 118   |

**Level curve:** `level = 1 + floor(sqrt(clout / 10))` — quadratic, gentle early and steeper late.

| Level | CLOUT required |
|------:|--------------:|
| 1     | 0             |
| 2     | 10            |
| 3     | 40            |
| 4     | 90            |
| 5     | 160           |
| 6     | 250           |
| 7     | 360           |
| 8     | 490           |
| 9     | 640           |
| 10    | 810           |

**Unlocks (`CLOUT_BONUSES`):**

| Level | Bonus                |
|------:|----------------------|
| 2     | +1 marketplace slot  |
| 3     | +1 max sorter        |
| 5     | +2 marketplace slots |
| 7     | +1 max sorter        |
| 10    | +3 marketplace slots |

So a Lv-10 player runs **16 listing slots** (vs 10 base) and **5 sorters** (vs 3 base). Effective caps recompute live — no migration needed when crossing a threshold.

A `Lv N · X CLOUT · Y to next` chip lives in the page header and pops with a pink flash on level-up. The vault button on each set track is magenta to visually separate it from the gold LIST action.

---

## 9. Helpers — Homies & Sorters

### Hire Homie — temporary auto-ripper

- **Cost:** $20 + 1 sealed box
- **Effect:** spawns an animated 🧢 sprite next to the pack opener with a live progress bar; auto-rips one pack every 3s from its personal 36-pack pool
- **Lifetime:** until the box is empty (~108s total, faster if you bought multiple homies — they tile to the right)
- Homie rips trigger the same pull / sand-pile / particle animations as manual rips, but don't drain `state.packSupply`

### Buy Sorter — permanent two-stage sorting machine

- **Cost:** $500 each, max **3** per save
- **Two buffers:** `input` (raw, manually loaded from the pile) and `output` (processed, ready to collect). The sorter only moves cards from `input` → `output`; it never reaches into the pile on its own.
- **Manual LOAD:** Click the blue **LOAD** button to scoop the entire bottom row of the sand pile into the sorter's input. LOAD is **all-or-nothing** — it requires both a *complete* bottom row (a live grain in every column) AND enough free combined capacity to fit the whole row. The button's `title` attribute names the failing constraint when disabled. Each loaded grain leaves a tombstone in the pile, and the rest of the pile gravity-settles down by one row.
- **Tick:** Every `sorterInterval(level)` ms the sorter pops one grain from input (weighted by what's loaded) into output. With input empty, the sorter idles.
- **Buffer cap:** 500 at Lv 1 (+10 per level). Counted across input + output combined. When at cap, LOAD refuses until you collect.
- **Manual COLLECT:** Click the gold-glowing **COLLECT** button to flush output into the collection — set-progress + flash animations fire as the cards land.

### Sorter upgrades

| Level | Interval | Capacity | Upgrade cost |
|------:|---------:|---------:|-------------:|
| 1     | 2000 ms  | 500      | —            |
| 2     | 1000 ms  | 510      | $500         |
| 3     |  667 ms  | 520      | $2,000       |
| 4     |  500 ms  | 530      | $4,500       |
| 5     |  400 ms  | 540      | $8,000       |

Floor on interval is 250ms; `SORTER_LEVEL_MAX = 5`. Upgrades are rate-focused — capacity grows by only +10 per level since 500 is already enough for many bottom-row loads in a row. Upgrade cost = `SORTER_UPGRADE_BASE × level²` ($500 base).

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

Market baselines (drift around these, configurable). Foils are priced one tier above the non-foil tier above them — a foil epic ($250) outvalues a non-foil mythic ($100), reflecting its ~4× rarity:
- Epic: $5, Legendary: $25, Mythic: $100
- Foil Rare: $30, Foil Epic: $250, Foil Legendary: $1,000, Foil Mythic: $4,000
- Standard Set: $30, Rare Set: $200, Foil Standard Set: $120

---

## 12. Market Simulation — NPC inventory model

Prices are driven by simulated NPC supply, not a periodic ticker.

Each priced (key, id) — every per-id mythic/foil card and every set item — has a virtual NPC inventory level `state.marketInv[key][id]`. The price is computed lazily whenever it's read:

```
price = baseline × (MARKET_TARGET_STOCK / max(1, currentInv))
       clamped to [baseline × PRICE_FLOOR_RATIO, baseline × PRICE_CEIL_RATIO]
```

Defaults: `MARKET_TARGET_STOCK = 5`, `PRICE_FLOOR_RATIO = 0.2`, `PRICE_CEIL_RATIO = 7`. So:
- **5 in NPC stock** → price = baseline
- **1 in NPC stock** → price = 5× baseline (rare, expensive)
- **25 in NPC stock** → price = 0.2× baseline (flooded, cheap)

Events that move inventory:

- **Listing sells (player → NPC)** → `marketInv[key][id] += 1`. The very next sale of that same id fetches less.
- **NPC absorption** → no real ticker. When the next price is read, `decayMarketInv` lazily subtracts `elapsed_seconds × MARKET_DECAY_RATE` from the inventory (default 0.05/sec → 1 card absorbed per 20s of real time).
- **Listing cancelled / new pull** → market unaffected.

The lazy-compute pattern means prices only update when the game reads them — `renderVault` doesn't tear down buttons on a 7-second ticker anymore.

Dramaturgy: flood the market with foil epics in a session and prices visibly tank. Take a break, come back later, NPCs have absorbed the surplus, prices have recovered. Permanent crashes are impossible (decay eventually wins), but short-term swings are real.

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
