# DefaultExecutor Design Document

## Overview

`DefaultExecutor` is the abstract base class that all WindBot deck AIs inherit from. It sits one level above `Executor` in the class hierarchy and provides:

- Pre-built logic for ~50 commonly played cards, usable as-is or overridden.
- Default battle-phase targeting logic that accounts for power comparisons, invincibility, dangerous effects, and equip cards.
- Guard rails that prevent the bot from walking into obvious traps (negation fields, dangerous defenders, etc.).
- A central registry of card IDs (`_CardId`) shared across all child AIs.

Custom executors declare themselves with the `[Deck("Name", "file.ydk")]` attribute and call `AddExecutor()` in their constructor to register card handlers.

---

## Class Structure

```
Executor  (framework base)
  └── DefaultExecutor  (this file — abstract)
        └── YourDeckExecutor  (your custom AI)
```

`DefaultExecutor` is `abstract` and cannot be instantiated directly. Its constructor takes `GameAI ai` and `Duel duel`, passing them up to `Executor`, and then registers two built-in handlers: `DefaultChickenGame` for Chicken Game and a bare activation for Santa Claws.

---

## Inner Class: `_CardId`

A `protected` nested class of integer constants covering every card whose logic is hard-coded into `DefaultExecutor`. Grouped conceptually:

| Group | Cards |
|---|---|
| Kaiju monsters | Jizukiru, Thunder King, Dogoran, Radiant, Gadarla, Kumongous, Gameciel, Mecha Dogoran |
| Timelords | Sandaion, Gabrion, Michion, Zaphion, Hailon, Raphion, Sadion, Metaion, Kamion, Lazion |
| Exodia pieces | All five pieces |
| Dangerous monsters | Ultimate Conductor Tytanno, El Shaddoll Construct, Ally of Justice Catastor |
| Attack redirectors | Dupe Frog, Marauding Captain |
| Board-wipe threats | Black Rose Dragon, Judgment Dragon, Topologic Trisbaena, Exciton Knight, Harpie's Feather Duster, Dark Magic Attack |
| Removal spells | MST, Cosmic Cyclone, Galaxy Cyclone, Book of Moon, Compulsory Evacuation Device |
| Traps | Call of the Haunted, Scapegoat, Breakthrough Skill, Solemn Judgment/Warning/Strike, Torrential Tribute, Evenly Matched |
| Mass removal | Heavy Storm, Hammer Shot, Dark Hole, Raigeki, Smashing Ground |
| Draw/search | Pot of Desires, Allure of Darkness, Upstart Goblin, Cyber Emergency |
| Hand traps | Ash Blossom, Maxx C, Lock Bird, Ghost Ogre, Ghost Belle, Effect Veiler, Artifact Lancea |
| Meta counters | Called by the Grave, Infinite Impermanence, Dimensional Barrier, Interrupted Kaiju Slumber |
| Lockdown spells | Imperial Order, Royal Decree, Naturia Beast, Anti-Spell Fragrance |
| Extra-deck staples | Castel, Crystal Wing, Utopia / Utopia the Lightning, Ultimaya Tzolkin, Mekk-Knight Crusadia Astram |
| Equip / continuous | Moon Mirror Shield, Phantom Knights' Fog Blade |
| ATK boosters | Vampire Fraeulein, Injection Fairy Lily |
| Field threats | Hamon Lord of Striking Thunder, Chicken Game, Blue-Eyes Chaos MAX Dragon |
| Misc | Scapegoat, Santa Claws, Macro Cosmos, Galaxy Soldier, Eater of Millions |

Child executors access these as `_CardId.SomeName`.

---

## Battle Phase Logic

### `OnSelectAttackTarget(ClientCard attacker, IList<ClientCard> defenders)`

The main combat decision loop. For each defender (sorted externally by the caller):

1. Resets `attacker.RealPower` to base ATK and `defender.RealPower` to its effective defense power.
2. Calls `OnPreBattleBetween` — if it returns `false`, that matchup is skipped.
3. Attacks if `attacker.RealPower > defender.RealPower`, or if equal **and** the attacker is the last attacker **and** the defender is in attack position (tiebreaker rule).
4. Falls back to a direct attack if no target was chosen and `attacker.CanDirectAttack`.
5. Returns `null` to signal "this attacker cannot safely attack anything."

### `OnPreBattleBetween(ClientCard attacker, ClientCard defender)`

A comprehensive safety filter called before every potential attack. It mutates `RealPower` values to reflect in-battle effects and returns `false` to block an attack. Key checks:

**Defender-side blocks** (skipped if attacker has a prevent-activation effect in battle):

