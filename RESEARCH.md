# Elytra Slot — MC 26.1.2 Research Report

Sources:
- Vanilla client jar: `C:/Users/warwa/.gradle/caches/fabric-loom/26.1.2/minecraft-client.jar`
- Fabric networking: `fabric-networking-api-v1-6.3.0+50a808ceb3.jar` (ships with Fabric API 0.145.3+26.1.1)
- NeoForge: `neoforge-26.1.2.1-beta-universal.jar`

All method references in this doc use mojmap (mojang names), since the jar is already remapped.

---

## R1. `LivingEntity.collectEquipmentChanges` + equipment tick pipeline

### Method: `LivingEntity.tick()` invokes `detectEquipmentUpdates()`

```
// LivingEntity.detectEquipmentUpdates()
private void detectEquipmentUpdates() {
    Map<EquipmentSlot, ItemStack> changes = collectEquipmentChanges();
    if (changes != null) {
        handleHandSwap(changes);
        if (!changes.isEmpty()) {
            handleEquipmentChanges(changes);
        }
    }
}
```

### `LivingEntity.collectEquipmentChanges()` — JVM sig: `()Ljava/util/Map;` (private)

Exact reconstruction (from bytecode dump):

```java
private Map<EquipmentSlot, ItemStack> collectEquipmentChanges() {
    Map<EquipmentSlot, ItemStack> changed = null;

    // PASS 1 — detect change, deactivate OLD attributes/location effects for every slot
    for (EquipmentSlot slot : EquipmentSlot.VALUES) { // 8 slots: MAINHAND, OFFHAND, FEET, LEGS, CHEST, HEAD, BODY, SADDLE
        ItemStack oldStack = this.lastEquipmentItems.get(slot);
        ItemStack newStack = this.getItemBySlot(slot);
        if (this.equipmentHasChanged(oldStack, newStack)) {
            if (changed == null) changed = Maps.newEnumMap(EquipmentSlot.class);
            changed.put(slot, newStack);
            AttributeMap attrs = this.getAttributes();
            if (!oldStack.isEmpty()) {
                // Removes old item's attribute modifiers + stops old location-based enchantment effects
                this.stopLocationBasedEffects(oldStack, slot, attrs);
            }
        }
    }

    // PASS 2 — apply NEW attribute modifiers + run NEW location-based enchantment effects
    if (changed != null) {
        for (Map.Entry<EquipmentSlot, ItemStack> e : changed.entrySet()) {
            EquipmentSlot slot = e.getKey();
            ItemStack newStack = e.getValue();
            if (!newStack.isEmpty() && !newStack.isBroken()) {
                // Applies attribute modifiers (the lambda calls attributeInstance.removeModifier(id) then addTransientModifier(mod))
                newStack.forEachModifier(slot, (attrHolder, modifier) -> {
                    AttributeInstance inst = this.attributes.getInstance(attrHolder);
                    if (inst != null) {
                        inst.removeModifier(modifier.id());   // JVM: (Lnet/minecraft/resources/Identifier;)Z
                        inst.addTransientModifier(modifier);  // JVM: (Lnet/minecraft/world/entity/ai/attributes/AttributeModifier;)V
                    }
                });
                if (this.level() instanceof ServerLevel sl) {
                    EnchantmentHelper.runLocationChangedEffects(sl, newStack, this, slot);
                }
            }
        }
    }
    return changed;
}
```

### `LivingEntity.stopLocationBasedEffects(ItemStack, EquipmentSlot, AttributeMap)` — private

```java
private void stopLocationBasedEffects(ItemStack stack, EquipmentSlot slot, AttributeMap attrs) {
    stack.forEachModifier(slot, (attrHolder, modifier) -> {
        AttributeInstance inst = attrs.getInstance(attrHolder);
        if (inst != null) inst.removeModifier(modifier);  // by AttributeModifier (not by id)
    });
    EnchantmentHelper.stopLocationBasedEffects(stack, this, slot);
}
```

### `LivingEntity.handleEquipmentChanges(Map)` — private

```java
private void handleEquipmentChanges(Map<EquipmentSlot, ItemStack> changes) {
    List<Pair<EquipmentSlot, ItemStack>> broadcast = Lists.newArrayListWithCapacity(changes.size());
    changes.forEach((slot, stack) -> {
        ItemStack copy = stack.copy();
        broadcast.add(Pair.of(slot, copy));
        this.lastEquipmentItems.put(slot, copy);
    });
    ((ServerLevel)this.level()).getChunkSource()
        .sendToTrackingPlayers(this, new ClientboundSetEquipmentPacket(this.getId(), broadcast));
}
```

**IMPORTANT:** `handleEquipmentChanges` does NOT call `onEquipItem`. It only updates `lastEquipmentItems` and broadcasts to trackers. The `onEquipItem` hook (sound + GameEvent.EQUIP/UNEQUIP) fires only from `LivingEntity.setItemSlot(slot, stack)`, NOT from the tick-based diff path.

