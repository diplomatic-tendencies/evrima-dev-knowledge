# KillFeed design

> Bot-side specifics in this document (table schemas, the "bonecoin" naming, the 0.5 per kg payout rate, the 20-second combat window) reflect one example production integration. Treat them as illustrative, not as universal requirements.

KillFeed emits one event per player kill to a tailable NDJSON file for downstream bot consumption (typically a bonecoin-style economy that pays a player per kg of dino killed). This document covers the architecture, the hook-source research that informed the design, and the upgrade path from a pure-Lua implementation to a UE4SS C++ side mod for the version that catches DoT and environmental kills.

## Final spec

| Rule | Behavior |
|---|---|
| Event payload | `{killer_steam, victim_weight_kg, ts, idempotencyKey}`. No species, no killer-species. |
| Conversion (bot-side) | Configurable. Typical: 0.5 coin per kg. |
| Combat timer | 20 seconds from the last direct hit by any player. |
| Direct hit from a player | Resets the timer, updates the cached killer. |
| DoT tick (bleed, poison, etc.) | Does NOT reset the timer; damage applies, attribution unchanged. |
| Environmental (fall, drown, starve) | Does NOT reset the timer. |
| Death within 20s of last direct hit | Last direct hitter gets payout (regardless of actual cause). |
| Death after 20s of last direct hit | No payout. |
| Suicide with no preceding hit | No payout. |
| Cannibalism | Counts. |
| Multi-attacker | Last direct hit wins; no payout split. |
| Baby or juvenile | Payout at `GetWeight()` at moment of death; no minimum floor. |
| Offline RCON kills | No payout. |

## Hook source research

EVRIMA's damage and death pipeline is non-obvious. The relevant research, condensed:

Damage flow goes through Unreal's Gameplay Ability System (GAS). The hierarchy:

1. Attacker calls `/Script/TheIsle.TICharacterBase:ApplyDamage(Target, ...)` (a UFunction).
2. ApplyDamage applies a gameplay effect modifying the `Damage` attribute on the target's `UTIAttributeSetBase`.
3. The AttributeSet's `PostGameplayEffectExecute` (a C++ virtual on `UAttributeSet`, NOT a UFunction) processes the effect and decreases the `Health` attribute.
4. When Health reaches zero, the AttributeSet triggers death state.

Out of 20+ candidate hooks tested for "fires on damage" or "fires on death," exactly two fire reliably:

- `ApplyDamage(attacker)` is the attacker-side combat damage UFunction. Fires on every player-initiated direct attack. Does NOT fire on DoT ticks or environmental damage. This is the right attribution hook.

- `PlayerStats` snapshot stream (an indirect "hook" via tailing the events.ndjson file from PlayerStats mod). Watch for HP transition from >0 to 0 per steam. Worst-case 5-second lag from the actual death; acceptable for a kill feed.

Several hooks that look obvious but DON'T fire:

- `OnDeath` (BIE on TICharacterBase): script hook only, never fires.
- `OnPawnDeath` on the player controller: never fires.
- `UpdateCharacterCooldownOnDeath`: never fires.
- `SetHealth`: only fires when explicit C++ setter is called; not on natural death.
- `SetIsAlive`, `ToggleServerRagdoll`, `WaitAndDestroyCorpse`: don't fire on natural death.
- `UpdateBleedingEffect`, `ServerUpdateBleedingEffect`: not triggered during environmental damage.

The bee-damage test was the definitive case. A player took 60 seconds of bee damage in a swarm zone, HP draining about 185 per second from 9350 down to zero. Every candidate damage and death hook was instrumented. Zero fires across all hooks during the entire damage sequence. The death happened, the player respawned, no hook ever ran.

The conclusion: EVRIMA's damage pipeline (environmental and AI alike) bypasses every UFunction reachable from UE4SS Lua reflection. Death detection must use POLLING. There is no server-side UFunction death event.

## Lua v1 architecture

The Lua v1 implementation uses the two confirmed sources:

```
ApplyDamage hook (attacker, target, weight)
    -> damage cache: { victim_addr -> {attacker_steam, ts, victim_weight} }

PlayerStats events.ndjson tail
    -> on HP=0 transition per steam:
       - look up victim_addr in cache
       - if cache hit AND (now - cache.ts) <= 20s:
         emit kill event to events.ndjson
```