| Condition | Action |
|---|---|
| Defender is invincible + defense position | Block |
| Defender is dangerous (unless attacker is UCT/El Shaddoll/Catastor with matching conditions) | Block |
| Defender equipped with undisabled Moon Mirror Shield | Block |
| Mekk-Knight Crusadia Astram (attack pos) vs. special-summoned attacker | Block |
| Crystal Wing (attack pos) vs. Level 5+ attacker | Block |
| Ally of Justice Catastor vs. non-DARK attacker | Block |
| Number S39 Utopia the Lightning with 2 Utopia materials | Set `defender.RealPower = 5000` |
| Vampire Fraeulein | Add up to 3000 to `defender.RealPower` |
| Injection Fairy Lily (enemy LP > 2000) | Add 3000 to `defender.RealPower` |

**Attacker-side boosts** (skipped if defender has a prevent-activation effect in battle):

| Condition | Action |
|---|---|
| Utopia the Lightning with 2 Utopia materials | Set `attacker.RealPower = 5000` |
| Eater of Millions (undisabled) | Set `attacker.RealPower = 9999` |
| Attacker is invincible | Set `attacker.RealPower = 9999` |
| Attacker equipped with Moon Mirror Shield | Set `attacker.RealPower = defender.RealPower + 100` |

**Field-wide blocks** (checked regardless of battle effects):

- Enemy has undisabled Mekk-Knight Crusadia Astram and the target is not Astram → block.
- Enemy has undisabled Dupe Frog and the target is not Dupe Frog → block.
- Enemy has undisabled Marauding Captain and the target is not Captain and is a Warrior → block.
- Target is Ultimaya Tzolkin (undisabled) and enemy has another Synchro → block.
- Any non-target enemy monster is an undisabled defense-position Hamon → block.
- Target has an undisabled Phantom Knights' Fog Blade attached → block.

### `OnSelectBattleReplay()`

Called when the game asks whether to re-enter the battle phase after a replay trigger. Re-uses `OnSelectAttackTarget` against the current battling monster; returns `true` only if another valid attack target still exists.

---

## Pre-Activation Gate

### `OnPreActivate(ClientCard card)`

Runs before any card activation is attempted. Returns `false` (blocks activation) in two cases:

1. **Timelord chain protection**: If a Timelord effect is currently on the chain during the Standby Phase, nothing else activates — letting the Timelord resolve cleanly.
2. **Negation awareness**: If the card is in hand or face-down in the spell zone, and the board state means spells will be negated (Imperial Order/Naturia Beast) or traps will be negated (Royal Decree), the card is not played. Prevents wasted activations.

---

## Card Position Selection

### `OnSelectPosition(int cardId, IList<CardPosition> positions)`

When the AI must choose a position for a card being summoned/set:

- If the card's ATK is 0, always choose face-up Defense.
- Otherwise returns `0`, delegating the decision to the framework default.

---

## Summon-or-Set Decision

### `OnSelectMonsterSummonOrSet(ClientCard card)`

Returns `true` to set a monster face-down instead of summoning it when all three conditions are true:

- The card is Level 4 or lower.
- The bot has no face-up monsters.
- Every enemy monster has higher ATK (using `Util.IsAllEnemyBetterThanValue`).

This avoids suiciding a low-ATK monster into a stronger field.

---

## Default Card Handlers

Each method below is a `protected bool` intended to be passed directly to `AddExecutor` or called from a child class override. They all return `true` to activate/use and `false` to skip.

### Spell/Trap Removal

#### `DefaultMysticalSpaceTyphoon()`
- Blocks if MST is already on the chain (no double-MST).
- Targets: floodgates first, then face-down spells on the bot's turn, or continuous/equip/field spells on the opponent's turn.

#### `DefaultCosmicCyclone()`
- Wraps `DefaultMysticalSpaceTyphoon`.
- Also requires bot LP > 1000 (pays the 1000 LP cost safely) and no Cosmic Cyclone already on chain.

#### `DefaultGalaxyCyclone()`
- From grave: targets the best enemy spell (face-up priority).
- From hand/field: targets a face-down enemy spell.

#### `DefaultBookOfMoon()`
- Activates when all enemies are stronger.
- Targets the highest-ATK enemy effect monster that is Xyz or Level 5+ (not a Link).

#### `DefaultCompulsoryEvacuationDevice()`
- Primary: bounces a problematic enemy monster.
- Fallback: if this card is being targeted, bounces the best available enemy monster.

#### `DefaultCallOfTheHaunted()`
- Only activates if all enemies are stronger.
- Revives the highest-ATK revivable monster from the bot's graveyard.

### Monster Defense

#### `DefaultScapegoat()`
Complex activation conditions:
- Never activates if spells would be negated.
- Never activates on the bot's own turn.
- Always activates during the End Phase.
- Activates if this card is targeted.
- During the Battle Phase: activates if total attacking enemy ATK would lethal — **unless** the enemy has a monster that attacks through tokens (UCT, Purgatrio, Ancient Gear, Red Dragon Archfiend).