### `LivingEntity.setItemSlot(EquipmentSlot, ItemStack)` — public

```java
public void setItemSlot(EquipmentSlot slot, ItemStack stack) {
    ItemStack prev = this.equipment.set(slot, stack); // EntityEquipment.set returns the previous stack
    this.onEquipItem(slot, prev, stack);
}
```

### `LivingEntity.onEquipItem(EquipmentSlot, ItemStack from, ItemStack to)` — public

```java
public void onEquipItem(EquipmentSlot slot, ItemStack from, ItemStack to) {
    if (this.level().isClientSide() || this.isSpectator()) return;
    if (ItemStack.isSameItemSameComponents(from, to) || this.firstTick) return;
    Equippable eq = to.get(DataComponents.EQUIPPABLE);
    if (!this.isSilent() && eq != null && slot == eq.slot()) {
        this.level().playSeededSound(null, getX(), getY(), getZ(),
            this.getEquipSound(slot, to, eq), this.getSoundSource(), 1.0F, 1.0F, this.random.nextLong());
    }
    if (this.doesEmitEquipEvent(slot)) {
        this.gameEvent(eq != null ? GameEvent.EQUIP : GameEvent.UNEQUIP);
    }
}
```

### EXACT signatures used in the tick pipeline

- `ItemStack.forEachModifier(EquipmentSlot, BiConsumer<Holder<Attribute>, AttributeModifier>)` — JVM: `(Lnet/minecraft/world/entity/EquipmentSlot;Ljava/util/function/BiConsumer;)V`
- `AttributeInstance.removeModifier(Identifier)` — returns `boolean`; NEW signature in 26.1 takes `Identifier` (mojmap `ResourceLocation` → `Identifier`)
- `AttributeInstance.removeModifier(AttributeModifier)` — used by `stopLocationBasedEffects`
- `AttributeInstance.addTransientModifier(AttributeModifier)` — (no plural `AttributeMap.addTransientAttributeModifiers` usage here; it works per-instance)
- `EnchantmentHelper.runLocationChangedEffects(ServerLevel, ItemStack, LivingEntity, EquipmentSlot)` — static
- `EnchantmentHelper.stopLocationBasedEffects(ItemStack, LivingEntity, EquipmentSlot)` — static
- `ServerChunkCache.sendToTrackingPlayers(Entity, Packet<? super ClientGamePacketListener>)` — used by `handleEquipmentChanges`
- `ServerChunkCache.sendToTrackingPlayersAndSelf(Entity, Packet)` — alternative (for self-visibility)

**What this means for our fix:** To integrate a custom elytra slot into the vanilla equipment-attribute + enchantment pipeline, the mixin must (a) include our slot in the "change detection" pass and (b) replicate exactly the two-pass sequence: `stopLocationBasedEffects` → `forEachModifier` with transient add → `EnchantmentHelper.runLocationChangedEffects`. Sound + GameEvent live in `onEquipItem` and fire from `setItemSlot`; for our custom slot we must call `onEquipItem(slot, oldStack, newStack)` ourselves when we set the slot.

---

## R2. `InventoryMenu.quickMoveStack` — slot routing

Player-inventory slot layout (from bytecode constants):

| Const | Value |
|---|---|
| `CRAFT_SLOT_START` | 1 (inclusive) |
| `CRAFT_SLOT_END` | 5 (exclusive) |
| `ARMOR_SLOT_START` | 5 |
| `ARMOR_SLOT_END` | 9 |
| `INV_SLOT_START` | 9 |
| `INV_SLOT_END` | 36 |
| `USE_ROW_SLOT_START` | 36 |
| `USE_ROW_SLOT_END` | 45 |
| `SHIELD_SLOT` | 45 |

- Index 0 is the craft **result** slot.
- Armor slot index is computed as `8 - EquipmentSlot.getIndex()` → FEET idx=0 → slot 8, LEGS idx=1 → slot 7, CHEST idx=2 → slot 6, HEAD idx=3 → slot 5.

Reconstructed `quickMoveStack`:

