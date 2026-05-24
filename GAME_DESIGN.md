# Ripping Packs — Game Design Document

A single-file browser idler about cracking trading-card booster packs, sorting the sand pile, and selling completed sets on a live market. Built as one HTML file with vanilla JS + CSS. All gameplay state persists in `localStorage`.

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

> **Multi-set context.** The game ships with 30 yearly sets (2001 – 2030). The
> sections below describe the loop *within an active year*. Set 30 is the
> current year and the only one unlocked at fresh-boot — older years (the
> "vintage shop") open up as you craft Complete Sets. The cross-year
> structure, pricing curve, CLOUT scaling, and per-year state buckets are
> covered in detail in **§19 Multi-Set / Vintage System**.

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

## 8b. CLOUT — craft + vault a Complete Set

CLOUT comes from one path only: **vaulting a COMPLETE SET**. There are two flavors, split by foil status so the (much rarer) foil ingredients don't gate progress entirely:

| Set                  | Ingredients (5 each)                                           | Baseline   | Vault → CLOUT |
|----------------------|----------------------------------------------------------------|-----------:|--------------:|
| **Complete Set**     | standardSet, rareSet, epicSet, legendarySet, mythicSet         |  $1,500    |  50           |
| **Foil Complete Set**| foilStandardSet, foilRareSet, foilEpicSet, foilLegendarySet, foilMythicSet | $35,000 | 500           |

The non-foil path is the "starter" trickle of CLOUT — quick to assemble once you have a couple of full sets, modest payout. The foil path is the late-game push — 5× the per-set value but each ingredient takes orders of magnitude longer to pull.

Each Complete Set lands in `state.invById[key]` and can be:

- **Vaulted** for the set's flat `clout` value — one-way, consumes the Complete Set.
- **Listed** for cash on the marketplace at the baseline price.

Per-rarity sets have no vault path anymore — they're ingredients (or cash, via LIST). Both Complete Set rows sit at the top of the vault column. The non-foil row uses a gold accent; the foil row gets a full rainbow border + shimmering label. Each row's CRAFT button shows `X/5` progress until all five matching ingredients are held, then lights up.

Sorter upgrades are no longer CLOUT-bought — they cost cash via each sorter card's `⬆ $X` button. Three branches remain on the upgrade grid:

| Branch | Tier 1 → ... | What it does |
|--------|-------------|-------------|
| PAINT  | Steadier Hand (8) → Wider Sweep (25) → Pile Mastery (80) → Steady Aim (250), Bigger Brush (100) → Massive Brush (300) → Mop Brush (700), Quick Eye (40) → Snap Focus (120) → Reflexes (350) | +marks / +cursor radius / -paint dwell |
| LISTINGS | Side Hustle (5) → Shop Front (20) → Storefront (60) → Online Empire (200) | +1 / +1 / +2 / +3 marketplace slots |
| MARKET | Quick Sales (20) → Hot Demand (80) → Premium Sets (250) | listing cooldown -3s, NPC absorb +0.03/s, set craft mult +0.10 |

Costs in CLOUT; numbers above are the node cost. Effects are deltas added to the base constants via `unlockedNodeEffects()`.

**CLOUT is a spendable balance**, not a running total. The header pink chip shows `N CLOUT · UPGRADES` and opens the upgrade grid modal. Click any **available** node to spend CLOUT and apply the bonus live — effective caps recompute on the spot.

Modal UI: three columns (one per branch), each node card shows its name, effect, cost, and status — **owned** (gold border), **available** (pink pulse, clickable), **unaffordable** (dim pink), **locked** (dashed, prereqs unmet).

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
| **Shop** (left)| Buy buttons (Pack/Box/Case), Hire Homie / Buy Sorter, set-progress meters, the pack opener, typing zone, hint line, sand canvas, sorter cards |
| **Vault** (mid)| Header (coins, packs opened, foils), marketplace listings panel, collection grid (cells per rarity row, foil row beneath each), then the three bulk tracking rows (Rare / Foil Standard / Standard) with set craft + list buttons |
| **Footer**     | One-line flavor copy                                              |