### Hand Traps

#### `DefaultMaxxC()`
Simple: activates only on the opponent's turn (`Duel.Player == 1`).

#### `DefaultAshBlossomAndJoyousSpring()`
Activates on opponent's chain, with an ignore list:
- Skips Macro Cosmos, Upstart Goblin, Cyber Emergency.
- Skips Danger! archetype hand-revelation effects.

#### `DefaultGhostOgreAndSnowRabbit()`
Activates via `DefaultTrap()` unless the last chain card is already disabled.

#### `DefaultGhostBelleAndHauntedMansion()`
Delegates directly to `DefaultTrap()`.

#### `DefaultEffectVeiler()`
Wraps `DefaultBreakthroughSkill`. Also returns `false` if the last chain card is:
- Galaxy Soldier with 3+ cards in enemy hand, or
- Effect Veiler or Infinite Impermanence (avoids chaining on top of these).

#### `DefaultCalledByTheGrave()`
Counters common hand traps on the opponent's chain (Maxx C, Lock Bird, Ghost Ogre, Ash Blossom, Ghost Belle, Effect Veiler, Artifact Lancea). Selects the matching target and checks `UniqueFaceupSpell` to avoid redundant copies.

#### `DefaultInfiniteImpermanence()`
Same Galaxy Soldier / self-chain exclusion as Veiler, then delegates to `DefaultDisableMonster`.

### Monster Disruption

#### `DefaultBreakthroughSkill()`
Requires `DefaultUniqueTrap()` (no same-named trap already face-up or chained), then `DefaultDisableMonster`.

