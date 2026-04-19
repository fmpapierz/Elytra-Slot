# Elytraslot Multiloader — Handoff Document

**Project**: `C:\Users\warwa\ModDev\Elytra Slot UPdate FR 26.1\elytraslot-multiloader\`
**Target MC**: 26.1.2 | **NeoForge**: 26.1.2.1-beta | **Fabric loader**: 0.18.6 / API: 0.145.3+26.1.1 | **Java**: 25
**Mod version**: `2.0.0`
**Last update**: 2026-04-19

Ported from the canonical NeoForge 1.21.11 reference at
`..\.claude\worktrees\naughty-herschel\MDK-1.21.11-ModDevGradle-main\`
(that folder may have been deleted; the `.claude/worktrees/naughty-herschel` worktree
remains available via git).

Companion docs:
- **`VANILLA_PARITY_AUDIT.md`** — deep audit of every deviation from vanilla, with status per item.
- **`RESEARCH.md`** — exact bytecode/API references (vanilla method sigs, Fabric/NeoForge networking patterns, `moveItemStackTo` semantics, etc.) used by the fixes.
- **`CHANGELOG.md`** — chronological log of changes in this branch.

---

## Status: all audited deviations FIXED (except three deliberate scope carve-outs)

| Deviation group | Status |
|---|---|
| A1–A3 (flight) | ✅ Fixed |
| B1 / B3 (equip path) | ✅ Fixed |
| C1 (storage) | ✅ Intentional (platform-agnostic, not a bug) |
| D1–D7 (death/drop/respawn) | ✅ Fixed (D7 reorder is cosmetic and not worth the injection complexity) |
| E1–E5 (rendering + panel art) | ✅ Fixed |
| F (remote-player sync) | ✅ Fixed (custom S2C payload, both loaders) |
| G1 (equip pipeline — attributes + enchant effects) | ✅ Fixed |
| G2 (`Equippable.swappable()` honored) | ✅ Fixed |
| UI2 (Curse of Binding → `mayPickup`) | ✅ Fixed |
| UI3 / UI4 / UI5 (shift-click parity + stack merge + mayPlace) | ✅ Fixed |
| SE2 (server-side validation) | ✅ Fixed (folded into UI3–UI5 via `moveItemStackTo`) |
| C2 (`INVENTORY_CHANGED` advancement trigger) | ✅ Fixed |
| C1 (Curios/Trinkets conflict) | ⚠️ Detection + WARN only. Deep integration deferred — see `VANILLA_PARITY_AUDIT.md` section 3.7 |
| F1 (non-Player mobs gliding) | ⚠️ Intentional scope limit |
| D7 (death-drop order) | ⚠️ Cosmetic, no observable impact |

Explicit `[elytraslot] ...` INFO log lines are wired at every server-side and client-side code path so a live run can be verified end-to-end. See "Debug logging" below.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Player (server side)                          │
│                                                                         │
│  @Unique ElytraSlotContainer  ← PlayerElytraStorageMixin                │
│  addAdditionalSaveData         ← persistence (key "elytraslot_item")    │
│  readAdditionalSaveData        ← load silent (no equip sound)           │
│  dropEquipment (TAIL inject)   ← death: keepInventory + curse-vanishing │
│                                                                         │
│  ServerPlayerCloneMixin.restoreFrom ← respawn carry-over                │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼ container.setItem on any interactive write
┌─────────────────────────────────────────────────────────────────────────┐
│  ElytraEquipEffects.onSlotChanged(player, oldStack, newStack)           │
│                                                                         │
│  Step 1: sound + GameEvent.EQUIP/UNEQUIP      (mirror onEquipItem)      │
│  Step 2: stop-OLD attribute modifiers + location-based enchant effects  │
│  Step 3: apply-NEW modifiers + runLocationChangedEffects                │
│  Step 4: CriteriaTriggers.INVENTORY_CHANGED   (C2 advancement trigger)  │
│  Step 5: ElytraSyncDispatcher.broadcastSlotChange                       │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ElytraSlotSyncPayload (UUID, ItemStack) — S2C                          │
│                                                                         │
│  Fabric:    PayloadTypeRegistry.clientboundPlay + ServerPlayNetworking  │
│             .send for each PlayerLookup.tracking(player) + self         │
│  NeoForge:  PayloadRegistrar.playToClient +                             │
│             PacketDistributor.sendToPlayersTrackingEntityAndSelf        │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Client receiver (Fabric or NeoForge) → ElytraSlotSyncClient.handle     │
│                                                                         │
│  Look up target Player by UUID in client level, write stack silently    │
│  via container.setItemSilent(0, stack).                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Rendering pipeline (every client with this Player tracked)             │
│                                                                         │
│  HumanoidMobRenderer.extractHumanoidRenderState (static)                │
│    → HumanoidRenderStateMixin.onExtractHumanoidRenderState              │
│    → state.elytraslot_setAccessoriesElytra(container.getItem(0).copy()) │
│                                                                         │
│  WingsLayer.submit @At("RETURN")  (fires at BOTH return points)         │
│    → ElytraLayerMixin.onSubmit                                          │
│    → guard cascade (holder-empty / chest-already-elytra /               │
│       equippable-missing)                                               │
│    → equipmentRenderer.renderLayers(WINGS, assetId, model, ..., 0)      │
│                                                                         │
│  CapeLayer.submit @At("HEAD", cancellable=true)                         │
│    → CapeLayerMixin — ci.cancel() when holder has elytra                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Flight pipeline

```
LivingEntity.canGlide              ← ElytraSlotMixin.onCanGlide
  guarded on instanceof Player     ← non-player mobs fall through (F1)
  → return true if custom-slot has GLIDER-component item