Mobile collapses to single column. Cells shrink to 28px and the standard grid stays 18-wide.

---

## 14. State Schema (high level)

Save schema is **v5**. Top-level state holds cross-set fields (cash, CLOUT,
upgrade nodes, listings, staff); per-year inventory lives under
`state.sets[setId]` and is lazy-materialized — touched years exist, others
don't.

```
state = {
  // ---- cross-set --------------------------------------------------------
  coins, packsOpened,
  activeSetId,                   // 1..30; which year you're viewing/ripping
  clout, cloutSpent,             // spendable balance + lifetime spend
  unlockedNodes: [],             // upgrade tree (CLOUT-bought, global)
  completeSetCrafts: { [setId]: count },
                                 // lifetime non-foil Complete Set crafts
                                 // per year — drives the vintage unlock gate
  homies:   [{ uid, slot, setId, packsRemaining, lastRipAt, startAt }],
  sorters:  [{ uid, level, setId, input{}, output{}, lastSortAt }],
  listings: [{ uid, setId, rarity, cardId, foil, askPrice, resolveAt,
               willResolve }],
  listingsUidCounter, homieUidCounter, sorterUidCounter,
  marketVersion, setVersion,     // schema stamps

  // ---- per-year buckets — lazy-materialized via setStateFor(setId) ------
  sets: {
    [setId]: {
      packSupply, boxesOwned,
      unsorted:   { standard, rare, foilStandard },
      stock:      { [rarity]: count, non-foil },
      stockFoil:  { [rarity]: count, foil },
      opened, openedFoil,        // lifetime pull counters (per year)
      pile: [[grainKey,...], ...],
      owned:     { [rarity]: [unique IDs collected] },
      ownedFoil: { [rarity]: [unique foil IDs collected] },
      setsCelebrated, setsCelebratedFoil,   // first-completion flags
      invById:     { [marketKey]: {id:count} },   // priced + set items
      bulkInvById: { standard, rare, foilStandard: {id:count} },
      vaultedSets: { [completeSetKey]: count },   // lifetime vault deposits
      marketInv,                                  // NPC market level per
      marketInvUpdated,                           // (key,id), per year
    },
  },
}
```

Two accessors do almost all the work:

- **`setStateFor(setId)`** — returns the per-year bundle, creating an empty
  one on first touch.
- **`cur()`** — shortcut for `setStateFor(state.activeSetId)`. Almost every
  inventory-touching code path uses `cur()` so it reads/writes the active
  year's bundle.

A third helper, **`withActiveSet(setId, fn)`**, temporarily swaps
`state.activeSetId` for the duration of `fn` and restores it in a `finally`
block. Used for staff operations that must mutate a year other than the one
currently being viewed (a Set 25 homie ripping while the player is on Set 30).

Save migration: bumping `SAVE_KEY` v4 → v5 means old saves are silently
ignored — the player starts fresh on the new schema. (Legacy migration
tail in `load()` was removed during the refactor; future schema bumps will
similarly assume reset.) `load()` does minimal sanity backfills against the
v5 shape: top-level cross-set fields default to safe values, each per-year
bundle is re-skeletoned against `blankSetState()` for any missing fields,
and pile tombstones from interrupted chain animations are cleaned.

---

## 15. Configurability — Game Design Panel

Press the **DESIGN** button to open a modal exposing every tunable as a numeric input, grouped: Economy, Pack Odds, Sorting, Marketplace, Staff, Market Baselines. Values persist in `localStorage` per-browser. **Apply & Reload** restarts the game with the new constants. A **RESET** button at the bottom wipes the save entirely.

This makes the whole game a live design sandbox — every drop-rate, every baseline, every cooldown, every cost is hot-swappable for tuning.

---

## 16. Out-of-Scope (current build) / Roadmap Hooks

