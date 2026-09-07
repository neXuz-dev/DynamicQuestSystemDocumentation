# Inventory native migration contract

## Status and scope

This document records the Inventory audit baseline and the native contract required before any Blueprint consumer is rewired. The first live adapter is retained as a cold-buildable source checkpoint; it is not integrated or runtime-validated:

- `QInventory` owns the gameplay item record, semantic validation, versioned codec, endpoint state, strict compare-and-swap reconciliation, one atomic whole-record transfer funnel, and the neutral two-endpoint durability journal.
- `InventoryComponent_C` remains in production. It is adapted incrementally; it is not rewritten wholesale.
- `QStorage` remains the persistence mechanism for local containers. It is not the gameplay Inventory authority.
- `ItemsManagerGS_C` remains a downstream identity/materialization consumer. DataManager now exposes the bounded synchronous durable-row primitive used by the tutorial handoff, but remains a persistence consumer rather than an Inventory mutation owner.
- `InventoryComponent_C` and its four reliable server RPCs were inspected live; no Inventory Blueprint asset is claimed rewired by this checkpoint.

The source phase is not integration parity. The in-progress adapter does not become a production owner until it is integrated, its competing Blueprint writers are removed in the same bounded slice, and the gates below pass. Until then, no existing gameplay caller is claimed to reach `QInventory`.

Checkpoint boundary: `InventoryComponent_C` still inherits `ActorComponent`; its four transfer RPCs and their callers remain unchanged. `QInventoryIntegration` contains the dormant live adapter, including packed bag/equipment snapshots, lossless legacy payload, rollback, access validation and lifecycle synchronization. It still calls legacy post-commit replication/save; the production paired-journal publisher and startup recovery hook are NOT implemented or wired. The incomplete publisher draft was removed before this checkpoint. Do not reparent Inventory or route gameplay through the adapter before closing that persistence boundary.

The completed DataManager changes are independently active: ordinary TempDB saves and durable writes share one per-world FIFO, and TempDB now uses one managed primary/backup save completion. DataObject synchronous array encoding uses the same delimiter as its decoder and asynchronous encoder.

## Audited production baseline

### Live assets

The live project index and read-only Blueprint inspection on 2026-08-30 establish this baseline:

| Asset | Current role | Relevant observed shape |
|---|---|---|
| `/Game/Systems/Item/InventoryComponent` | Inventory state, equipment maps, replication, save/load, and all public mutation entry points | Actor component; 44 variables, 67 functions, 3 macros, 3 event graphs, about 1,595 nodes |
| `/Game/Systems/Item/Obj_ItemInstance` | Replicated mutable item object backed by a DataObject | `ItemInstanceId`, `ItemDataAsset`, `Stack`, `Rarity`, `Owner`, attachment links, customization ID, invalid flag; 35 functions, 282 nodes |
| `/Game/Systems/Item/Lib_Inventory` | Cross-inventory helper library | 20 functions, 303 nodes; `TryMoveItemToAnotherInventory` is 45 nodes |
| `/Game/Systems/Item/ItemsManagerGS` | Global legacy instance map and item materializer | 21 variables, 33 functions, 547 nodes; maps instance ID to `Obj_ItemInstance_C` and owns world-drop/shop helpers |
| `/DataManager/GameDataManager` | DataObject database coordinator | Loaded-object map and database save/load functions |
| `/DataManager/DataObject` | Typed key/value legacy record | Persistent flag, ID, typed maps, async encode/decode |
| `/DataManager/PersistentDataComponent` | Actor-to-DataObject lifecycle bridge | Resolves/creates the actor record and participates in EndPlay |
| `/DataManager/DataManagerLib` | DataManager utility surface | Broad legacy read/write referencer surface |

Exact project-index referencer counts were 126 packages for `InventoryComponent`, 90 for `Obj_ItemInstance`, 82 for `Lib_Inventory`, 20 for `ItemsManagerGS`, 26 for `GameDataManager`, 72 for `DataObject`, 194 for `PersistentDataComponent`, and 52 for `DataManagerLib`. These counts make a wholesale replacement unsafe.

The current live bag lookup is not backed by an independent slot map: `InventoryComponent.GetInventoryIndexByInstance` returns `InventoryItems.Array_Find`. `InventoryItems` is therefore the authoritative packed bag order, while the equipment maps are a separate collection. Native bag `SlotIndex` must equal the item's array position and must be reindexed contiguously after insertion or removal; it must not be inferred from an equipment map or treated as a stable sparse slot.