```java
public ItemStack quickMoveStack(Player player, int index) {
    ItemStack returned = ItemStack.EMPTY;
    Slot src = this.slots.get(index);
    if (!src.hasItem()) return ItemStack.EMPTY; // (returns EMPTY via fallthrough at end)
    ItemStack fromSlot = src.getItem();
    returned = fromSlot.copy();
    EquipmentSlot prefEq = player.getEquipmentSlotForItem(fromSlot);

    if (index == 0) {                                         // CRAFT RESULT (slot 0)
        if (!this.moveItemStackTo(fromSlot, 9, 45, true)) return ItemStack.EMPTY;
        src.onQuickCraft(fromSlot, returned);
    } else if (index >= 1 && index < 5) {                     // CRAFT GRID (1-4)
        if (!this.moveItemStackTo(fromSlot, 9, 45, false)) return ItemStack.EMPTY;
    } else if (index >= 5 && index < 9) {                     // ARMOR (5-8)
        if (!this.moveItemStackTo(fromSlot, 9, 45, false)) return ItemStack.EMPTY;
    } else if (prefEq.getType() == EquipmentSlot.Type.HUMANOID_ARMOR
               && !this.slots.get(8 - prefEq.getIndex()).hasItem()) {
        // from inventory/hotbar, item is armor and matching armor slot is empty
        int armorIdx = 8 - prefEq.getIndex();
        if (!this.moveItemStackTo(fromSlot, armorIdx, armorIdx + 1, false)) return ItemStack.EMPTY;
    } else if (prefEq == EquipmentSlot.OFFHAND && !this.slots.get(45).hasItem()) {
        if (!this.moveItemStackTo(fromSlot, 45, 46, false)) return ItemStack.EMPTY;
    } else if (index >= 9 && index < 36) {                    // MAIN INVENTORY → hotbar
        if (!this.moveItemStackTo(fromSlot, 36, 45, false)) return ItemStack.EMPTY;
    } else if (index >= 36 && index < 45) {                   // HOTBAR → main inventory
        if (!this.moveItemStackTo(fromSlot, 9, 36, false)) return ItemStack.EMPTY;
    } else {                                                  // fallback (e.g. SHIELD slot index 45)
        if (!this.moveItemStackTo(fromSlot, 9, 45, false)) return ItemStack.EMPTY;
    }

    if (fromSlot.isEmpty()) src.setByPlayer(ItemStack.EMPTY, returned);
    else src.setChanged();
    if (fromSlot.getCount() == returned.getCount()) return ItemStack.EMPTY;
    src.onTake(player, fromSlot);
    if (index == 0) player.drop(fromSlot, false);
    return returned;
}
```

Key observations:
- `reverseDirection=true` is used only for moving *out of* the craft-result slot (slot 0) — it places into the hotbar first (high→low).
- The "stack onto matching first, then empty" logic lives in `moveItemStackTo`, NOT in `quickMoveStack`.
- The armor-auto-route branch (`HUMANOID_ARMOR`) uses a 1-slot range `[armorIdx, armorIdx+1)`. This means if we add an elytra slot at, say, index 46, vanilla quickMove will NOT auto-route elytras into it — we need to intercept before the `prefEq==OFFHAND` branch or the final fallback.
- `Slot.mayPlace` IS checked server-side inside `moveItemStackTo`'s PASS 2 (see R3 bytecode offset 268).

**What this means for our fix:** `InventoryMenuQuickMoveMixin` must inject an auto-route branch for elytras BEFORE the fallback `moveItemStackTo(stack, 9, 45, false)` call, targeting the elytra slot's index by `mayPlace` check, and must NOT use a slot range that overlaps vanilla's 9..45.

---

## R3. `AbstractContainerMenu.moveItemStackTo(ItemStack, int, int, boolean)` — semantics

JVM signature: `protected boolean moveItemStackTo(Lnet/minecraft/world/item/ItemStack;IIZ)Z`

Reconstructed:

```java
protected boolean moveItemStackTo(ItemStack stack, int startIndex, int endIndex, boolean reverse) {
    boolean didMove = false;
    int i = reverse ? endIndex - 1 : startIndex;

    // PASS 1: merge into existing matching stacks (does NOT call mayPlace)
    if (stack.isStackable()) {
        while (!stack.isEmpty() && (reverse ? i >= startIndex : i < endIndex)) {
            Slot slot = this.slots.get(i);
            ItemStack inSlot = slot.getItem();
            if (!inSlot.isEmpty() && ItemStack.isSameItemSameComponents(stack, inSlot)) {
                int totalCount = inSlot.getCount() + stack.getCount();
                int maxStack   = slot.getMaxStackSize(inSlot);
                if (totalCount <= maxStack) {
                    stack.setCount(0);
                    inSlot.setCount(totalCount);
                    slot.setChanged();
                    didMove = true;
                } else if (inSlot.getCount() < maxStack) {
                    stack.shrink(maxStack - inSlot.getCount());
                    inSlot.setCount(maxStack);
                    slot.setChanged();
                    didMove = true;
                }
            }
            i += reverse ? -1 : 1;
        }
    }

    // PASS 2: drop into empty slots (DOES call mayPlace + mayPlace-limited setByPlayer)
    if (!stack.isEmpty()) {
        i = reverse ? endIndex - 1 : startIndex;
        while (reverse ? i >= startIndex : i < endIndex) {
            Slot slot = this.slots.get(i);
            ItemStack inSlot = slot.getItem();
            if (inSlot.isEmpty() && slot.mayPlace(stack)) {
                int maxStack = slot.getMaxStackSize(stack);
                slot.setByPlayer(stack.split(Math.min(stack.getCount(), maxStack)));
                slot.setChanged();
                didMove = true;
                break;                          // PASS 2 breaks after first placement
            }
            i += reverse ? -1 : 1;
        }
    }
    return didMove;
}
```