#### `DefaultDisableMonster()`
Central "negate a monster" logic, checked in priority order:
1. On opponent's turn: finds a monster that should be disabled before it uses its effect.
2. Chains onto the last chain card if it is an enemy monster effect that isn't already disabled or untargetable.
3. During battle: disables Eater of Millions on the enemy battling monster.
4. At Battle Start (opponent's turn): disables Utopia the Lightning if it is present.

### Trap Counters

#### `DefaultSolemnJudgment()`
`DefaultTrap()` + not solely targeting this card + not activating on own summon.

#### `DefaultSolemnWarning()`
`DefaultTrap()` + bot LP > 2000 + not own-summon trigger.

#### `DefaultSolemnStrike()`
`DefaultTrap()` + bot LP > 1500 + not own-summon trigger.

#### `DefaultTorrentialTribute()`
Activates when all enemies are stronger and no friendly trap has already been chained this chain.

### Mass Removal

#### `DefaultHeavyStorm()`
Only if bot has fewer spells/traps than the enemy.

#### `DefaultHarpiesFeatherDusterFirst()`
Only if enemy has 2 or more spells/traps.

#### `DefaultHammerShot()`
At least one enemy monster is better than the bot's best.

#### `DefaultDarkHole()` / `DefaultRaigeki()` / `DefaultSmashingGround()`
All use `Util.IsOneEnemyBetter()` — at least one enemy monster has better ATK or DEF.

### Card Advantage

#### `DefaultPotOfDesires()`
Only if the bot's deck has more than 15 cards remaining (safe to banish 10).

#### `DefaultAllureofDarkness()`
Only if the bot has a DARK-attribute card in hand to banish.

#### `DefaultChickenGame()`
Multi-effect field spell handler:
- Blocks if multiple copies are registered (prevents infinite loop).
- If face-down, activates to set.
- Never activates if bot LP ≤ 1000.
- Effect 0 (draw): activates when bot LP ≤ enemy LP.
- Effect 1 (destroy): activates when bot LP > enemy LP.

### Advanced / Complex Logic

#### `DefaultDimensionalBarrier()`
Intelligently picks the extra-deck type to declare:

- On the opponent's turn: surveys enemy monsters for Tuner + non-Tuner (→ Synchro), same-level multiples (→ Xyz), Xyz-ready combinations, or Pendulum scales.
- Chains onto an active enemy extra-deck summon spell/effect.
- If this card is targeted: defaults to declaring Xyz.

#### `DefaultInterruptedKaijuSlumber()`
- From grave: selects highest-defense Kaiju to special summon for the bot (Gameciel first to preserve removal value).
- From hand/field: uses `DefaultDarkHole` condition; if true, tributes a problematic enemy monster (highest Kaiju first) and special summons a Mecha Dogoran or Gameciel to the bot's field.

#### `DefaultKaijuSpsummon()`
For individual Kaiju cards being special-summoned:
- If an enemy Kaiju is already present, only proceeds if this Kaiju's defense power beats theirs.
- Prefers floodgates → dangerous monsters → any enemy monster stronger than this Kaiju.

#### `DefaultNumberS39UtopiaTheLightningSummon()` / `DefaultNumberS39UtopiaTheLightningEffect()`
- Summon condition: at least one enemy monster has higher ATK than the bot's best.
- Effect condition: activates in battle if this card's current ATK is under 5000 and the enemy battling monster is in attack position, face-down, or has DEF ≥ this card's ATK.

#### `DefaultEvilswarmExcitonKnightSummon()` / `DefaultEvilswarmExcitonKnightEffect()`
- Summon: bot's total card count (monsters + spells + hand − 1 for this card) is fewer than the opponent's, AND the effect condition is met.
- Effect: bot has fewer on-field cards than the opponent, **or** bot's total ATK sum is lower.

#### `DefaultStardustDragonSummon()` / `DefaultStardustDragonEffect()`
- Summon: enemy's best power ≤ 2500 and bot's best ATK isn't ahead — or it's Turn 1 / Main Phase 2.
- Effect: activates from grave (self-revive), or chains opponent's destruction effect.

#### `DefaultCastelTheSkyblasterMusketeerSummon()` / `DefaultCastelTheSkyblasterMusketeerEffect()`
- Summon: a problematic enemy card exists.
- Effect: ignores Effect 0 (ATK swap); uses Effect 1 to detach and bounce the problematic card, selecting itself as material first.

#### `DefaultScarlightRedDragonArchfiendSummon()` / `DefaultScarlightRedDragonArchfiendEffect()`
- Summon: enemy best power ≤ 3000 and bot's best ATK isn't ahead — or the effect condition is met.
- Effect: counts special-summoned effect monsters with ATK ≤ Scarlight's; activates when enemy count ≥ bot count and enemy has at least one, or enemy has 3+.

#### `DefaultTimelordSummon()`
Only if the bot's monster zone is completely empty.

#### `DefaultHonestEffect()`
- From hand (battle effect): activates when the bot's battling monster is in attack position AND either it would lose the battle, or the ATK boost would be enough for a direct kill, or the combined boost overcomes a high-DEF defender.
- From field (recycle): activates only on Turn 1 or during Main Phase 2.

---

## Environment State Helpers

| Method | Meaning |
|---|---|
| `DefaultSpellWillBeNegated()` | Imperial Order or Naturia Beast is active (not currently being chained) |
| `DefaultTrapWillBeNegated()` | Royal Decree is active |
| `DefaultSpellMustSetFirst()` | Anti-Spell Fragrance is active on either side |
| `DefaultOnBecomeTarget()` | This card is being targeted, or a board-wipe (Exciton, Black Rose, Judgment Dragon, Topologic, Evenly Matched) or opponent-only wipe (HFD, Dark Magic Attack) is on the chain |

---

## Utility Helpers

| Method | Purpose |
|---|---|
| `DefaultTrap()` | True if opponent triggered the chain, or opponent just summoned and no chain player yet |
| `DefaultUniqueTrap()` | `DefaultTrap()` + no same-named bot trap already chained or face-up |
| `UniqueFaceupSpell()` | No copy of `Card.Id` is face-up in the bot's spell zone |
| `UniqueFaceupMonster()` | No copy of `Card.Id` is face-up in the bot's monster zone |
| `DefaultDontChainMyself()` | Prevents the bot from chaining its own activation — only blocks Activate-type executors when the last chain player is also the bot |

### `DefaultMonsterSummon()`
- Level 1–4: always summon.
- Level 5+: only if `UniqueFaceupMonster()` (no duplicate on field) and all tribute monsters have lower defense power than this card's ATK.

### `DefaultField()`
Activates a field spell only when the bot's field zone (index 5 of spell zone) is empty.

### `DefaultMonsterRepos()`
Decides whether to change a monster's battle position. Priority order:
1. Invincible monsters always stay in Defense.
2. ATK-0 monsters flip to Defense if face-up in Attack.
3. Blue-Eyes Chaos MAX Dragon avoidance: if MAX Dragon is on the field, stays in the position that minimizes piercing damage.
4. All enemies stronger → switch to Defense; or switch back to Attack if this monster's ATK can compete.

---

## Design Patterns Used

- **Template Method**: `OnSelectAttackTarget` / `OnPreBattleBetween` form a template pair — subclasses override `OnPreBattleBetween` to inject ATK modifiers (e.g., Honest pre-battle boost) while the attack loop stays in the base.
- **Strategy via `AddExecutor`**: Each registered `(ExecutorType, cardId, handler)` triple is a strategy; priority is determined by registration order.
- **Guard Clause Chains**: Every `DefaultXxx` method is structured as a series of early-return `false` guards followed by a positive return, keeping logic flat and readable.
- **RealPower mutation**: Rather than creating new objects, `OnPreBattleBetween` mutates `attacker.RealPower` / `defender.RealPower` in-place so the outer loop can do a single numeric comparison.
- **Shared constants via `_CardId`**: Centralizes all numeric IDs in one place, making it easy to audit which cards have built-in behavior without searching call sites.