### Current identity and payload

`Obj_ItemInstance_C` currently uses an `FName` `ItemInstanceId`. New IDs are produced by `Lib_ItemSystem.GenerateNewItemInstance` from a timestamp-formatted name and are materialized through `ItemsManagerGS.GetCreateItemInstance`. This is not a suitable native concurrency or persistence identity contract. The native identity is therefore an `FGuid` generated by the authority. There is no ordinal, array-index, timestamp, or QBuilder numeric-ID fallback.

Prompt 09 must migrate identity once at the adapter boundary. A persistent legacy item without a native GUID receives an authority-generated GUID that is written durably into its DataObject before the item is exposed for mutation; failure to persist it makes the endpoint read-only. A transient item receives one authority-generated GUID for its lifetime. A GUID is never derived from the legacy `FName`, array order, slot, or builder ID.

The legacy complete payload is distributed across the item object and its DataObject:

- ItemData stable key / data asset reference;
- stack and rarity;
- owner ID;
- inventory/equipment slot;
- attached-to relationship and the `Slot:AttachmentId` map;
- customization instance ID;
- additional DataObject values that do not yet have a native field.

`Obj_ItemInstance.LoadFromDataObject` reads these values, clamps a legacy stack to at least one, and deletes an instance whose ItemData cannot be resolved. The native boundary does not clamp malformed input: it rejects it. Unknown but valid legacy payload is carried losslessly in versioned extension maps until each field acquires a typed owner.

### Current `TryMoveItemToAnotherInventory`

The exact Blueprint signature is:

- inputs: source `InventoryComponent`, target `InventoryComponent`, `Obj_ItemInstance`, hidden world context;
- output: one `bool Success`;
- no expected source version, target version, transaction identity, quantity contract, authority context, or typed failure.

The graph caches the legacy item ID and persistence flag, resolves the source index, tests `HasFreeSlot`, unequips when needed, calls `RemoveItemFromInventory`, recreates/materializes an object by the cached ID through `ItemsManagerGS`, and calls `AddItemToInventory` on the target. The remove path unregisters and destroys the replicated item and its attachment objects. The add path can merge stacks, consume the incoming identity, or drop overflow. There is no rollback or postcondition.

There are exactly four call nodes in the loaded `InventoryComponent_C` graphs, all in `RoutedPlayerStateFunctions`:

- `SV_MoveItemToVault`: pawn inventory to PlayerState vault; result ignored;
- `SV_MoveItemToInventory`: vault to pawn inventory; false only emits a log;
- `SV_InventoryToVehicle`: pawn inventory to a supplied vehicle inventory, bracketed by QBuilder resource-deposit notifications; result ignored;
- `SV_VehicleToInventory`: supplied vehicle inventory to pawn inventory; result ignored.

All four events are reliable client-to-server RPCs. They accept an item ID, and the vehicle variants accept an `InventoryComponent` reference supplied by the client. They do not carry endpoint versions. The adapter must resolve both endpoints on the server from an authorized interaction/session; a client-provided component reference is not an authority proof.

### Existing mutation ownership

`InventoryComponent_C` currently has multiple mutation owners:

- bag add/remove, stack merge, consume, equipment, attachments, initial spawn, replication, and DataObject save/load;
- QModule reflection bridges that call consume/generate/add and report success without a typed result or postcondition;
- QModule refund that recreates from ItemData and optional rarity, losing identity and other payload;
- NPC dialogue and quest code that writes the reflected `Stack` property directly or invokes reflected removal with an ad-hoc parameter struct, then separately refreshes/saves;
- QWeapon bullet/fire-control paths that still resolve equipped instances reflectively and require exact-item magazine compare-and-swap plus reserve-ammo reservation/rollback;
- QuestManager item inspection and QuestAction removal that still probe ItemData/instance IDs or invoke item removal through reflection;
- restore/travel code that snapshots and reapplies legacy inventory/DataObject state;
- QStorage container APIs that persist container stacks but do not own gameplay Inventory mutation.

Every add, remove, consume, grant, refund, equip, transfer, restore, loot, trade, shop, drop, and quest objective path must eventually enter one authority-checked native mutation funnel. Until that staged integration happens, the legacy paths remain explicit migration debt; the new core does not pretend to intercept them.

