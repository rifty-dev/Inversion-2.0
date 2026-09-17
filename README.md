# Division Rod / Inversion Rod — Paper 1.21.11

Two custom fishing rods with a two-step "chain link then swap" mechanic.

## Building

```bash
unzip SwapRod-source.zip
cd divisionrod
mvn package
```

Output: `target/SwapRod.jar`. Needs JDK 21 + Maven. The pom points at the PaperMC
repo and `paper-api:1.21.11-R0.1-SNAPSHOT` (scope `provided`).

Drop the jar in `plugins/`, start once to generate `plugins/SwapRod/config.yml`.

## Command

`/swaprod Division [player]` · `/swaprod Inversion [player]` · `/swaprod reload`

Alias `/srod`. Permissions `swaprod.give` and `swaprod.reload`, both op by default.

## Behaviour

**No chat output at all.** Lock, swap, and every failure case are silent — sound and
particles only.

**No cooldown.** The only timing guard is a 150 ms debounce that swallows the duplicate
right-click event some client/server combos fire, so one physical click cannot both lock
and swap. Real consecutive clicks are always accepted. Links never expire either
(`link-timeout-seconds: 0`).

**Vanilla removal.** `PlayerInteractEvent` cancelled with `setUseItemInHand(DENY)`, plus
`PlayerFishEvent` cancelled at LOWEST with the hook removed as a fallback. No bobber.

**Step 1 — lock in.** Crosshair raycast from the eye location (`rayTraceEntities`, 0.35 ray
size). The block raycast runs first and shortens the ray, so you cannot lock through a wall
(`require-line-of-sight: false` to disable). Any entity except yourself, dead entities and
spectators. On success the player→target UUID pair is stored and the lock sound plays at
both locations.

**Step 2 — swap.** Second click checks the target still exists, is in the same world and is
within range, then exchanges both full `Location`s — x/y/z *and* yaw/pitch, so camera angles
swap exactly. Vehicles are dismounted first, otherwise the teleport silently fails. Link is
cleared afterwards, and also cleared on failure, death or quit.

**Wind charge combo.** The link lives in a `Map<UUID, Link>` on the plugin, never on the
item, so hotbar switching cannot disturb it. Landing further than the rod's range from the
target makes the second click fail and breaks the link.

Fall distance carries across the swap by default, so whoever you yank into your wind-charge
arc inherits the drop. `reset-fall-distance: true` if that's too harsh.

## Division vs Inversion

| | Division | Inversion |
|---|---|---|
| Range | 15 | 20 |
| Lock sound | `block.chain.place` | `block.respawn_anchor.charge` @ pitch 0.6 |
| Swap sound | `block.chain.break` | `block.respawn_anchor.deplete` @ pitch 0.6 |
| Particles | none | sculk soul + shriek + red dust |
| Nausea | — | 3s, on the **swapped entity** only |
| Texture | vanilla | custom, with a separate locked state |

Both: white non-italic name, no lore, no flags, Unbreaking III only.

## The Inversion texture

Your two textures are already in the pack:

```
assets/divisionrod/textures/item/inversion_rod.png          idle
assets/divisionrod/textures/item/inversion_rod_locked.png   locked (line out)
```

The locked swap is driven by the plugin, not by the vanilla `fishing_rod/cast` property —
that property only flips when a real bobber exists, and we never spawn one. Instead, on a
successful lock the plugin rewrites the `minecraft:item_model` component on every copy of
that rod in the player's inventory to `divisionrod:inversion_rod_locked`, and puts it back
on swap, failure, death or quit. Scanning the whole inventory rather than the held slot
means the texture survives hotbar juggling mid-combo.

Chain: `item-model` in config.yml → the `minecraft:item_model` component →
`assets/divisionrod/items/<name>.json` → `assets/divisionrod/models/item/<name>.json`
(parent `minecraft:item/handheld_rod`) → your PNG.

To ship: keep `pack.mcmeta` and `assets/` at the **top level** of the zip, host it with a
direct download URL, then set `resource-pack` and `resource-pack-sha1` in `server.properties`.

`pack.mcmeta` declares format 75 (the 1.21.11 client format), supported range down to 46.

## Notes

- Nausea only applies to `LivingEntity`, so boats and armour stands just teleport.
- If a player logs out mid-lock the texture resets on quit; if you set a non-zero
  `link-timeout-seconds`, an expired link will leave the locked texture until the next click.
- More variants: copy a block under `rods:`. The key becomes the `/swaprod <id>` argument.
  No code changes needed.