- **Multiplayer / trades:** none.
- **Cosmetics:** no custom card art or alternate sleeve themes yet. (Cards
  across years are visually distinguished only by the patina tint and corner
  setId chip — no per-set art exists.)
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

## 19. Multi-Set / Vintage System

The game ships with **30 yearly sets, Set 01 (2001) through Set 30 (2030)**.
Set 30 is the current year and the only set unlocked at fresh-boot; older
years open up through the "vintage shop" as you make progress in the
current year. All 30 years share the same card layout, rarities, and core
loop — they differ only by their `setId` tag, visual patina tint, pricing,
and CLOUT reward scaling.

### 19.1 The active-set pointer

`state.activeSetId` is the single source of truth for "which year is the
player currently working on." It's mutated by:

- Clicking a year chip in the year picker (above the shop's buy buttons)
- Clicking a vault summary stamp (above the listings panel)
- Internally, the `withActiveSet(setId, fn)` helper that wraps staff ops
  (homie rips, sorter load/collect) so they mutate their pinned year's
  bundle, not whatever the player happens to be viewing

The active set drives:
- Which year's pack supply / box stash you rip from
- Which year's vault grid + craft tracks render
- Which year's sand pile + sorters animate
- Pack/box/case price labels in the shop
- Patina filter on the active workspace

### 19.2 The year picker

A horizontal scroll of 30 chips sits above the buy buttons:

```
[S30 '30 ★] [S29 '29] [S28 '28 🔒] [S27 '27 🔒] ... [S01 '01 🔒]
```

- Active year highlighted in gold
- Unlocked years readable in their patina tier color
- Locked years are dimmed with a 🔒 prefix and a tooltip explaining how to
  unlock
- Click an unlocked chip → switch active set (full re-render, save fires)
- Click a locked chip → hint line explains the unlock requirement

### 19.3 Unlock gate — "graduate to vintage"

A year **Set N** unlocks when you've crafted at least one **non-foil Complete
Set in Set N+1**. Set 30 is always unlocked.

```
isSetUnlocked(setId):
  if (setId == 30) return true
  return state.completeSetCrafts[setId + 1] > 0
```

Lifetime non-foil crafts are tracked at `state.completeSetCrafts[setId]`;
the counter increments inside `craftCompleteSet` for non-foil only. Foil
Complete Sets are not a shortcut — they're the apex prize, not the
progression gate.

The full vintage ladder takes **29 sequential non-foil Complete Set crafts**
to walk back from Set 30 to Set 01.

### 19.4 Pricing curve — buying

Pack / box / case cost scales **linearly** with set age:

```
setAgeMult(setId)  = max(1, 31 − setId)
packCost(setId)    = PACK_COST * setAgeMult(setId)
boxCost(setId)     = BOX_COST  * setAgeMult(setId)
caseCost(setId)    = CASE_COST * setAgeMult(setId)
```

| Set | Mult | Pack | Box   | Case  |
|----:|----:|------|-------|-------|
| 30  | 1×  | $3   | $100  | $500  |
| 25  | 6×  | $18  | $600  | $3,000|
| 15  | 16× | $48  | $1,600| $8,000|
| 01  | 30× | $90  | $3,000| $15,000|

Vintage packs cost a lot; the rewards scale separately (next section).

### 19.5 Reward curve — selling + CLOUT

Sell-price baselines **and** Complete Set vault CLOUT both scale by a
**gentler** curve:

```
cloutScaleFor(setId) = 1 + max(0, 30 − setId) * 0.07
```

| Set | Scale | Complete Set CLOUT | Foil Complete CLOUT | Mythic sell mult |
|----:|------:|-------------------:|--------------------:|------------------:|
| 30  | 1.00× | 50                 | 500                 | 1.00× baseline   |
| 25  | 1.35× | 68                 | 675                 | 1.35× baseline   |
| 15  | 2.05× | 103                | 1,025               | 2.05× baseline   |
| 01  | 3.03× | 151                | 1,515               | 3.03× baseline   |

This is applied in two places:

- **`marketPriceFor(setId, key, id)`** multiplies `BASELINE[key]` by
  `cloutScaleFor(setId)` before the NPC inventory factor.
- **`vaultCompleteSet`** awards `round(def.clout * cloutScaleFor(activeSetId))`
  CLOUT.

**Why the asymmetric curves?** Pack costs scale steeply (30×) so vintage
hunting is a real cost commitment, but rewards scale gently (3×) so
chasing old years is a CLOUT play, not a cash arbitrage. Set 01 foil
Complete Set is the apex prize at **1,515 CLOUT** per vault deposit.

> **Tuning note.** The 0.07/year gentle scale was tentatively set to make
> vintage hunting expensive-but-not-impossible. A real playtest may show
> the cost curve dominates and vintage feels unrewarding — if so, bump the
> scale (0.10? 0.15?) or flip pack cost to also use the gentle scale.

### 19.6 Per-year state buckets

See §14 for the full schema. Conceptually:

- **Shared at top level:** cash, lifetime packs opened, CLOUT (it's a
  cross-year currency), upgrade nodes, listings (each tagged with its
  setId), homies, sorters.
- **Per year (`state.sets[setId]`):** pack supply, sealed boxes,
  inventory (priced + bulk), unsorted pile, sand pile, owned card IDs,
  set-completion flags, vault deposits, **NPC market levels**.

Each year has its **own** NPC market — a Set 25 mythic and a Set 30 mythic
are different items with separately-flooding inventories. Listing a glut
of Set 25 mythics doesn't drop Set 30 mythic prices.

Bundles are **lazy-materialized**. A fresh boot only writes `state.sets[30]`;
older years are created on demand via `setStateFor(setId)` (called by
`cur()`, by buying packs of that year, by hiring a homie there, etc.).
Most saves will only have 2–4 entries in `state.sets` — the years the
player has actually touched.

### 19.7 Listings across years

`state.listings` stays a **flat array** — set listings, crafted-set
listings, and complete-set listings all live together. Each listing
record carries its own `setId`. This means:

- **Render** is set-agnostic — the marketplace shows all your active
  listings regardless of which year you're viewing.
- **Cancel** returns inventory to `setStateFor(l.setId)`, not `cur()` —
  so a S25 listing cancelled while you're on S30 returns the set item to
  the right bundle.
- **Resolve** (expired path) likewise routes to the listing's own bundle.
- **NPC absorption** on sale: `npcAbsorbSale(l.setId, l.rarity, l.cardId)` —
  each year's market inflates only from sales within that year.

The 10-slot listing cap (`LISTING_SLOTS`, upgradable) is shared — you can
hold 10 listings total across all years, not 10 per year.

### 19.8 Staff pinned to their hire year

Homies and sorters both record their `setId` at hire/install time:

```
state.homies.push({ uid, slot, setId: activeSetId, ... })
state.sorters.push({ uid, level, setId: activeSetId, ... })
```

- **Hiring** consumes a box from the **active year's** stash. If you want
  a Set 25 homie, you need to buy a Set 25 box first.
- **Homie rips** wrap `homieRipPack(homie)` in `withActiveSet(homie.setId,
  fn)` so the rip mutates the homie's year's inventory. Visual side
  effects (falling sand, hit pops, particle confetti, flash queue) are
  gated on `homie.setId === state.activeSetId` ("watching") — off-screen
  rips silently update inventory without spamming the active view.
- **Sorter LOAD/COLLECT** likewise wrap their inner functions in
  `withActiveSet(sorter.setId, fn)`. The sorter pulls from its pinned
  year's pile and writes to its pinned year's stock.
- **Sprites + sorter cards** show a small `S{N}` badge so the year tag is
  visible at a glance.

This means you can run staff in parallel across years — a Set 30 homie
ripping current packs while a Set 25 sorter chews through your vintage
pile in the background, no interference.

#### The `fallingGrains` setId tag

Sand grains in mid-fall are tagged with their origin setId. Three rules
keep them tidy:

- **`syncPileWithStock`** only counts in-flight grains for the active set
  when checking if the pile needs a rebuild (prevents off-screen grains
  from corrupting the active set's pile reconciliation).
- **`drawSand`** skips grains whose `setId !== activeSetId` so off-screen
  rips don't visually bleed into the wrong canvas.
- **`stepSandAnim`** settles off-screen grains *instantly* into their own
  year's pile (the player isn't watching them animate anyway).
- **`rebuildPileFromStock`** drops the rebuilt year's in-flight grains to
  avoid double-counting when they would otherwise later land.

### 19.9 Visual patina — three tiers

The age tier drives a sepia CSS filter applied to the player's active
workspace:

| Set range | Tier         | Filter                                              |
|-----------|--------------|-----------------------------------------------------|
| 20–30     | "" (current) | none — full color                                   |
| 10–19     | midcentury   | `sepia(0.25) hue-rotate(-6deg) saturate(0.92)`      |
| 01–09     | vintage      | `sepia(0.55) hue-rotate(-12deg) saturate(0.85) brightness(0.95)` |

The tier is applied to **the pack, the vault grid, and the stock-rows
container** in `renderHeader`. Year-picker chips, vault summary stamps,
and individual listing cards each carry their own tier class — so when
viewing the marketplace, you can read each listing's year tier (current,
midcentury, vintage) at a glance.