The damage cache has a 30-second TTL (longer than the 20-second combat window so attribution isn't lost to a marginal late death). Cache entries are keyed by victim's UObject address; resolve the victim to a stable identifier before the entry expires.

Death detection runs either:
- By tailing PlayerStats events.ndjson and reading the snapshot's `health` field per player, or
- By the mod running its own per-pawn Health attribute poll on a 1 to 2-second interval.

The PlayerStats tail is simpler (PlayerStats is already running for other purposes) but has worst-case 5-second lag. A dedicated 1-second poll is tighter but adds work.

## What Lua v1 misses

Three categories of kills go undetected:

1. **DoT-only kills.** Player A inflicts a bleed on Player B, walks away, B dies of bleed 30 seconds later with no direct hits in the cache. The death is detected by the polling layer, but there's no attribution; no event emits. Per spec, this is intentional ("direct-hit-within-20s only counts") but it's worth flagging because the spec's intent and the implementation's intent should match.

2. **Late deaths from direct hits.** Player A bites Player B (cache entry: attacker=A, ts=T0). B runs away and dies 30 seconds later from a different cause. The cache TTL is 30 seconds so the entry might still exist, but the 20-second combat window has expired so no event emits. Correct per spec.

3. **Suicide attribution.** Player jumps off a cliff. No `ApplyDamage` fire occurred. No cache entry. Death is detected by polling, no event emits. Correct per spec.

These are all acceptable for a simple kill-counting payout system. They are NOT acceptable for a deeper PvP analytics system that wants to track every kill including DoT chains. For that case, you need the C++ side-mod approach below.

## C++ side-mod upgrade path

For a v2 or premium version, build as a UE4SS C++ side mod that hooks `PostGameplayEffectExecute` directly. This is the C++ virtual on `UAttributeSet` that processes every gameplay effect modifying Health. It fires for direct damage, DoT damage, and environmental damage (all of which create gameplay effects modifying the Health attribute on the target's AttributeSet).

Architecturally:

```
PostGameplayEffectExecute hook (intercepts at the C++ vtable level)
    -> read FGameplayEffectModCallbackData (data.EvaluatedData.Attribute,
       data.EvaluatedData.Magnitude, data.Target, data.EffectSpec.GetContext())
    -> filter: only fire on Health attribute changes
    -> extract instigator from data.EffectSpec.GetContext()
    -> on Health crossing 0:
       emit kill event with full attribution
```

The benefits over Lua v1:

- No polling lag (synchronous hook on every damage event)
- Catches DoT kills (every DoT tick creates a gameplay effect that hits the hook)
- Catches environmental kills (fall, drown, etc. all go through the gameplay-effect path)
- Direct access to instigator info from the gameplay effect context (no separate ApplyDamage cache needed)

The cost: a UE4SS C++ side mod requires the C++ toolchain (`EVRIMA_CPP_Side_Mods.md` covers setup). The C++ virtual hook itself requires either signature scanning or careful vtable slot identification.

## NDJSON output format

```json
{"ts":1716508800,"killer_steam":"76561198XXXXXXX","victim_steam":"76561198YYYYYYY","victim_weight_kg":4500,"idempotencyKey":"76561198XXXXXXX:0x1cf81ff7810:1716508800"}
```

Fields:
- `ts`: unix timestamp of the kill emit.
- `killer_steam`: steam ID of the player who landed the last direct hit.
- `victim_steam`: steam ID of the victim. May be empty for AI-on-AI kills (the spec excludes payout for those anyway).
- `victim_weight_kg`: kg value from `GetWeight()` at the moment of the damage hit (NOT at moment of death, since the dead pawn's weight may be stale).
- `idempotencyKey`: deterministic key from `<killer>:<victim_addr>:<floor(ts)>`. Prevents double-emission if the poll loop sees the same HP=0 twice within a second.

## Bot-side handler

The bot's responsibilities on receiving a kill event:

1. Parse the line, validate the idempotency key.
2. Look up the killer's current bonecoin balance in the database.
3. Multiply `victim_weight_kg` by the configured rate (typical 0.5).
4. Increment the balance.
5. Write an audit row to a `bonecoin_transactions` table (kill amount, source event ID, timestamp).
6. Skip if the idempotency key was already processed (prevents double-credit on bot restart re-replay).

Schema additions for a typical bot:

```sql
ALTER TABLE profiles ADD COLUMN bonecoin_balance DECIMAL(10,2) DEFAULT 0;
CREATE TABLE bonecoin_transactions (
    id SERIAL PRIMARY KEY,
    steam VARCHAR(20) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    source_type VARCHAR(20),  -- 'kill' for this case
    source_key VARCHAR(80),   -- the idempotency key
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(source_key)        -- idempotency guard at the DB layer
);
```

The unique constraint on `source_key` prevents double-credit even if the bot's in-memory idempotency check fails (e.g. after a restart with partial state).

## Build estimate

Lua v1: about 6 hours mod-side, 1-2 hours bot-side, 30 minutes for the DB migration. About a day end-to-end.

C++ v2: about 1-2 days once the C++ toolchain is set up (see `EVRIMA_CPP_Side_Mods.md`). Most of the work is finding the right hook address; the actual hook logic is short. The toolchain bring-up itself is another half-day if you've never built a UE4SS C++ side mod before.

## Closing notes

The narrow spec ("direct-hit-within-20s only counts") was the key design decision. Without it, the kill feed would need to handle DoT chains, environmental deaths, and AI predation, all of which are not cleanly attributable from a pure-Lua mod. The narrow spec keeps Lua v1 feasible and ships in a day.

The C++ v2 upgrade path is real if broader attribution is needed. It's not necessary for the narrow spec.