Key facts:
- PASS 1 (merging into existing) does NOT check `mayPlace` — Mojang assumes same-item merges are legal.
- PASS 2 DOES check `mayPlace` AND uses `getMaxStackSize(stack)` (slot-aware, per-stack).
- Return value: `true` iff any item moved; caller in `quickMoveStack` treats `false` as "failed, return EMPTY".
- PASS 2 only places into ONE empty slot, then breaks — so a 64-stack of potatoes does not spray across 4 empty chests of the same cell range in a single call.

**What this means for our fix:** If our elytra slot's `mayPlace` returns `true` only for elytras (vanilla `Equippable.slot == CHEST` + our predicate), putting its index OUTSIDE the 9..45 hotbar/inv range ensures vanilla `moveItemStackTo(stack, 9, 45, ...)` never lands in our slot. We then add our own targeted `moveItemStackTo(stack, elytraIdx, elytraIdx+1, false)` for shift-click auto-equip.

---

## R4. `ArmorSlot.mayPickup` + `PREVENT_*` components

Class: `net.minecraft.world.inventory.ArmorSlot extends Slot` (package-private).

```java
public boolean mayPickup(Player player) {
    ItemStack inSlot = this.getItem();
    if (!inSlot.isEmpty()
        && !player.isCreative()
        && EnchantmentHelper.has(inSlot, EnchantmentEffectComponents.PREVENT_ARMOR_CHANGE)) {
        return false;
    }
    return super.mayPickup(player); // base Slot.mayPickup returns true
}
```

`EnchantmentHelper.has(ItemStack, DataComponentType<?>)` — static. Returns true if ANY enchantment on the stack carries that component.

### `EnchantmentEffectComponents` — which is which?

Both exist and mean different things:

| Field (JVM type `DataComponentType<Unit>`) | Enchantment | Effect |
|---|---|---|
| `PREVENT_ARMOR_CHANGE` | **Curse of Binding** | Blocks taking the item out of an armor slot (this is what `ArmorSlot.mayPickup` checks) |
| `PREVENT_EQUIPMENT_DROP` | **Curse of Vanishing** | Prevents the item from dropping on death (different hook) |

**What this means for our fix:** Our custom elytra slot must implement the same `mayPickup` pattern using `EnchantmentEffectComponents.PREVENT_ARMOR_CHANGE`. Using `PREVENT_EQUIPMENT_DROP` would be wrong — it's the vanishing curse.

---

## R5. `ClientboundSetEquipmentPacket` — vanilla broadcast

Class: `net.minecraft.network.protocol.game.ClientboundSetEquipmentPacket implements Packet<ClientGamePacketListener>`.

Constructor: `ClientboundSetEquipmentPacket(int entityId, List<Pair<EquipmentSlot, ItemStack>> slots)`.
Static: `public static final StreamCodec<RegistryFriendlyByteBuf, ClientboundSetEquipmentPacket> STREAM_CODEC`.

**Trigger:** fires only from `LivingEntity.handleEquipmentChanges` (see R1). It is built from the *set* of slots whose stacks differ from `lastEquipmentItems`.

**Audience:** `ServerChunkCache.sendToTrackingPlayers(Entity, Packet)` — everyone in the entity's tracking chunk range (NOT the entity itself; use `sendToTrackingPlayersAndSelf` to include the entity when the entity is a player).

**Payload format:** `[varInt entityId] [loop of (byte slotId ORed with 0x80 continuation bit)(ItemStack)]`. The `CONTINUE_MASK` (0x80) on the slot byte marks "more entries follow". Final entry has that bit cleared.

**What this means for our fix:** The elytra slot is NOT covered by vanilla `EquipmentSlot` so it cannot ride `ClientboundSetEquipmentPacket`. We must define our own S2C payload and broadcast it alongside vanilla equipment updates when the elytra slot changes.

---

## R6. Fabric Networking API (MC 26.1, fabric-networking-api-v1 6.3.0)

### Payload Type & StreamCodec registration (mod init, both sides for play phase)

```java
// CustomPacketPayload interface:
// net.minecraft.network.protocol.common.custom.CustomPacketPayload
// Payload needs: CustomPacketPayload.Type<Self> TYPE  + StreamCodec STREAM_CODEC
// + impl: Type<? extends CustomPacketPayload> type() { return TYPE; }

public record ElytraSlotSyncPayload(UUID playerId, ItemStack stack) implements CustomPacketPayload {
    public static final CustomPacketPayload.Type<ElytraSlotSyncPayload> TYPE =
        new CustomPacketPayload.Type<>(ResourceLocation.fromNamespaceAndPath("elytraslot", "slot_sync"));
    public static final StreamCodec<RegistryFriendlyByteBuf, ElytraSlotSyncPayload> STREAM_CODEC =
        StreamCodec.composite(
            UUIDUtil.STREAM_CODEC, ElytraSlotSyncPayload::playerId,
            ItemStack.OPTIONAL_STREAM_CODEC, ElytraSlotSyncPayload::stack,
            ElytraSlotSyncPayload::new);
    @Override public Type<? extends CustomPacketPayload> type() { return TYPE; }
}
```