### 19.10 Vault summary

A new horizontal strip of "vault stamps" lives between the marketplace
listings and the per-year vault grid. Each stamp:

```
[S30 '30 ★1]  [S29 '29 —]  [S28 '28 —]  ... [S25 '25 ★1 ✨1]
```

- Visible for any year with vault activity OR unlock status OR any owned
  cards (Set 30 is always visible as the anchor)
- Shows non-foil count (`★N`) and foil count (`✨N`) of vaulted Complete
  Sets
- Tooltip includes the per-year unique-card-owned total
- Active year highlighted gold
- Patina tier class applied so old years read as antique stamps
- Click → switch active set (same gate as the year picker)

Per-year vault grid title shows `COLLECTION · SET {N} · {year}` so the
title bar always tells you which year you're inspecting.

### 19.11 Sold / expired / cancelled messages

Listing-resolution hints prefix the year, e.g.

- `"Sold Set 25 FOIL MYTHIC SET @ $5,432."`
- `"Set 5 listing expired — returned to inventory."`
- `"Cancelled Set 12 listing — returned to inventory."`

When multiple listings resolve in the same tick, only the last hint shows
— but the year tag makes it unambiguous which one paid out.

### 19.12 Open design questions

These were intentionally left as v1 defaults pending real playtest:

- **Cost-vs-reward curve.** Linear pack costs (31−setId) with gentle
  reward scaling (1 + 0.07×age) makes vintage hunting a CLOUT-only play.
  If it feels like a money pit in practice, candidate fixes: bump CLOUT
  scale to 0.10–0.15/year, OR flip pack cost to gentle, OR drop pack
  cost scaling entirely and let only CLOUT scale.
- **Unlock gate granularity.** One non-foil Complete Set per year is the
  current threshold. If players grind 5+ before moving on, the gate
  feels symbolic; if they barely make 1, it feels punishing. Watch
  median crafts-per-year for the first 3 sets unlocked.
- **Cross-year sorter ergonomics.** A Set 25 sorter installed beside the
  Set 25 pile is invisible to a player who's currently on Set 30 (the
  sorters modal lists all of them, but the active canvas only shows the
  current year's pile). May want a "switch to S25" shortcut in the
  modal when clicking on a non-active-year sorter card.
- **Per-card visual differentiation.** Cards across years currently
  differ only by their setId tag and the patina filter applied at the
  grid level. Per-year card art, alternate frames, or holo variants
  are pure cosmetics work and out of scope so far.

---

*End of design doc.*