LivingEntity.updateFallFlying       ← ElytraSlotMixin.onUpdateFallFlying
  iterate EquipmentSlot.VALUES in vanilla order;
  substitute custom-slot at CHEST position if CHEST is empty (A2).
  hurtAndBreak() with onBreak consumer:
    snapshot pre-break stack
    entityEvent(50) — CHEST break VFX
    strip attribute modifiers
    stopLocationBasedEffects(preBreakSnapshot, player, CHEST) (A1)
```

### Inventory UI

```
Player.InventoryMenu.<init>   ← InventoryMenuMixin.elytraslot$addElytraSlot
  menu.addSlot(new Slot(container, 0, -25, 8) {
    mayPlace:   isElytraLike(stack) && chest slot not elytra-like
    mayPickup:  !creative && PREVENT_ARMOR_CHANGE on stack → DENY  (UI2, Binding)
    noItemIcon: elytraslot:container/slot/elytra
  })
  // slot lands at index = slots.size()-1  (46 in vanilla)

InventoryMenu.quickMoveStack  ← InventoryMenuQuickMoveMixin.elytraslot$quickMove
  Case 1 (source == elytra slot):
    moveItemStackTo(stack, 9, 45, false)    ← main inv first, hotbar last
    onSlotChanged(player, pre, post)        ← fire UNEQUIP side effects
  Case 2 (source != elytra slot, stack isElytraLike):
    moveItemStackTo(stack, elytraIdx, elytraIdx+1, false)
      ← pass 1 merges; pass 2 honors mayPlace (SE2)

Equippable.swapWithEquipmentSlot  ← ElytraEquipMixin.onSwapWithEquipmentSlot (HEAD cancellable)
  gates: isElytraLike + Equippable.slot==CHEST + Equippable.swappable (G2)
  mirror of vanilla swap (single-item vs stacked branches)