Note: `UUIDUtil.STREAM_CODEC` has type `StreamCodec<ByteBuf, UUID>` (class `net.minecraft.core.UUIDUtil`). Since `RegistryFriendlyByteBuf extends FriendlyByteBuf extends ByteBuf`, this composes with `ItemStack.OPTIONAL_STREAM_CODEC` (which is `StreamCodec<RegistryFriendlyByteBuf, ItemStack>`) — use `OPTIONAL_STREAM_CODEC` if the stack may be EMPTY (usually yes).

### Register the payload type at mod init (common to both sides)

```java
// Server/common init:
PayloadTypeRegistry.playS2C().register(ElytraSlotSyncPayload.TYPE, ElytraSlotSyncPayload.STREAM_CODEC);
// (Actual static method is called clientboundPlay() in this version — see signatures below.)
```

Actual signatures (`PayloadTypeRegistry<B>` interface):
- `static PayloadTypeRegistry<RegistryFriendlyByteBuf> clientboundPlay()` — S2C play
- `static PayloadTypeRegistry<RegistryFriendlyByteBuf> serverboundPlay()` — C2S play
- `<T extends CustomPacketPayload> TypeAndCodec<? super B, T> register(CustomPacketPayload.Type<T>, StreamCodec<? super B, T>)`

### Register server-side receiver (if we ever need C2S)

```java
ServerPlayNetworking.registerGlobalReceiver(SomePayload.TYPE,
    (payload, ctx) -> { /* ctx.server(), ctx.player(), ctx.responseSender() */ });
```

Handler interface:
```java
public interface ServerPlayNetworking.PlayPayloadHandler<T extends CustomPacketPayload> {
    void receive(T payload, ServerPlayNetworking.Context ctx);
}
// Context methods: server(), player(), responseSender()
```

### Register client-side receiver (for S2C — our elytra sync handler)

```java
// Fabric client init:
ClientPlayNetworking.registerGlobalReceiver(ElytraSlotSyncPayload.TYPE,
    (payload, ctx) -> { /* ctx.client(), ctx.player() (LocalPlayer), ctx.responseSender() */ });
```

Handler interface is `ClientPlayNetworking.PlayPayloadHandler<T>` with method `void receive(T, Context)`.

### Send S2C to all players tracking an entity (broadcast)

`PlayerLookup.tracking(Entity)` returns `Collection<ServerPlayer>`. Loop + `ServerPlayNetworking.send(player, payload)`:

```java
for (ServerPlayer p : PlayerLookup.tracking(entity)) {
    ServerPlayNetworking.send(p, new ElytraSlotSyncPayload(entity.getUUID(), stack));
}
// To include the entity (if entity is a ServerPlayer):
if (entity instanceof ServerPlayer self) ServerPlayNetworking.send(self, payload);
```

Signature: `static void send(ServerPlayer, CustomPacketPayload)`.

Other useful `PlayerLookup` methods:
- `all(MinecraftServer)` / `level(ServerLevel)` / `tracking(Entity)` / `tracking(BlockEntity)` / `tracking(ServerLevel, BlockPos)` / `tracking(ServerLevel, ChunkPos)` / `around(ServerLevel, Vec3, double)`.

**What this means for our fix:** One payload record + three registration calls (type on both sides, client receiver on the client, server-side broadcast helper that does `PlayerLookup.tracking(entity).forEach(p -> ServerPlayNetworking.send(p, payload))` plus the self-send).

---

## R7. NeoForge Payload Registrar (neoforge 26.1.2.1-beta)

### Register via `RegisterPayloadHandlersEvent` (mod bus)

```java
@SubscribeEvent // on the mod bus
public static void onRegisterPayloads(RegisterPayloadHandlersEvent event) {
    PayloadRegistrar registrar = event.registrar("1").optional(); // version "1", optional()

    registrar.playToClient(
        ElytraSlotSyncPayload.TYPE,
        ElytraSlotSyncPayload.STREAM_CODEC,
        (payload, ctx) -> { /* client-thread handler, ctx.enqueueWork for main-thread work */ });

    // If we need C2S too:
    // registrar.playToServer(TYPE, CODEC, (payload, ctx) -> { ... });
}
```

Signatures (from `PayloadRegistrar`):
- `<T> playToClient(Type<T>, StreamCodec<? super RegistryFriendlyByteBuf, T>, IPayloadHandler<T>)` — S2C with handler
- `<T> playToClient(Type<T>, StreamCodec<? super RegistryFriendlyByteBuf, T>)` — S2C with no handler (useful if client only decodes)
- `<T> playToServer(Type<T>, StreamCodec, IPayloadHandler<T>)` — C2S
- `playBidirectional(...)` — both directions with distinct handlers
- Builder methods: `.versioned(String)`, `.optional()`, `.executesOn(HandlerThread)` (default is the main game thread)