### Persistence modes and lifecycle

The legacy component supports two real modes:

- persistent inventories resolve an Inventory ID and array of item IDs through a persistent DataObject;
- transient inventories use the non-persistent load path and may generate `InitialSpawn` content.

Recent history is part of the contract. Commit `14d000eb2` fixed persistent item payload survival and invalid-item cleanup. Commit `8ef55f618` restored first-spawn loadouts for transient patrol guards without regenerating persistent inventories. Commit `311656ff7` added player inventory/vault/equipment snapshot and idempotent post-travel restore. Commit `5a3982a2a` coupled vehicle deposits to quest attribution and fixed the first transfer into an empty target. Commit `99580fc5f` introduced QStorage's codec and container subsystem. These behaviors are restart/network gates, not optional legacy details.

## Native ownership boundary

### Chosen owner

The data owner is the runtime `QInventory` module, with no dependency on UI, QModule, DynamicQuestSystem, QStorage, DataManager, QBuilder, or Blueprint presentation modules. The separate `QInventoryIntegration` module in the same plugin owns the live Unreal/DataManager/interaction bridge. Public core data types are transportable by adapters, but the core does not use reflection.

Dependency direction after integration is one-way:

```text
Blueprint / QModule / DQS / trade / loot adapters
                         |
                         v
                    QInventory
                         |
                         v
           persistence adapter interfaces
                         |
              QStorage or DataManager backend
```

`QInventory` never calls back into a consumer module. This prevents QStorage or presentation cycles.

### Versioned item record

Schema v2 carries:

- `RecordVersion`;
- stable `FGuid Identity`;
- exact legacy `DataObjectId`, separate from the native GUID;
- explicit `Persistent` or `Transient` item mode;
- stable `FName ItemDataKey`;
- positive `Stack`;
- byte-range `Rarity` represented as `int32` at the API boundary;
- `OwnerId`, required for persistent records; a transient owner may legitimately be `None`;
- packed bag array `SlotIndex` plus an optional stable equipment-slot key;
- complete attachment records, each with its own stable identity, ItemData key, stack, rarity, owner, mount key, customization, extension payload, and validity;
- `CustomizationId` plus sorted key/value customization data;
- sorted key/value extension data for lossless fields not yet promoted to typed members;
- sorted opaque DataObject rows for fields not owned by native records; ownership is by exact `(type, key)` tuple, so independent typed fields with the same name are preserved;
- explicit `bValid`.

The root record and every embedded attachment are semantically validated. No field is populated by a partial reflected struct write.

### Endpoint state

Each endpoint carries:

- stable `FGuid InventoryId` in both persistent and transient modes;
- explicit `Persistent` or `Transient` mode;
- owner ID and positive capacity;
- monotonic `ContentVersion` used for optimistic concurrency;
- monotonic `PersistenceGeneration` used for ordered durable writes;
- fail-closed `bReadOnly` state;
- unique root item identities, unique attachment identities across the endpoint, contiguous packed bag indices, and unique equipment-slot keys.

Transient means "not written across process lifetime"; it does not mean "identity may be unstable". Both modes require GUID identity while alive.

## Semantic invariants

The validator rejects the complete state when any invariant fails:

1. schema and record versions are exactly supported v2;
2. inventory, root item, and attachment GUIDs are valid;
3. ItemData keys and DataObject IDs are not `None`; persistent owners are not `None`;
4. stacks are positive and rarity is in `[0, 255]`;
5. bag records have an empty equipment key and exactly fill the contiguous range `[0, BagRecordCount)` with one record per index; equipped records live in the separate equipment maps, have a non-empty unique equipment key and `SlotIndex == -1`;
6. attachment mount keys are non-empty;
7. no root or attachment identity appears twice, and an item cannot attach itself;
8. customization and extension keys are non-empty and remain within codec limits;
9. gameplay bag capacity is in `[1, 100000]`, while content version and persistence generation are valid and incrementable;
10. bag record count does not exceed bag capacity; equipped records do not consume bag capacity, while the complete root-record set remains bounded by the distinct serialization/allocation limit of `4096`;
11. `bValid` is true for every accepted item and attachment.

Malformed or semantically invalid decoded data is never partially installed. The caller retains the last valid snapshot and marks the affected persistence segment/endpoint read-only. Mutation then returns an explicit read-only result.