```

### Inventory panel (custom render)

`InventoryScreenBgMixin.extractBackground` (TAIL) procedurally composites a 32×32 panel
from vanilla `textures/gui/container/inventory.png`:

- 3×3 corner blits (TL/TR/BL/BR) — carries vanilla's rounded-corner art.
- 3-px edge blits (top/bottom/left/right).
- Solid `0xFFC6C6C6` center fill.
- **Two 1×1 bulge-pixel blits** (E5 fix):
  - Panel `(3, 3)` ← vanilla UV `(3, 3)` — white highlight.
  - Panel `(28, 28)` ← vanilla UV `(172, 162)` — dark-gray shadow.
- 18×18 slot recess blit from vanilla UV `(7, 7)` (helmet-slot region).

The `inventory_panel.png` asset in `common/src/main/resources/assets/elytraslot/textures/gui/`
is **not used** by the current render — it was an earlier bitmap approach and is now vestigial.

---

## Deviation table — all entries

All historical deviations collected. Status column matches the "Status" summary above.

### A. Flight (`ElytraSlotMixin`)

| # | Deviation | Status |
|---|---|---|
| A1 | Break path wasn't calling `stopLocationBasedEffects` on the elytra's own enchantments. | ✅ fixed — `onBreak` snapshots pre-break stack, inlines vanilla's `stopLocationBasedEffects` (forEachModifier removeModifier + `EnchantmentHelper.stopLocationBasedEffects`). |
| A2 | Damage-picker list appended custom slot AFTER `EquipmentSlot.VALUES`, biasing order. | ✅ fixed — iterate `VALUES` in vanilla order, substitute at CHEST position. |
| A3 | `canGlideUsing(elytra, CHEST)` requires `Equippable.slot()==CHEST`. | ✅ intentional — every vanilla glider is CHEST-slotted. |

### B. Equip path (`ElytraEquipMixin` + `ElytraEquipEffects`)

| # | Deviation | Status |
|---|---|---|
| B1 | Intercepted any elytra-like regardless of `Equippable.slot()`. | ✅ fixed — gate on `slot()==CHEST`. |
| B3 | Missing vanilla `!firstTick` guard in `onSlotChanged`. | ✅ fixed — `LivingEntityAccessor` with `@Accessor("firstTick")`. |

### C. Storage (architectural)

| # | Deviation | Status |
|---|---|---|
| C1 | Per-player NBT via `addAdditionalSaveData` rather than a platform attachment. | ✅ intentional — platform-agnostic. |

### D. Death / drop / respawn

| # | Deviation | Status |
|---|---|---|
| D1 | `dropOnDeath` ignored `KEEP_INVENTORY`. | ✅ fixed — early return when gamerule is set. |
| D2 | Curse of Vanishing (`PREVENT_EQUIPMENT_DROP`) ignored. | ✅ fixed — destroy via `setItemSilent(0, EMPTY)` without drop. |
| D3 | `ServerPlayerCloneMixin.carryOver` always transferred on clone. | ✅ fixed — match vanilla: `restoreAll \|\| isSpectator \|\| KEEP_INVENTORY`. |
| D4 | `player.drop(stack, true, false)` signature matches vanilla `Inventory.dropAll`. | ✅ parity. |
| D5 | `@At("TAIL")` on `Player.dropEquipment`. | ✅ parity — runs after vanilla inventory drops, same net result. |
| D6 | Silent-set for curse-vanishing vs vanilla's `removeItemNoUpdate` — both produce no sound. | ✅ parity. |
| D7 | Drop order: our elytra drops AFTER vanilla inventory. | ⚠️ cosmetic — no observable difference. |

### E. Rendering

| # | Deviation | Status |
|---|---|---|
| E1 | `compatibilityLevel: JAVA_25` in mixins.json. | ✅ fixed — lowered to `JAVA_21`. |
| E2 | Cape was not hidden when custom-slot elytra was worn. | ✅ fixed — `CapeLayerMixin` HEAD cancel. |
| E3 | Render passed `null` skin texture and `DYED_COLOR.rgb()` tint. | ✅ fixed — inlined vanilla's `getPlayerElytraTexture`, passes `0` for tint. |
| E4 | `@At("TAIL")` missed vanilla's early-return branch. | ✅ fixed — `@At("RETURN")` catches every return opcode. |
| E5 | Panel center-fill overwrote vanilla's two bevel-bulge pixels. | ✅ fixed — two 1×1 blits after the center fill. |

### F. Remote-player sync

| # | Deviation | Status |
|---|---|---|
| F | Vanilla `ClientboundSetEquipmentPacket` iterates only `EquipmentSlot.VALUES` — our custom slot is invisible to remote clients. | ✅ fixed — `ElytraSlotSyncPayload` broadcast from `ElytraEquipEffects.onSlotChanged`. Both loaders wired. |
| F1 | Only Player gets the custom-slot pipeline; non-player mobs fall through. | ⚠️ intentional scope limit. |

### G. Equip pipeline (post-audit)

| # | Deviation | Status |
|---|---|---|
| G1 | `onSlotChanged` only fired sound + GameEvent — vanilla's tick diff path (`collectEquipmentChanges`) also applies attribute modifiers and runs location-based enchantment effects. | ✅ fixed — `ElytraEquipEffects.onSlotChanged` now mirrors both pipelines server-side. |
| G2 | `Equippable.swappable()` flag ignored. | ✅ fixed — return early if `swappable()==false`. |

### UI (post-audit)

| # | Deviation | Status |
|---|---|---|
| UI1 | `mayPlace` blocks non-elytra items and blocks when vanilla chest already has elytra. | ✅ parity-extension. |
| UI2 | Curse of Binding not honored. | ✅ fixed — `Slot.mayPickup` override checks `EnchantmentEffectComponents.PREVENT_ARMOR_CHANGE`. |
| UI3 | Shift-click OUT didn't mirror vanilla's hotbar-preference order. | ✅ fixed — uses `moveItemStackTo(9, 45, false)` matching vanilla armor-slot behavior. |
| UI4 | Shift-click didn't stack onto existing matching items. | ✅ fixed — `moveItemStackTo` pass-1 merges. |
| UI5 | Shift-click IN bypassed `mayPlace` server-side. | ✅ fixed — `moveItemStackTo` pass-2 always calls `mayPlace`. |
| UI6 | `hasClickedOutside` extends click region to the panel (prevents accidental drops). | ✅ parity-extension. |
| EX1 | `ArmorSlot.mayPlace` intercept blocks chest-equip while custom slot has elytra. | ✅ parity-extension. |

### Security

| # | Deviation | Status |
|---|---|---|
| SE1 | Shift-click uses `stack.copy()` twice — server-authoritative sync protects. | ✅ parity. |
| SE2 | Server-side validation on shift-click / drag. | ✅ fixed — all paths go through `moveItemStackTo` which calls `mayPlace`. |
| SE3 | NBT malformation falls back to `Optional.empty` — no crash. | ✅ parity-safe. |

### Compatibility

| # | Deviation | Status |
|---|---|---|
| C1 | Curios / Trinkets potential conflict (double render / damage / modifier). | ⚠️ detection + WARN only. Deep integration would require ~42h of engineering — see `VANILLA_PARITY_AUDIT.md` section 3.7. |
| C2 | Advancement trigger for wearing elytra not fired. | ✅ fixed — `CriteriaTriggers.INVENTORY_CHANGED.trigger(sp, sp.getInventory(), newStack)` in `onSlotChanged`. |
| C3 | `detectEquipmentUpdates` iterates `EquipmentSlot.VALUES` only. | N/A — folded into G1. |

---

## Debug logging (runtime verification)

Core logger: `com.warwa.elytraslot.ElytraSlotConstants.LOGGER`. Every log line is tagged `[elytraslot]`. No debug logging was added — only INFO and WARN.

### Verifiable log lines by subsystem

**Startup (both loaders):**
```
[elytraslot] Trinkets not detected — running standalone         (Fabric)
[elytraslot] Curios not detected — running standalone           (NeoForge)
[elytraslot] Fabric clientboundPlay payload type registered: elytraslot:slot_sync
[elytraslot] ElytraSyncDispatcher registered: com.warwa...Fabric$$Lambda/...
[elytraslot] Fabric S2C receiver registered for elytraslot:slot_sync
[elytraslot] elytraSlot added to InventoryMenu at index=46 player=…
```

**Equip cycle (server-side):**
```
[elytraslot] onSlotChanged ENTRY old={old} new={new} firstTick=false silent=false creative=false
[elytraslot] onSlotChanged playing equip sound={sound}
  (or)  onSlotChanged equip-sound skipped silent=… equippable-null=… slot-mismatch=…