### `IPayloadHandler<T>` and `IPayloadContext`

```java
public interface IPayloadHandler<T extends CustomPacketPayload> {
    void handle(T payload, IPayloadContext ctx);
}
// IPayloadContext methods we care about:
// - Player player()  -> on client, the LocalPlayer; on server, the ServerPlayer
// - CompletableFuture<Void> enqueueWork(Runnable)   -> force main-thread work
// - PacketFlow flow() / ConnectionProtocol protocol()
// - ICommonPacketListener listener()
```

**Important:** Handlers run on the network thread by default. Any world/entity mutation must go inside `ctx.enqueueWork(() -> ...)`.

### Send S2C / broadcast — `PacketDistributor` (static methods)

```java
// single player:
PacketDistributor.sendToPlayer(serverPlayer, payload);
// dimension:
PacketDistributor.sendToPlayersInDimension(serverLevel, payload);
// near:
PacketDistributor.sendToPlayersNear(serverLevel, exclude, x, y, z, radius, payload);
// all:
PacketDistributor.sendToAllPlayers(payload);
// tracking an entity (NOT including the entity itself):
PacketDistributor.sendToPlayersTrackingEntity(entity, payload);
// tracking + self (self is included if it's a ServerPlayer):
PacketDistributor.sendToPlayersTrackingEntityAndSelf(entity, payload);
// chunk trackers:
PacketDistributor.sendToPlayersTrackingChunk(serverLevel, chunkPos, payload);
```

Every method's last parameter is actually varargs (`CustomPacketPayload, CustomPacketPayload...`) — you can batch multiple payloads in one call.

**What this means for our fix:** Parallel story to Fabric. The only difference is the registration entrypoint (`RegisterPayloadHandlersEvent` on the mod bus vs Fabric's `PayloadTypeRegistry` called at mod init) and the broadcast API (`PacketDistributor.sendToPlayersTrackingEntityAndSelf` vs Fabric's `PlayerLookup.tracking + ServerPlayNetworking.send` loop).

---

## R8. Advancement trigger for wearing body armor / elytra

### Search result

- `CriteriaTriggers` in 26.1.2 has NO `EQUIPPED_ITEM`, `EQUIPMENT`, or `WEAR_ARMOR` trigger.
- `ServerPlayer.class` has no `onEquipItem` override — the only relevant new method is `onEquippedItemBroken(Item, EquipmentSlot)` (breakage, not equipping).
- `LivingEntity.onEquipItem` does NOT call any advancement trigger directly (R1 bytecode confirmed: only sound + GameEvent).
- `EnchantedItemTrigger`, `UsingItemTrigger`, `PickedUpItemTrigger`, `ConsumeItemTrigger` exist, but none is equip-specific.

### Actual mechanism vanilla uses: `CriteriaTriggers.INVENTORY_CHANGED`

Class: `net.minecraft.advancements.criterion.InventoryChangeTrigger extends SimpleCriterionTrigger<TriggerInstance>`
Relevant methods:
- `public void trigger(ServerPlayer, Inventory, ItemStack)` — convenience (uses the inventory + the single-item predicate)
- `private void trigger(ServerPlayer, Inventory, ItemStack, int, int, int)` — full form

Called internally from `Inventory.setChanged()` / `Inventory.add()` via `ServerPlayer` when stacks mutate — it scans the full `Inventory` (including armor slots at indices 36..39 in the old array layout / the `armor` list in 26.1) and runs the advancement predicate. So "Suit Up" / "Cover Me With Diamonds" advancements fire based on inventory contents, not equipment events.

**What this means for our fix:** Because our elytra is stored in an external container, not `Inventory`, the vanilla `INVENTORY_CHANGED` trigger won't see it unless the predicate targets the inventory slot where we mirror the elytra. We have two options:
1. Keep a mirror ItemStack inside `Inventory.armor` and sync both. (Easiest — vanilla "wear elytra" advancement works automatically.)
2. Call `CriteriaTriggers.INVENTORY_CHANGED.trigger(serverPlayer, inventory, stack)` manually when our slot changes. (Server-side only — `SimpleCriterionTrigger.trigger` is safe to call from any server thread hook.)