## Single atomic transfer funnel

### Request and authority

The first vertical moves a whole root record. The request contains a transaction GUID, item GUID, full-stack quantity, optional target slot, and expected versions for both endpoints. Partial stack transfer is explicitly unsupported by this funnel because splitting while preserving one identity would violate uniqueness.

An authority context is created by the server adapter and must prove all of:

- execution is on network authority;
- the principal is identified;
- source access was authorized;
- target access was authorized.

The core rejects a request if any proof is missing. Blueprint RPC ownership alone is not sufficient.

### Deterministic locking

Each endpoint owns one native critical section. The funnel rejects identical endpoint objects or equal `InventoryId` values before locking. Otherwise it compares the four `FGuid` words lexicographically and always locks the lower GUID first, then the higher GUID. Caller direction never changes lock order.

The two locks cover validation, expected-version checks, snapshots, mutation, postconditions, version/generation increments, and rollback. The native critical section is recursive for synchronous same-thread reads, so a backend callback may call `ReadSnapshot()` on either locked endpoint. Callbacks are strictly synchronous: they must not retain state references, launch blocking/async work, or start a nested transfer. The core never waits on persistence while holding an endpoint lock.

### Transaction lifecycle

1. Validate authority and request structure.
2. Reject same source/target; derive deterministic endpoint order and acquire both locks.
3. Validate both complete endpoint states; reject read-only endpoints.
4. Compare both expected `ContentVersion` values.
5. Resolve the source identity exactly once and reject duplicates/missing identity.
6. Require the requested quantity to equal the complete source stack.
7. Resolve a packed target insertion index in `[0, TargetBagCount]`; auto-slot appends at `TargetBagCount`, and an explicit index inserts there. Reject a full target or any request outside the packed range. No sparse hole, stack merge, or overflow drop occurs in this vertical.
8. Copy complete source and target rollback snapshots, then let the typed mutation backend capture any external adapter state.
9. Remove exactly the source record through the mutation backend and verify the removed record is byte-semantic equivalent to the expected record.
10. Apply only the controlled transfer changes: removing a bag record shifts later source indices down, inserting it shifts target indices at or above the insertion point up, the moved root receives the resolved packed index, the prior equipment-slot assignment is cleared, and root/attachment owner and persistence mode become those of the target endpoint. Identity, ItemData key, stack, rarity, attachment payload, customization, validity, and extensions remain unchanged.
11. Add exactly that record to the target.
12. Build the only permitted post-mutation states (`source snapshot - moved record` and `target snapshot + transferred record`) and require semantic equality with both live endpoints. Also verify source absence, target uniqueness/exact payload, expected counts, and complete semantic validity. Any unrelated record or endpoint change fails the transaction.
13. Increment both content versions and both persistence generations once; validate again.
14. Finalize the backend's non-durable external state. A failed begin, remove, add, postcondition, or finalize restores both native snapshots first, then invokes the rollback hook under the same locks so a synchronous re-entrant read observes the restored boundary.
15. Return a typed success result containing before/after versions, item identity, and target slot. Durable persistence is queued afterward and is never awaited while endpoint locks are held.

Any remove failure, add failure, or failed postcondition restores both full endpoint snapshots while both locks remain held. The restored snapshots are revalidated. A rollback failure is a distinct terminal result and must stop further mutation; it is never reported as ordinary transfer failure.

## Codec and persistence adapter contract

`QInventory` schema v2 uses a bounded deterministic binary codec with a magic tag and explicit versions. Strings are length-prefixed UTF-8 with hard limits. Record counts, attachments, maps, and total payload bytes are bounded before allocation. Map keys and opaque typed rows are encoded in canonical order so identical states produce identical payloads. Decode succeeds only when the entire buffer is consumed and semantic validation passes. Output is assigned only on success. The neutral journal is also versioned at v2 and can retain bounded backend recovery payloads; that is groundwork, not a production recovery adapter.

### QBuilder is a resource ledger, not a whole-item endpoint

`QBuilder_Builder_Actor_BP.InventoryComponent` is rebuilt from `Builder_Advanced_Resource` when interacting with a build zone. Its projected items are destroyed and regenerated; their identities are not durable ledger identities. The native adapter explicitly excludes builder-owned components while the existing builder Blueprint route remains intact.

