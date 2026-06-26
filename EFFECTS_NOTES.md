# Keyword & Effect Implementation Notes

How the 120-minion keyword system is implemented in `index.html`, and where the
implementation takes deliberate shortcuts vs. the "ideal" card text. Card text lives
in `minions.json`; runtime behaviour lives in the inline `<script>` (engine + the
`EFFECTS` registry).

## Architecture

- **Generic keywords** are handled directly in the engine and work on *any* minion that
  has the field, no per-card code:
  `taunt`, `shield` (Divine Shield), `cleave`, `windfury`, `reborn`, `toxin`,
  `bulwark`, `kindle`, `haunt`, `plunder`, `apex`, `brood`, `suppress`, `reflect`.
- **Per-card effects** live in `EFFECTS[fx]` as hooks: `battlecry`, `onBuy`, `onSell`,
  `startTurn`, `endTurn`, `startCombat`, `onAttack`, `allyDeath`, `toxinDeath`,
  `onSummon`, `deathrattle`, `endCombat`.
- **Permanent (cross-combat) buffs** are collected into `ctx.perma` during combat and
  written back to the real board minion by `id` afterwards (`applyPerma`).

## Simplifications & shortcuts

These are intentional approximations — the game plays correctly, but the behaviour is
not a literal reading of the card text. Listed by area.

### Mirrors / Reflect
- **Reflect is resolved by rewriting the clone's tribe** at combat start
  (`resolveReflectTribes`). Consequence: Reflect affects **combat** tribe synergies
  (Kindle, Swarm, Raider/Titan combat triggers) but **not buy-phase** tribe buffs —
  a Mirror on your board still counts as "Mirrors" when, say, a Raider buff is handed
  out during the recruit phase.
- **Prism / The Infinite** ("counts as *all* tribes") instead join the single **biggest
  tribe package** on the board. Prism's downside ("gains no stats from tribal buffs")
  is **not enforced**.
- **Mimic** ("both neighbours") resolves to a single effective tribe — the left
  neighbour, falling back to the right only if the left is absent/Mirrors.
- **Hall of Mirrors** ("Reflect minions count as two copies") is approximated as
  **+2/+2 to friendly Reflect minions** at combat start, rather than literal double-counting.
- **Glass Shard / Reflection / Doppel / Kaleidos** copy *keyword fields* (and, for
  Reflection, the `fx` hook) — not a literal full clone. Kaleidos picks the highest
  stat-total ally as "strongest".
- **Perfect Mirror** triggers each neighbour's battlecry (with itself as the target) and
  adopts one neighbour's deathrattle `fx`; it does not stack two separate copied effects.

### Wardens
- **Counterspell** uses **Attack as the "power" proxy**: it blocks enemy
  start-of-combat effects whose Attack is lower than Counterspell's.
- **Sentinel Prime** cancels and re-runs the **highest-Attack** enemy start-of-combat
  effect for its own side.
- **Mute Colossus** ("enemies can't gain stats") is approximated by **zeroing enemy
  Kindle and Haunt** at combat start (the main in-combat growth engines), rather than
  intercepting every stat mutation.
- **Null Sovereign** grants +2/+2 to a random friendly each time a friendly effect
  suppresses an enemy keyword (hooked inside `stripKeywords`).
- **Suppress** (the keyword) strips the **board-adjacent** enemy minions (indices
  i-1, i, i+1), an approximation of positional adjacency across two boards.

### Spores
- **Sporemother** grants Toxin to all friendly minions for the **whole combat**
  (card text: only their *first* attack).
- Fully Bulwark-absorbed hits apply **no Toxin** (keeps Adamant Colossus' "immune while
  Bulwark holds" consistent).

### Phantoms
- **The Reckoning** buffs **itself** +1/+1 per friendly death (card text: a random
  friendly) so the permanent buff reliably maps back to a board minion.
- **Grave Titan** resummons a fresh minion with the last-dead's **base stats**, not the
  exact buffed instance.

### Raiders
- **First Mate** ("attacks immediately") is implemented as one extra start-of-combat
  strike.
- **Krakenlord** splits its Attack as `floor(atk / enemies)` to each enemy (even split).

### Alchemists
- **Transmuter / Elixir Master** discounts are applied mechanically (the gold charged in
  `buyMinion` is reduced via `applyShopDiscounts`), but the **current card UI does not
  display the reduced price**. Discounts recompute on each shop refresh; a frozen shop
  keeps its discounted costs.
- **Shop-cost discounts that need ordering** are applied at refresh time, so a Transmuter
  bought *this* turn discounts the shop from the *next* refresh onward.

### Engine / presentation
- The combat overlay shows the **start board, then the final board**, animating attack
  flashes; intermediate summons and start-of-combat effects aren't individually
  animated (the final state is correct).

## Fully implemented (no shortcut)

All other keywords and effects behave as written, including: Toxin DoT (escalating,
ticks at the start of the poisoned minion's attacks), Bulwark absorb + on-break effects
(Magma reflect, Unbroken AoE, Worldcore refresh), Kindle ramp + sharing (Forgekeeper)
+ Sunshatter health, Haunt, Brood tokens (+ Drone Captain / Brood Queen / Overmind /
Swarmguard token modifiers), Plunder, Apex (+ Singularity doubling), Windfury, Reborn
(+ Phoenix full-health variant), Cleave, Necrolord deathrattle doubling, Gigaton
survive-once, Grand Inquisitor deathrattle lockout, and the permanent scalers
(Eternal Rot, Eternal Flame, Genesis Engine, The Grand Work, Philosopher, Transcendant).