All advancement triggers must be called server-side only (`SimpleCriterionTrigger` looks up the player's `PlayerAdvancements` via `ServerPlayer.getAdvancements()`, which is server-only).

---

## R9. `EquipmentSlot` enum + `EntityEquipment`

`EquipmentSlot` declaration order (matches bytecode `static {}`):
1. `MAINHAND` — HAND, idx=0, id=0
2. `OFFHAND`  — HAND, idx=1, id=5
3. `FEET`     — HUMANOID_ARMOR, idx=0, id=1
4. `LEGS`     — HUMANOID_ARMOR, idx=1, id=2
5. `CHEST`    — HUMANOID_ARMOR, idx=2, id=3
6. `HEAD`     — HUMANOID_ARMOR, idx=3, id=4
7. `BODY`     — ANIMAL_ARMOR, idx=0, id=6
8. `SADDLE`   — SADDLE, idx=0, id=7

`VALUES` is initialized in `<clinit>` as `List.of(values())` — so it contains ALL 8 enum values in the declaration order above.

Supporting sigs:
- `public static EquipmentSlot byName(String)` — reverse lookup by `getSerializedName()`
- `public static StreamCodec<ByteBuf, EquipmentSlot> STREAM_CODEC`
- `public boolean isArmor()` / `canIncreaseExperience()` / `getType()` / `getIndex()` / `getId()`

`EntityEquipment` (class `net.minecraft.world.entity.EntityEquipment`) has:
- `ItemStack set(EquipmentSlot, ItemStack)` — returns **the previous stack** (used by `LivingEntity.setItemSlot`, see R1)

**What this means for our fix:** Iterating `EquipmentSlot.VALUES` in our equipment-diff hook covers every vanilla slot without us re-enumerating. Our custom elytra slot does NOT live in `EquipmentSlot` — that's intentional (we use a separate storage + index) and we must diff it separately from the vanilla pipeline.

---

## R10. Curios / Trinkets detection

### What's on disk

- `C:/Users/warwa/.gradle/caches/modules-2/files-2.1/top.theillusivec4.curios/curios-neoforge/15.0.0-beta.1+26.1/.../curios-neoforge-15.0.0-beta.1+26.1.jar` — present for NeoForge 26.1
- Earlier `curios-neoforge-14.0.0+1.21.11` also cached (older MC)
- No `trinkets-*` jar anywhere in the cache — Trinkets (Fabric equivalent) is not on this machine

### Compile-time-optional detection pattern

The mod's `build.gradle` does NOT include Curios as a compile dep (not required). Use reflective detection at runtime:

```java
private static final boolean CURIOS_LOADED = isClassPresent("top.theillusivec4.curios.api.CuriosApi");
private static boolean isClassPresent(String fqn) {
    try { Class.forName(fqn, false, ElytraSlotMod.class.getClassLoader()); return true; }
    catch (ClassNotFoundException e) { return false; }
}
```

For NeoForge side: `net.neoforged.fml.ModList.get().isLoaded("curios")` is the cleaner API (no reflection).
For Fabric side: `net.fabricmc.loader.api.FabricLoader.getInstance().isModLoaded("trinkets")` (Trinkets is the Fabric analogue).

Curios 15.0 (26.1) API entrypoints (based on package layout): `top.theillusivec4.curios.api.CuriosApi`, `SlotTypePreset`, `CuriosCapability`. Detection is the only part we need — actual integration is future work.

**What this means for our fix:** If Curios is detected on the NeoForge side, we should disable our elytra slot UI (to avoid duplicating Curios' "back" slot) but keep the server-side storage for data continuity. On Fabric, Trinkets isn't present here so no conflict.

---

## R11. `Slot.mayPickup(Player)` protocol

`net.minecraft.world.inventory.Slot.mayPickup(Player)`:

```java
public boolean mayPickup(Player player) {
    return true;
}
```

Default returns `true`. It's called from multiple places, including `AbstractContainerMenu.doClick` (the click-handling state machine) and indirectly via `Slot.tryRemove(int, int, Player)`:

```java
// Slot.tryRemove excerpt (proof mayPickup gates actual take):
if (!this.mayPickup(player)) return Optional.empty();
// ... (then allowModification check for partial split) ...
ItemStack removed = this.remove(Math.min(count, maxCount));
```

So returning `false` blocks both full-take and partial-split from the slot.

**What this means for our fix:** Our elytra slot subclass in `InventoryMenuMixin` must override `mayPickup(Player)` with the binding-curse predicate from R4 (PREVENT_ARMOR_CHANGE + creative bypass). Return semantics: `true`=pickupable, `false`=locked.

---

## R12. `Equippable.swapWithEquipmentSlot` + `swappable()` predicate

### The record and its fields

```
public record Equippable(
    EquipmentSlot slot,
    Holder<SoundEvent> equipSound,
    Optional<ResourceKey<EquipmentAsset>> assetId,
    Optional<ResourceLocation> cameraOverlay,
    Optional<HolderSet<EntityType<?>>> allowedEntities,
    boolean dispensable,
    boolean swappable,            // <-- the flag
    boolean damageOnHurt,
    boolean equipOnInteract,
    boolean canBeSheared,
    Holder<SoundEvent> shearingSound) {...}
```

### `swappable()` predicate is checked BY THE CALLER, not inside `swapWithEquipmentSlot`

`Equippable.swapWithEquipmentSlot(ItemStack inHand, Player)` does NOT check its own `swappable` field. The swap logic runs unconditionally once called.

Instead, `swappable` is checked in `Item.use(Level, Player, InteractionHand)`:

```java
public InteractionResult use(Level level, Player player, InteractionHand hand) {
    ItemStack held = player.getItemInHand(hand);
    Consumable c = held.get(DataComponents.CONSUMABLE);
    if (c != null) return c.startConsuming(player, held, hand);

    Equippable eq = held.get(DataComponents.EQUIPPABLE);
    if (eq != null && eq.swappable()) {          // <-- the gate
        InteractionResult r = eq.swapWithEquipmentSlot(held, player);
        if (r instanceof InteractionResult.Success) return r;
    }
    // ... default fallback ...
}
```

### `swapWithEquipmentSlot` reconstruction

```java
public InteractionResult swapWithEquipmentSlot(ItemStack inHand, Player player) {
    if (!player.canUseSlot(this.slot) || !this.canBeEquippedBy(player.typeHolder()))
        return InteractionResult.PASS;
    ItemStack currentlyWorn = player.getItemBySlot(this.slot);
    // Binding curse check + sameness check
    if ((EnchantmentHelper.has(currentlyWorn, EnchantmentEffectComponents.PREVENT_ARMOR_CHANGE)
         && !player.isCreative())
        || ItemStack.isSameItemSameComponents(inHand, currentlyWorn)) {
        return InteractionResult.FAIL;
    }
    if (!player.level().isClientSide()) {
        player.awardStat(Stats.ITEM_USED.get(inHand.getItem()));
    }
    ItemStack resultHeld, newEquip;
    if (inHand.getCount() <= 1) {
        // Single-item swap: swap-in-hand
        resultHeld = currentlyWorn.isEmpty() ? inHand : currentlyWorn.copyAndClear();
        newEquip   = player.isCreative() ? inHand.copy() : inHand.copyAndClear();
        player.setItemSlot(this.slot, newEquip);
        return InteractionResult.SUCCESS.heldItemTransformedTo(resultHeld);
    } else {
        // Stack > 1 special path
        ItemStack prev = currentlyWorn.copyAndClear();
        ItemStack consumed = inHand.consumeAndReturn(1, player);
        player.setItemSlot(this.slot, consumed);
        if (!player.getInventory().add(prev)) player.drop(prev, false);
        return InteractionResult.SUCCESS.heldItemTransformedTo(inHand);
    }
}
```

**What this means for our fix:** If our elytra item uses the vanilla `Equippable` component with `swappable=true` and `slot=CHEST`, right-clicking will swap with the chestplate slot, NOT our elytra slot — that's the vanilla behaviour. To make right-click route to our custom slot, we need to either (a) set `swappable=false` on the Equippable and intercept `Item.use` / `onUse` on the elytra item, or (b) add an `Item.use` pre-hook that calls our own swap method when the hand item's equippable slot matches our custom slot concept.

---

## R13. Broadcasting payloads to all players tracking an entity

### Vanilla (baseline)

`ServerChunkCache.sendToTrackingPlayers(Entity, Packet<? super ClientGamePacketListener>)` — the mojang API that `handleEquipmentChanges` uses. Also `sendToTrackingPlayersAndSelf`. These take a vanilla `Packet`, not a payload.

To send a custom payload this way:
- Fabric: `ServerPlayNetworking.createClientboundPacket(CustomPacketPayload)` returns a `Packet<ClientCommonPacketListener>`. Then `ServerChunkCache.sendToTrackingPlayers(entity, packet)` works (thanks to covariance on the packet signature).
- NeoForge: easier — just use `PacketDistributor.sendToPlayersTrackingEntityAndSelf(entity, payload)`.

### Fabric — idiomatic

```java
import net.fabricmc.fabric.api.networking.v1.PlayerLookup;
import net.fabricmc.fabric.api.networking.v1.ServerPlayNetworking;

public static void broadcastToTrackers(Entity entity, CustomPacketPayload payload) {
    for (ServerPlayer p : PlayerLookup.tracking(entity)) {
        ServerPlayNetworking.send(p, payload);
    }
    if (entity instanceof ServerPlayer self) {
        ServerPlayNetworking.send(self, payload);
    }
}
```

### NeoForge — idiomatic

```java
import net.neoforged.neoforge.network.PacketDistributor;

public static void broadcastToTrackers(Entity entity, CustomPacketPayload payload) {
    PacketDistributor.sendToPlayersTrackingEntityAndSelf(entity, payload);
}
```

### Timing

Call this from the same server-tick hook where we detect elytra-slot diffs — conceptually right after vanilla's `handleEquipmentChanges` would have fired, i.e. inside `LivingEntity.tick()` post-`detectEquipmentUpdates`. A `@Inject(at="TAIL")` on `LivingEntity.detectEquipmentUpdates()` or a mirror method is the cleanest injection point.

**What this means for our fix:** Two one-liners in the loader-specific modules wrap the networking API, exposed via a common interface used by the core diff logic.

---

## Appendix — file/offset references

All bytecode dumps done with `javap -c -p` against the extracted class files at `/tmp/research/net/minecraft/**`. Offsets quoted inline are from the `javap` listings. Vanilla jar: `C:/Users/warwa/.gradle/caches/fabric-loom/26.1.2/minecraft-client.jar` (mojmapped).