The ledger is persisted by QBuilder's separate world-specific `*_Build_0..3.sav` family. The manager's external autosave delegate ultimately calls the native QBuilder file writer, not TempDB. Runtime builder numeric IDs are reassigned when loading; the existing QStorage projection ID must not be assumed to identify a ledger recovery participant.

A future typed builder transaction must consume a real inventory record for deposit, or create a new real inventory record for withdrawal, while atomically changing the resource ledger. Mapping, exact quantity, access, versions, overflow, rollback and paired durability belong inside that boundary; quest events and projection refresh are post-commit consumers. Do not persist a synthetic Inventory endpoint for the ledger or call builder add/remove delegates after a generic whole-item commit. Remove the superseded projection persistence and the dead `RequestBuilderUpdate -> SendUpdatedResourcesToQBuilder` chain only when that replacement is working.

### Backend publication contract

The QStorage/DataManager adapter is a later integration owner and must obey these rules:

- serialize the complete `QInventory` endpoint snapshot; never project it down to `FQST_ItemStack` fields and discard payload;
- write the snapshot tagged with its `InventoryId`, schema, `ContentVersion`, and `PersistenceGeneration`;
- persist one transfer as a two-endpoint journal unit keyed by `TransactionId`, including both before/after snapshots and generations. A durable prepare precedes endpoint writes; a durable commit marker follows both. Recovery restores both `before` states for an uncommitted prepare or completes both `after` states for a committed unit. One endpoint generation is never independently visible as the durable transfer result;
- when endpoints use different mechanics (for example DataManager player inventory and QStorage vehicle/container), one neutral Inventory persistence coordinator owns the pair journal while each backend remains only a snapshot writer;
- queue at most one write owner per endpoint/segment and commit generations monotonically;
- discard completion from generation `N` if a newer generation is already durable or queued;
- on write failure, restore both endpoints' dirty generations as one unit and surface a typed error;
- on malformed segment, keep the last valid snapshot, mark the segment and endpoint read-only, and do not migrate or overwrite it;
- on shutdown, stop accepting mutations, drain or resolve every prepared pair with an explicit result, then synchronously persist the newest committed generations; an older async write must never overwrite them;
- registration/unregistration must verify both stable identity and live actor/adapter identity;
- enabled QStorage identity resolution must fail explicitly if the GUID cannot be created/resolved; it must not fall back to `BuilderID` or another ordinal.

The QStorage integration source now adds complete `FQInventoryCodec` endpoint records, exact GUID/actor/adapter registration, expected content versions, persistence generations, ordered async ownership, shutdown draining and a neutral journal interface. It preserves the legacy crate path independently instead of projecting native records through `FQST_ItemStack`. Every queued endpoint generation, including generation zero, keeps its exact segment revision and serialized state until completion. Completion callbacks are resolved from detached records rather than while iterating their owning maps. Missing `Contents` for a live record freezes the whole segment read-only instead of writing an empty destructive replacement. Synchronous drain propagates failures, isolates failed segments, continues every ordered generation on healthy segments, and shutdown explicitly abandons prepares that never entered the writer. The neutral two-endpoint coordinator is implemented by `FQInventoryDurabilityCoordinator`; no production Inventory adapter calls it yet.

DataManager now owns one native, game-thread-only durable row bridge for the existing TempDB save object. It validates the exact backend/context identity, bounds record and line counts plus aggregate text before allocation, distinguishes found/not-found/error reads, captures the exact in-memory rows, writes and synchronously flushes the save slot, reloads through the low-level save-game system, and accepts only an order-independent exact encoded-row readback. Any failure restores and verifies the original rows. The Offline Tutorial subsystem consumes this primitive and its duplicate reflection/save/readback implementation has been removed. Despite the legacy name `ResolveOfflineTempDbContext`, the resolver does not inspect net mode: its backend gate requires `TempDB_C` in the connection class hierarchy, and every other connection is rejected. This bridge is not a second Inventory owner.

The live authored default is coherent with that gate: `QangaGameState.GameDataManager.DataBaseConnection` is `TempDB_C`, and `BaseGameMode`, `Lobby_GM`, `Survival_GM`, `Deathmatch_GM`, and `Tutorial_GM` all author `QangaGameState_C` as their exact `GameStateClass`. The scoped six-asset Blueprint search found no `DataBaseConnection` graph override. The old HTTP `QangaDatabaseConnection` Blueprint has zero Asset Registry referencers and no text configuration reference, so no new HTTP Inventory adapter is required. Runtime mutation, an uninspected mode, or a separately selected GameState can still change the connection; those remain integration gates rather than a reason to add a second backend path.