[elytraslot] onSlotChanged gameEvent={EQUIP|UNEQUIP}
[elytraslot] onSlotChanged stopped OLD modifiers count=N stack={old}
  (or)  onSlotChanged OLD stack empty — skipped stop phase
[elytraslot] onSlotChanged stopLocationBasedEffects OLD stack={old}
[elytraslot] onSlotChanged applied NEW modifiers count=N stack={new}
  (or)  onSlotChanged NEW stack empty/broken — skipped apply phase
[elytraslot] onSlotChanged runLocationChangedEffects NEW stack={new}
[elytraslot] onSlotChanged firing CriteriaTriggers.INVENTORY_CHANGED for {name}
[elytraslot] onSlotChanged broadcasting sync payload to trackers of {name}
[elytraslot] FabricDispatcher.broadcastSlotChange player={name} stack={s} sent={N}
  (or)  NeoForgeDispatcher.broadcastSlotChange player={name} stack={s}
[elytraslot] onSlotChanged EXIT
```

**Remote sync (client-side):**
```
[elytraslot] Fabric S2C receive playerId={uuid} stack={s}         (Fabric)
[elytraslot] NeoForge S2C receive playerId={uuid} stack={s}        (NeoForge)
[elytraslot] SyncClient.handle writing stack={s} into player={name} containerId={hex}
```

**Shift-click (UI3/UI4/UI5):**
```
[elytraslot] quickMove OUT beforeMove={stack} player={name}
[elytraslot] quickMove OUT SUCCESS pre={pre} remainingInSource={rem}
[elytraslot] quickMove IN beforeMove={stack} elytraIdx={i} srcIdx={s}
[elytraslot] quickMove IN SUCCESS pre={pre} remainingInSource={rem}
```

**`mayPlace` / `mayPickup`:**
```
[elytraslot] elytraSlot.mayPlace DENY not-elytra-like stack={stack}
[elytraslot] elytraSlot.mayPlace DENY vanilla chest already has elytra chestItem={item}
[elytraslot] elytraSlot.mayPickup DENY binding-curse player={name} stack={s}
```

**`ElytraEquipMixin` (right-click swap):**
```
[elytraslot] ElytraEquipMixin.onSwap gate-trip swappable=false stack={s}
[elytraslot] swapWithEquipmentSlot inHand={s} existing={e} count={n} creative={b}
```

**Persistence:**
```
[elytraslot] loadData restored stack={s}
[elytraslot] saveData wrote stack={s}
```

**Death / respawn:**
```
[elytraslot] dropOnDeath dropping player={name} stack={s}
[elytraslot] dropOnDeath skipped (keepInventory) player={name} stack={s}
[elytraslot] dropOnDeath destroyed (vanishing curse) player={name} stack={s}
[elytraslot] updateFallFlying damage pick=GliderSource[…] sources=N player={name}
[elytraslot] custom-slot elytra broke player={name} preBreakStack={s}
```

**Container (if anything writes directly):** — no debug overrides remain; we removed the setItem/removeItem tracers during cleanup. If you need to re-enable, add `@Override` loggers in `ElytraSlotContainer`.

---

## Project layout

```
elytraslot-multiloader/
├── HANDOFF.md                       ← this file
├── VANILLA_PARITY_AUDIT.md          ← deviation audit with status
├── RESEARCH.md                      ← vanilla bytecode + loader API reference
├── CHANGELOG.md                     ← chronological change log
│
├── common/src/main/java/com/warwa/elytraslot/
│   ├── ElytraSlotConstants.java          Logger, mod id
│   ├── ElytraSlotContainer.java          1-slot SimpleContainer + setItemSilent
│   ├── ElytraSlotUtil.java               isElytraLike helper (GLIDER component check)
│   ├── ElytraEquipEffects.java           onSlotChanged server-side (G1)
│   ├── ElytraSlotSyncPayload.java        S2C payload record (F)
│   ├── ElytraSyncDispatcher.java         Loader-agnostic broadcast contract (F)
│   ├── GliderSource.java                 Record: (stack, slot, fromCustomSlot)
│   ├── IElytraHolder.java                Interface mixed into HumanoidRenderState
│   ├── IElytraSlotPlayer.java            Interface mixed into Player
│   └── client/
│       └── ElytraSlotSyncClient.java     Client-side S2C handler (F)
│
├── common/src/main/java/com/warwa/elytraslot/mixin/
│   ├── PlayerElytraStorageMixin.java     @Unique container + save/load/drop
│   ├── ServerPlayerCloneMixin.java       Respawn carry-over (D3)
│   ├── ElytraSlotMixin.java              canGlide + updateFallFlying hooks
│   ├── ElytraEquipMixin.java             swapWithEquipmentSlot intercept (B1/G2)
│   ├── ElytraExclusiveMixin.java         Block chest equip when custom has elytra
│   ├── InventoryMenuMixin.java           addSlot with mayPlace + mayPickup (UI2)
│   ├── InventoryMenuQuickMoveMixin.java  Shift-click (UI3/UI4/UI5/SE2)
│   ├── LivingEntityAccessor.java         @Accessor("firstTick")
│   └── client/
│       ├── CapeLayerMixin.java           Hide cape when holder has elytra (E2)
│       ├── ElytraLayerMixin.java         WingsLayer.submit @At("RETURN") (E4)
│       ├── HumanoidRenderStateMixin.java extractHumanoidRenderState TAIL inject
│       ├── HumanoidRenderStateAccessorMixin.java  @Unique field + IElytraHolder impl
│       └── InventoryScreenBgMixin.java   Procedural panel + E5 bulge pixels
│
├── common/src/main/resources/
│   ├── elytraslot-common.mixins.json     Mixin config (compatLevel JAVA_21)
│   ├── elytraslot.accesswidener          Access widener (for Slot.container, etc.)
│   ├── META-INF/accesstransformer.cfg    AT (NeoForge side)
│   └── assets/elytraslot/
│       ├── lang/en_us.json
│       └── textures/gui/sprites/container/slot/elytra.png  (empty-slot hint)
│
├── fabric/src/main/java/com/warwa/elytraslot/fabric/
│   ├── ElytraSlotModFabric.java          @ModInitializer — main init (C1 detect)
│   ├── ElytraSlotModFabricClient.java    @ClientModInitializer — S2C receiver
│   └── ElytraSlotNetworkFabric.java      Payload register + dispatcher (F)
│
├── fabric/src/main/resources/
│   └── fabric.mod.json                   main + client entrypoints
│
├── neoforge/src/main/java/com/warwa/elytraslot/neoforge/
│   ├── ElytraSlotModNeoForge.java        @Mod — main init (C1 detect)
│   └── ElytraSlotNetworkNeoForge.java    RegisterPayloadHandlersEvent + dispatcher (F)
│
└── neoforge/src/main/resources/
    └── META-INF/neoforge.mods.toml       Mod metadata