## Staged Blueprint and consumer cleanup

### P0: source boundary (this phase)

- Add `QInventory` records, validation, codec, endpoint locking, typed results, default exact mutation backend, and atomic transfer.
- Add isolated hard QATS source. Do not add QATS module/plugin dependencies in this lane.

### P1: first live transfer adapter

- Add one native adapter that materializes a complete `InventoryComponent_C` snapshot into a `QInventory` endpoint and applies a committed result through one verified write path.
- Read bag order from the authoritative `InventoryItems` array (`GetInventoryIndexByInstance` is `Array_Find`) and mirror its packed indices exactly. Treat equipment maps as a separate collection; any bag hole, duplicate index, or array/record mismatch fails validation.
- Plan GUID admission without side effects after complete transfer feasibility, durably prepare the paired journal before applying that plan, and move the same `Obj_ItemInstance_C` plus attachment objects between legacy owners without destruction/recreation.
- Replace `Lib_Inventory.TryMoveItemToAnotherInventory` internals with a typed server call; keep the Blueprint function only as a compatibility shell until every caller consumes the typed result.
- Resolve source/target server-side, carry both expected versions, and reject stale client requests.
- Rewire the generic `SV_*` paths only after paired persistence and startup recovery are complete. Keep QBuilder on its legacy route until a dedicated typed ledger transaction replaces it; its projected inventory must never become a generic endpoint.
- Remove the old remove/recreate-by-ID path once the native path is live. Do not leave it disabled or commented.

### P1 mutation convergence

- Route add/remove/consume/equip/attachment changes through the same native mutation owner.
- Replace QModule's reflection grant/consume/refund bridge with typed results and full snapshot restoration.
- Replace NPC dialogue and quest direct `Stack` writes/reflected removal with a native consume request.
- Bind `IQWeaponFireControlAmmoAdapter` through a neutral integration module to exact QInventory item/revision and reservation operations; QWeapon must not probe Inventory and QInventory must not depend on QWeapon.
- Replace `QWeaponBulletSubsystem` equipped-item reflection, `QuestManagerSubsystem` ItemData/identity reflection, and `QuestActionBase` reflected removal with the same typed record/transaction boundary.
- Route loot/drop/shop/trade/restore and initial-spawn changes only after their ownership, persistence mode, and postconditions are explicit.

### P2 downstream owners

- `ItemsManagerGS`: become a materialization/cache consumer keyed by native GUID, not the mutation authority. Remove timestamp identity generation only after all saved legacy names have a one-time, explicit migration policy; never synthesize GUIDs from ordinal position.
- DataManager: become a legacy persistence adapter for player inventories until QStorage/native persistence owns that domain. Map complete records, preserve generations, and fail closed on malformed values.
- The first DataManager P2 vertical is one versioned world-drop round trip using a complete payload encoded by the existing `FQInventoryCodec` and persisted through the exact durable-row bridge; it must not add a parallel item codec, become a bulk rewrite, or create an alternate Inventory mutation owner.
- Retire obsolete global maps, DataObject keys, Blueprint save/replication branches, and compatibility helpers immediately after their last consumer moves.

## Verification gates for integration owner

The central integration retains direct QATS dependencies on `QInventory`, `QStorage`, and DataManager. On 2026-09-01 the cold-built Editor executed `35/35` `QATS.QInventory.*`, `11/11` `QATS.QStorage.*`, `4/4` `QATS.DataManager.*`, and `5/5` `QATS.Quest.OfflineTutorial*` with zero failure. On 2026-09-07 the updated native code compiled through Live Coding and executed `36/36` `QATS.QInventory.*`, including `Transfer.PackedBagReindex`, plus `11/11` `QATS.QStorage.*`, with zero error or warning. The newer run validates packed bag removal/insertion reindexing in addition to the previously covered isolated record, bag/equipment separation, strict endpoint reconciliation, codec, locking, transaction and rollback core, synchronous re-entrant reads, durable journal coordinator, exact endpoint generations, fail-closed incomplete snapshots, ordered drain and shutdown failure propagation. DataManager and Offline Tutorial were not rerun in that newer gate; their `4/4` and `5/5` evidence remains the earlier cold-build result. None of these QATS validates crash interruption during physical file rotation, a production Inventory adapter, network parity, power-loss durability, or process-restart recovery; those gates remain open.

### Checkpoint verification — 2026-09-08

The cold `QangaEditor` build succeeded. A subsequent successful Live Coding patch applied the final DataManager worker-only wait and updated journal-v2 assertions. Final suites pass `36/36` QInventory, `11/11` QStorage and `7/7` DataManager with zero warnings; the async/durable FIFO regression passes twenty consecutive repetitions. The final wait uses a shared `FEventRef`, because `TTask::GetResult()` can retract work directly onto its Game Thread caller despite `DoNotRunInsideBusyWait`. The existing pipe still owns all storage execution and ordering.

TempDB and DataObject compile with zero errors/warnings. A transient two-attachment DataObject fixture verifies blocking encode/decode preserves both entries. The Inventory Blueprint remains parented to ActorComponent and the integration adapter remains dormant. No production journal owner or recovery bootstrap is installed. No PIE, multiplayer, packaged, physical-crash or power-loss result is claimed. The old Offline Tutorial `5/5` result above was not refreshed in this checkpoint.

### Hard QATS

- successful whole-record transfer preserves identity and complete payload;
- packed bag removal closes the source gap and insertion shifts the target range while equipment records remain outside bag indexing;
- target full and occupied target slot leave both endpoints unchanged;
- stale source and stale target versions fail independently;
- identical endpoint object and equal endpoint GUID fail before lock acquisition;
- duplicate root/attachment identity fails closed;
- invalid quantity, ItemData key, owner, slot, attachment, customization key, and validity fail;
- remove failure and add failure restore exact snapshots;
- backend callbacks may synchronously re-read both endpoints, including after rollback, without observing a half-restored state;
- deterministic GUID ordering is direction-independent;
- persistent and transient endpoint codec round trips are deterministic;
- truncated, foreign-version, oversized, trailing, and semantically malformed payloads do not replace output state.
- generation-zero overlap, exact later-generation publication, incomplete record/content keysets, failed drains, healthy-segment continuation and shutdown abandonment all resolve explicitly.

### Network/runtime

- dedicated server with two clients concurrently moving the same item: one commit, one stale typed result, no duplicate/loss;
- opposite-direction transfers between the same endpoints under load: no deadlock and monotonic versions;
- vault, player, and vehicle transfers preserve attachments/customization and update owner/slot only;
- full target never drops, deletes, merges, or consumes the source identity;
- unauthorized client-supplied vehicle/source/target references fail before mutation;
- quest deposit counts only committed quantity and emits exactly one add event for an initially empty target;
- QModule consume/grant/refund and dialogue consume return typed failures and preserve full payload on rollback.

### Restart/persistence

- persistent player inventory, vault, equipment, and attachment payload survive process restart;
- transient patrol loadouts spawn once per actor lifetime while persistent patrol inventories do not regenerate;
- tutorial travel capture/restore remains idempotent and exact;
- failed write is retried from the same generation without clearing dirty state;
- out-of-order async completion cannot overwrite a newer generation;
- shutdown during an in-flight flush persists the newest generation or fails explicitly before exit;
- malformed segment loads the last valid generation read-only and is never overwritten by cleanup;
- two QBuilder containers retain distinct GUID-owned content across restart with no ordinal fallback.

## Exact files for the source vertical

The audited source phase owns only:

- `Plugins/QInventory/QInventory.uplugin`;
- `Plugins/QInventory/Source/QInventory/QInventory.Build.cs`;
- `Plugins/QInventory/Source/QInventory/Public/QInventory.h`;
- `Plugins/QInventory/Source/QInventory/Public/QInventoryTypes.h`;
- `Plugins/QInventory/Source/QInventory/Public/QInventoryValidation.h`;
- `Plugins/QInventory/Source/QInventory/Public/QInventoryCodec.h`;
- `Plugins/QInventory/Source/QInventory/Public/QInventoryTransaction.h`;
- matching private `.cpp` files under `Plugins/QInventory/Source/QInventory/Private/`;
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QInventoryAutomationTests.cpp`;
- this document and the Prompt 06 handoffs.

Everything else is an explicit Prompt 09 integration request.