```

### Built artifacts

| Loader | Path |
|---|---|
| Fabric | `fabric/build/libs/elytraslot-fabric-26.1.2-2.0.0.jar` |
| NeoForge | `neoforge/build/libs/elytraslot-neoforge-26.1.2-2.0.0.jar` |

---

## Repository / command cheat sheet

```powershell
cd "elytraslot-multiloader"

# Fresh build both jars
.\gradlew clean :fabric:jar :neoforge:jar --no-configuration-cache

# Run dev clients
.\gradlew :fabric:runClient --no-configuration-cache
.\gradlew :neoforge:runClient --no-configuration-cache

# Verify a class in a built jar
unzip -p fabric/build/libs/elytraslot-fabric-26.1.2-2.0.0.jar `
   com/warwa/elytraslot/mixin/client/ElytraLayerMixin.class `
   | javap -c -p -

# Grep INFO-level mod logs from last run
Select-String -Path fabric/runs/client/logs/latest.log -Pattern '\[elytraslot\]'
Select-String -Path neoforge/run/logs/latest.log      -Pattern '\[elytraslot\]'

# Enable transformed-class dump for mixin debugging (add to fabric/build.gradle runs.client)
# vmArg('-Dmixin.debug.export=true')
# Output lands in fabric/runs/client/.mixin.out/class/
```

---

## Known scope carve-outs (not bugs)

See `VANILLA_PARITY_AUDIT.md` for in-depth justifications.

1. **F1 — Non-Player mobs gliding.** `@Unique ElytraSlotContainer` only exists on `Player`, not on generic `LivingEntity`. Supporting zombies/villagers gliding from our custom slot would require moving the container to `LivingEntity` (huge blast radius). Vanilla chest-slot gliding for non-players still works normally.

2. **D7 — Death-drop order.** Our elytra drops AFTER vanilla's `inventory.dropAll()` rather than interleaved with `destroyVanishingCursedItems`. Same items on the ground, same pickup behavior — only packet-emission order differs. Not worth the injection complexity.

3. **C1 — Curios (NeoForge) / Trinkets (Fabric) deep integration.** We detect and WARN at startup. Full interop (conditional deps, rendering mutual exclusion, damage coordination) is ~42h of work and brittle against upstream API changes. Most users don't pair both — the WARN makes it discoverable.

---

## What to do next (if picking this up cold)

The mod is feature-complete relative to the audit. Possible next directions:

1. **Modded-elytra regression test.** Install a mod that adds a glider with attribute modifiers or location-based enchantments; confirm G1 applies/removes them correctly via `[elytraslot] applied NEW modifiers count=N` log lines showing a non-zero count.
2. **Multiplayer sanity test.** Launch server + two clients; confirm one client sees the other's custom-slot elytra render (F fix) via `[elytraslot] S2C receive` lines on the non-owning client.
3. **Death-drop end-to-end.** Equip elytra in custom slot, `/kill`, confirm `dropOnDeath dropping` log. Repeat with `/gamerule keepInventory true` (expect `skipped (keepInventory)`). Repeat with a Curse-of-Vanishing elytra (expect `destroyed (vanishing curse)`).
4. **Polish `@Unique` name prefix.** All `@Unique` fields and methods should use the `elytraslot$` prefix for Mixin-best-practices compliance (reduces collision risk with other mods). Current code mostly does this; scan for stragglers.
5. **Runtime log noise.** The `[elytraslot] onSlotChanged EXIT` and per-stage logs are verbose. Consider gating behind a `-Delytraslot.debug=true` flag.
6. **Integration tests.** Spin up a Fabric dedicated server programmatically; drive a client; assert state. Heavy lift but catches F regressions.
7. **Curios/Trinkets optional deep integration.** See section 3.7 of the audit for the full checklist if someone demands it.
