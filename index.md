---
title: ItemForge
---

# ItemForge

A replicated grid inventory for Unreal Engine 5, in C++, usable from Blueprint. The server owns the items; the owner's client
applies its own changes at once (client-side prediction) and the server corrects it when it disagrees.

## Quick start

**No C++, five minutes.** After enabling the plugin (step 1 below), open `/ItemForge/QuickStart/L_ItemForgeQuickStart` and press
Play: a ready character (`BP_ItemForge_QuickStartCharacter`) walks with WASD / left stick, picks things up with E (gamepad X), opens its
inventory with Tab (View), uses quick slots with 1-4 (D-pad), and finds a chest, a trader, a workbench and loot. To use it in your
own map, set **World Settings -> GameMode Override** to `BP_ItemForge_QuickStartGameMode`. To use it on your own character instead, add
the **Inventory**, **Inventory Interaction** and **Inventory Equipment Visuals** components (plus **Inventory Crafting** for
crafting): the interaction component binds the default keys by itself. A Blueprint child of `BP_ItemForge_QuickStartCharacter` takes
your mesh and starting items.

1. Install it from Fab (the launcher installs it into the engine's `Engine/Plugins` folder), then enable **ItemForge**
   in **Edit -> Plugins** and restart the editor. A copy in `<YourProject>/Plugins/ItemForge` works too.
2. Add an **Inventory** component (`UIFInventoryComponent`) to your player's pawn or controller. It starts with one 8x6 grid
   named `Backpack`; add containers in **Containers**: more grids, or slots for equipment.
3. Make items: **Content Browser -> Miscellaneous -> Data Asset -> IFItemDefinition**. Examples are in
   `/ItemForge/Items` (tick **Show Plugin Content** in the Content Browser settings, and **Show Engine Content** when the
   plugin is installed into the engine).
4. Give items on the server with `AddItem`, or list them in the component's **Starting Items**.

## Concepts

| | |
|---|---|
| Item definition | A Data Asset: name, icon, size in cells, max stack, whether it may rotate, gameplay tags, and fragments |
| Fragment | Optional behavior on a definition: **Container** (the item has its own grid, and pockets), **Attachments** (slots for other items), **Magazine** (holds rounds), **Equippable**, **World Mesh**. Subclass `UIFItemFragment` for your own |
| Item | `FIFItem`: one stack. Id, definition, count, container, cell, rotation, loaded rounds |
| Container | A grid, or a slot that holds one item of any size. Root containers come from the component; nested ones from items (Container and Attachments fragments): an item's main grid shares its id, its pockets and attachment slots have ids made from the item's id and their name (`MakeItemContainerId`) |
| Required / Blocked tags | Which items a container accepts, matched against the definition's tags |
| Shape | `Shape` on a definition: rows of `#` (taken) and `.` (free) within its Size, for L, T or any other outline. Other items use the free cells. A shape that looks different upside down turns to 180 and 270 degrees too (`bFlipped` on the item, beside `bRotated`) |
| Layout | `Layout` on a container (root, Container fragment, pocket): rows of `#` (usable) and `.` (blocked): a case with a moulded hole, an odd-shaped pouch. Blocked cells aren't drawn |
| Filter | `Filter` on a container: a `UIFContainerFilter` class (C++ or Blueprint, `CanContain(Item)`) on top of the tags. **Filter Nested** applies it to what goes into bags inside it too. `UIFFilter_MaxSize` and `UIFFilter_MaxWeight` come with the plugin |

## Doing things

Every `Request*` function works on any machine. On a client it goes through the local player's own inventory, which is the only one
allowed to send to the server, so the player's pawn or controller needs an inventory for chests to work.

| Function | |
|---|---|
| `RequestMoveItem` | Move, or with a count, split. Onto the same item it merges; onto a single other item it swaps when that one fits where the first was |
| `RequestRotateItem` | Swap width and height in place |
| `RequestLoadAmmo`, `RequestUnloadAmmo` | Magazines: ammo whose tags include the magazine's `AmmoTag` |
| `RequestTransferItem` | To another inventory: a chest, a corpse, another player's when it allows it |
| `RequestDropItem`, `RequestPickup` | To and from an `AIFWorldItem` in the level, with everything nested in the item |
| `RequestSetQuickSlot` | Point a quick slot at an item. `GetQuickSlotItem` reads it |

Each returns the result of the local check (`EIFResult`). When the server refuses a request, `OnOperationFailed` fires on the
machine that made it. `OnInventoryChanged` fires whenever the items shown change, predictions included: bind your UI to it.

On the server: `AddItem` (with a policy: **Auto** tops up stacks first, **New Stacks** leaves them alone, **No Stacking** makes one per
stack), `RemoveItem`, `RemoveItemsByDefinition`, `ClearItems`, `ExportSaveData`, `ImportSaveData`.

## UI

`UIFInventoryScreen::OpenInventoryScreen(PlayerController, Inventory, Other)` opens a full-screen inventory: the player's
inventory, and another one beside it (a chest) when given. It takes input until closed (`CloseScreen`, Escape, Tab, I, or the
gamepad's View button) and `OnScreenClosed` fires.

| | Mouse and keyboard | Gamepad |
|---|---|---|
| Pick up / place | Drag, or click then click | A |
| Put back | Escape | B |
| Rotate | Right click or wheel while holding, R | Y |
| Item actions (rotate, split, unload, open, inspect, equip, quick slot, drop) | Right click, Q | Menu |
| Split a stack | Ctrl+click takes half, S opens the split dialog | X |
| Quick move (other side, or equip/unequip) | Shift+click, F | Right trigger |
| Open a bag, a vest's pockets, a rifle's attachments | Double click | Menu -> Open |
| Equip by dropping on the character | Release a held item over the character preview | Menu -> Equip |
| Turn the character preview | Drag it | Right stick |
| Category tab / search | Click a tab, C; type in the search box | Left trigger |
| Inspect: turn, zoom, close | Drag, wheel, Escape | D-pad or right stick, shoulders, B |
| Move the cell cursor / jump container | Arrows, Page Up/Down | D-pad or stick / shoulders |
| Drop in the world | Release a dragged item outside the panels | Menu -> Drop |

**Presentation.**

- **Inspect** (item menu): the item in 3D with what is fitted to it, on a stage of its own (`AIFItemStudio`) that only the window's
  capture sees. Size, turn speed and maximum zoom in the UI settings (**Rendered Items**).
- **Runtime icons:** an item with something fitted (a rifle with a scope) shows a rendered icon of the whole instead of its plain
  icon, made once per combination by `UIFIconSubsystem` (a few frames after it is first shown; off with **Runtime Icons**).
- **Compare:** hovering an item that would replace an equipped one shows both tooltips, and the hovered one lists what differs
  (weight, value, the stats both carry: "Durability 60 (-40)").
- **Filter:** each panel has a search box (name or tag) and category tabs (**Categories** in the UI settings; weapons, ammo, gear,
  consumables, materials by default). Items that don't match fade but keep their cells.
- **Input device:** `UIFInputLibrary::GetLastInputDevice` follows the player's last keyboard/mouse or gamepad input. The interaction
  prompt shows `PromptKey` or `PromptGamepadKey`, the hotbar numbers or D-pad arrows, and the screen opens in cursor mode from a gamepad.
- **Gamepad virtual cursor** (optional, **Gamepad Virtual Cursor**): the left stick moves a pointer and A clicks with it, like a mouse;
  the D-pad keeps the cell cursor.
- **Feedback:** `OnFeedback` fires on pick, place, equip, invalid, drop, open and close; **Feedback Sounds** in the UI settings plays a
  sound for each (none by default).

While an item is held, the cells it would cover light up: green to place, blue to merge onto a stack, yellow to swap, red when it
can't go there. The highlight comes from running the real operation on a copy, so it never disagrees with the result.

**Restyling.** Every widget is a C++ `UUserWidget` that builds a default tree when its Blueprint child has none. Make a Blueprint
child, design it with widgets named as the class's optional bindings (for example `Icon`, `NameText`, `CountText` on the item tile),
and set it in **Project Settings -> Plugins -> ItemForge UI**, where cell size, colors and font sizes are too.

| Class | |
|---|---|
| `UIFInventoryScreen` | The screen; all input ends in its public functions (`PickUp`, `PlaceHeld`, `RunAction`, `HandleKey`) |
| `UIFInventoryPanel` | One inventory: slots, grids, opened bags |
| `UIFContainerWidget` | One grid or slot, with the placement highlight and the gamepad cursor |
| `UIFItemWidget` | One item tile |
| `UIFTooltipWidget`, `UIFContextMenu`, `UIFSplitDialog`, `UIFInspectWidget` | Popups |
| `UIFCharacterPreviewWidget` | The live character |

## Editor tools

- **Thumbnails:** item definitions show their icon in the Content Browser, or their mesh when they have no icon.
- **Generate Icon:** right-click item definitions -> Generate Icon. The item's World Mesh (else Equippable mesh) is rendered, the
  background made transparent and the image cropped to the item, then saved as `T_<Item>_Icon` beside the definition and set as
  its Icon. From Python or an Editor Utility Blueprint: `IFEditorLibrary.generate_item_icon(definition, 256, True)` in an editor
  session (a `-run=pythonscript` commandlet renders thumbnails empty).
- **Footprint:** the details panel of an item definition draws the cells it takes (its shape), in each orientation it can turn to.
- **Item Editor** (Tools -> ItemForge Item Editor, or right-click a definition -> Open in Item Editor): every item
  definition in the project with a search box; the selected one's details; its **shape** drawn as a grid of cells to click; and an
  **icon studio** (turn, pitch, zoom, light direction and brightness, exposure, resolution) with Preview and Save as Icon. The icon
  studio renders in a preview scene of its own (`IFEditorLibrary.render_item_icon` from Python or Editor Utility Blueprints).
- **Spreadsheets:** right-click definitions -> **Export to CSV...**; edit in any spreadsheet; import the CSV as a Data Table with
  row struct **IFItemRow**; right-click the table -> **Create / Update Item Definitions** (one per row, in the table's folder,
  existing ones updated by name). Rows carry the definition's settings and the common fragments (world mesh, equippable, container,
  magazine, one stat); an empty field leaves a fragment alone, and import never removes fragments (actions, combine rules stay).
  Also callable: `ExportItemsToDataTable`, `ImportItemsFromDataTable`, `ItemsToCSV`, `ImportItemsFromCSV`.
- **Starting layout:** the details panel of an **Inventory** component (on a placed chest, or in a Blueprint) draws its root
  containers with its Starting Items where BeginPlay will put them. Pick an item definition (and Turned, Level), click a cell to
  place it; click an item to select it, then a cell to move it; R turns it, Delete or right click removes it; the selected item's
  Count and Level are editable below. Every change is checked against the real rules (the model's `AddStartingItem`) and can be
  undone; items that won't fit are reported. Also callable: `AddStartingItemAt`, `MoveStartingItem`, `RemoveStartingItem`,
  `SetStartingItemCountAndLevel`, `PreviewStartingLayout`. A Starting Items entry with a **Container** goes there at **Position**
  (`bRotated`), else wherever it fits.

## Equipment

| | |
|---|---|
| Pockets | A **Container** fragment's **Pockets**: more grids or slots on the same item, beside its main grid (a vest's chest pocket and holster). Each has its own size, Required / Blocked tags and Auto Fill. Opening the item shows them all in one row |
| Attachments | An **Attachments** fragment lists slots on the item (scope, muzzle, magazine well), each with Required / Blocked tags, a **Socket** on the item's mesh and an **Offset**. Fitted items move, drop and save with the item. When the item is equipped, `UIFEquipmentVisualsComponent` puts each fitted item's mesh (Equippable, else World Mesh) on that socket, attachments on attachments included. Pickups never fill attachment slots |
| Slot blocking | An **Equippable** fragment's **Blocks Slots** names root slots that must stay empty while the item is in a root slot (a two-handed rifle blocks `OffHand`). Equipping it while one is taken, or filling one while it is equipped, is refused with `SlotBlocked`; the drop highlight shows it |
| Animation layers | An **Equippable** fragment's **Anim Layers** (an Anim Blueprint implementing your Animation Layer Interface) are linked to the owner's body while the item is equipped, on every machine, by `UIFEquipmentVisualsComponent` (`bApplyAnimLayers`). With several equipped items bringing layers, the highest **Anim Layer Priority** wins (a rifle over a backpack); unequipping restores the next one. `GetActiveAnimLayers`, `OnAnimLayersChanged` |
| Skeletal armour | **Follow Body Pose** on an Equippable fragment with a **Skeletal Mesh** rigged to the character's skeleton: drawn following the body's bones (leader pose) instead of attached at a socket: vests, helmets, clothing |
| Character preview | The inventory screen shows the local player's character live beside its equipment: a scene capture of the pawn alone (body, equipment, attachments) in a render target, turned by dragging. Project Settings -> ItemForge UI -> **Character Preview** (size, distance, height, field of view, exposure, or off). `UIFCharacterPreviewWidget` can be placed anywhere else with `SetPreviewActor`. It renders only while on screen |

### Input

The **Inventory Interaction** component binds its default Enhanced Input on the local player's pawn (**Bind Default Input**): the
mapping context `IMC_ItemForge_Default` and the actions in `/ItemForge/Input` (E or gamepad X interact, Tab / I / View inventory,
1-4 or the D-pad for the quick slots). Point it at your own actions and context, or turn it off and call `Interact`,
`ToggleInventory` and `RequestUseQuickSlot` from your own input. The project needs Enhanced Input's player input and input component
classes (the default for new projects).

### CommonUI (optional module)

`ItemForgeCommonUI`: `UIFCommonInventoryScreen` is the inventory screen as a `UCommonActivatableWidget`. Push it on your
menu layer with `PushInventoryScreen(Stack, Inventory, Other)`; CommonUI handles input mode, focus and back, and closing the screen
deactivates it. Projects without CommonUI can ignore the module.

### UMG View Models (optional module)

`ItemForgeMVVM`, for UMG's **Model View View Model** plugin (enabled with ItemForge): bind widgets to inventory data without
code. **Inventory View Model** (`SetInventory`, optional `SetContainerName`, `SetCurrency`): `Title`, `Weight`, `MaxWeight`,
`WeightFraction`, `WeightText`, `MoneyAmount`, `MoneyText`, `ItemCount` and `Items`, a list of **Item View Models** (`DisplayName`,
`Description`, `Icon`, `Count`, `CountText`, `Level`, `LevelText`, `RarityColor`, `RarityName`, `HasStatBar`, `StatFraction`,
`LoadedCount`, `Weight`, `IsEquipped`). Every field notifies when it changes; it refreshes on every inventory change, on the server
and on clients, and an item keeps its view model object while it is listed (List and Tile Views keep their entries). View models
read; actions still go through the component's `Request*` functions.

### Gameplay Ability System (optional module)

The `ItemForgeGAS` module (enabled with the plugin; it needs the engine's Gameplay Abilities plugin, which the plugin turns on):

| | |
|---|---|
| **Abilities (GAS)** fragment | `GrantedAbilities` and `EquippedEffects` while the item is equipped (in a root slot, or fitted to an equipped item when `bActiveWhenAttached`), `UseEffects` for the action below. Every effect gets the item's stats as Set By Caller magnitudes under each stat's tag: a modifier with magnitude "Set By Caller ItemForge.Stat.Durability" follows the item's durability |
| **Inventory Ability Equipment (GAS)** component | Next to the inventory: on the server, grants and removes abilities and effects as items are equipped and unequipped, and applies an item's effects again when its stats change. Finds the ability system on the owner, or use `SetAbilitySystem` (a player state's) |
| **Apply Effects (GAS)** action | List it in an Actions fragment: applies the item's Use Effects to the user's ability system and uses one up (a medkit, a stim). `EffectTag` + `Magnitude` are passed as one more Set By Caller magnitude |

Projects without GAS can ignore the module: nothing in the main module depends on it.

## In the world

| | |
|---|---|
| `UIFInteractionComponent` | On the player's pawn. Finds the world item or storage under the crosshair, or failing that the nearest one in range in front of the camera (third-person cameras look over low things), and shows a prompt ("E   Pick up Pistol"). Bind your keys to `Interact`, `ToggleInventory`, and the inventory's `RequestUseQuickSlot`. Also puts the hotbar and the notifications on screen (`bShowHotbar`, `bShowNotifications`, your own classes in `HotbarClass` / `NotificationsClass`) |
| `UIFEquipmentVisualsComponent` | On the pawn. Attaches each equipped item's Equippable mesh (static, or skeletal when the fragment's SkeletalMesh is set) to the pawn's skeletal mesh, at the item's socket or the one set per slot (`SocketBySlot`, `OffsetBySlot`), and the items fitted to it on its sockets (`GetItemMesh`). Built from replicated items, so every client shows everyone's equipment; nothing on a dedicated server. `OnEquipped` / `OnUnequipped` fire everywhere: hook animation sets to them |
| `AIFStorageActor` | A chest: a mesh and an inventory anyone may open within range. Fill Starting Items |
| `AIFWorldItem` | An item lying in the world; set Initial Definition to place loot. Dropped items fall and settle with physics (`bSimulatePhysicsOnDrop`, `DropHeight`; needs simple collision on the mesh), then stop simulating so a resting item costs nothing |

`RequestEquipQuickSlot(Index)` equips what a quick slot points at: into the first empty slot that takes it, else swapping with the
first one that does.

## Per-item state

Every item can carry numbers of its own and one struct of your own:

- **Stats** (`FIFItemStat`: gameplay tag and value): durability, charges, freshness. Give a definition a **Stats** fragment
  to set what new items start with, their maximum, and whether the tile shows a bar. On the server: `SetItemStat`,
  `AddItemStat` (clamped to 0 and the maximum), and anyone reads `GetItemStat`.
- **Data** (`FInstancedStruct`): any struct, for what a number can't say (an attachment setup, an engraved name).
  `SetItemData` on the server, `GetItemData` anywhere.

Both replicate and save. Two stacks only merge when their state is the same, a split keeps it, and a worn stack picked up stays worn.

**Item level:** the stat `ItemForge.Stat.Level`. A Stats entry's **Per Level** is added to its start value and maximum for each
level above 1 (a level 4 sword with Durability 100 + 10 per level starts at 130 of 130), so equipment effects that read stats (GAS
Set By Caller) scale with it. Items of different levels don't stack. Make them with `AddItem(..., Level)`, a Starting Items entry's
**Level** or a loot entry's level range; tiles show "Lv 3", the tooltip shows it beside the rarity.
`SetContainers` changes the root containers at runtime (a bigger backpack).

On a Stats entry, **Decay Per Minute** wears the stat down on the server (every `DecayInterval` seconds: food going off, a torch
burning). **Break At Zero** fires `OnItemBroken` on the server and the owning client when it reaches 0, and **Destroy When Broken**
removes the item.

## Gameplay

| | |
|---|---|
| Actions | An **Actions** fragment lists `UIFItemAction` classes: they show in the item's menu, and the first one runs from the hotbar. An action has a display name, how many it uses up (`ConsumeCount`, predicted on the owner), a `Cooldown`, and an `EffectTag` + `Magnitude` your game reads. `CanExecute` / `Execute` run on the server: override them in C++ or Blueprint. `UIFAction_Consume` is "Use: take one". `OnItemUsed` fires on the server and the owner |
| Combine | A **Combine** fragment holds rules: dropped onto `Other`, make `Result` (x `ResultCount`) and use up one of either side, or repair (`RepairStat` + `RepairAmount`, clamped to the stat's maximum). Drag one item onto the other; the highlight shows a combine. Predicted, all or nothing |
| Weight | Each definition has a `Weight` (kg). `MaxWeight` on the inventory (0 = none) refuses adds, transfers and pickups past it with `TooHeavy`. The panel title shows the load. `GetTotalWeight` anywhere |
| Rarity | A `Rarity` tag on the definition (`ItemForge.Rarity.Common` to `Legendary`). Project Settings -> ItemForge UI -> **Rarities** sets each tier's colour, rarest last (the order is the sort order): tiles and tooltips use it |
| Sort | `RequestSort(Container, Mode)` merges stacks and lays the grid out again by type, size, name, rarity or weight. One predicted operation; the Sort button and T / R3 use **Default Sort Mode** |
| Quick slots | `RequestUseQuickSlot(Index)` equips what the slot points at when it goes in a free equipment slot, else runs its first action (and answers `OnCooldown` while it cools down) |

The example items show each: the bandage has a Use action (`BP_Action_Heal`), cloth dropped on herbs makes a bandage, a repair kit
dropped on the rifle restores 50 durability. The rifle has a scope rail (`DA_Item_Scope` fits it) and the vest a pocket and a holster.

## Economy

| | |
|---|---|
| Currency | Balances by gameplay tag on each inventory (`ItemForge.Currency.Credits` in the examples): `StartingCurrencies`, `GetCurrency`, and on the server `AddCurrency` (refuses to go below 0). Replicated, saved, shown in the panel title |
| Prices | Each definition's `Value` is the price of one; 0 means it isn't traded |
| Vendors | `AIFVendorActor`, or an **Inventory Vendor** component next to any inventory. `Stock` (finite or endless entries), `RestockInterval`, `BuyMultiplier` / `SellMultiplier`, `ConditionStat` (worn items are worth less), `bLimitedFunds` (the vendor has a balance), `AcceptedTags`, `bKeepSoldItems`. Players buy by taking from it and sell by putting into it: drag and drop, Shift+Click, or `RequestBuy` / `RequestSell` on their own inventory. The tooltip shows the price. Nothing else may change a vendor's inventory. `OnTraded` fires on the server and the owner |
| Crafting | `UIFRecipe` data assets: ingredients, results, `CraftSeconds`, `FailureChance`, `Station` (a tag; empty = anywhere). An **Inventory Crafting** component next to the player's inventory holds `KnownRecipes` and a replicated queue: ingredients leave when a job is queued and come back when it is cancelled, results arrive per craft (dropped at the feet when they don't fit). `AIFCraftingStation` (or an **Inventory Crafting Station** component on any actor) makes recipes with its tag craftable within `StationRange`. The inventory screen shows the crafting panel when the player opens their own inventory or uses a station |
| Loot | `UIFLootTable` data assets: `Guaranteed` entries, then `MinRolls`..`MaxRolls` weighted picks from `Entries` (count ranges, **level ranges** `MinLevel`..`MaxLevel`, nested tables, `bAllowRepeats`). With a **Value Budget**, entries are picked instead until the drops' total value (Value x count) reaches it, never past it (a rich chest, a poor corpse). Set `LootTable` (and `LootSeed` for the same loot every time) on an inventory or a world item; `AddLoot` rolls one at runtime |

The examples: `DA_Recipe_Bandage` (2 cloth, anywhere), `DA_Recipe_RepairKit` (2 scrap, at a workbench) and `DA_Loot_Supplies`.
Crafting is not predicted (it takes time anyway); the crafting panel is mouse-driven.

## Multiplayer

- Items replicate as a fast array: only changed stacks are sent.
- The owner's own moves, splits, merges, rotations and magazine changes are predicted, into pockets and attachment slots too. Each request carries a seed the new item ids
  derive from, so a split creates the same id on both sides. When the server's answer arrives, the client drops what the server has
  processed and replays what it hasn't. A refused request rolls back by itself.
- Transfers between the local player's inventory and one it has open (a chest) are predicted on both sides (`bPredictTransfers`): the
  player's side is acknowledged like its own moves, the chest's side when the chest's own update lists the transfer (`ForeignOps`),
  so neither side shows an item twice or not at all while the answer travels. Transfers between two other inventories, drops,
  pickups and trades wait for the server.
- **Iris:** works with Iris as with the generic system. Enable the Iris plugin and set
  `net.SubObjects.DefaultUseSubObjectReplicationList=1`, as any Iris project needs.
- An inventory others may open (a chest) needs **Allow External Access**; requests from farther than **Access Range** are refused.
  Override `CanBeAccessedBy` for your own rules.

## Saving

`ExportSaveData` returns an `FIFSaveData` you can put in your own SaveGame, or use `UIFSaveLibrary::SaveInventoryToSlot` and
`LoadInventoryFromSlot`, which keep several inventories by key in one slot. Definitions are saved by path: an item whose asset was
removed since is skipped on load and logged.

## Debugging

- **Gameplay Debugger:** press `'` (apostrophe) in game, then the **ItemForge** category's number (6 by default). For the debug actor
  (or your pawn) it lists each inventory as the server has it and as this client sees it: items, each grid's fill, slots, weight,
  money, and the predictions still waiting for the server; once nothing is pending it says whether both agree. In C++ or Blueprint:
  `DescribeForDebug()` and `GetPendingPredictionCount()` on the component. Not compiled into Shipping builds.
- **Unreal Insights:** the model's operations (`ItemForge::ApplyOp`, `MoveItem`, `CanPlace`, `AddItems`, `Sort`, `InsertTree`,
  `RollLoot`, `LoadSave`), prediction (`ItemForge::Reconcile`), replication (`ItemForge::RebuildClientView`, `CommitAuthority`) and
  the UI (`ItemForge::UI::PanelRefresh`, `ContainerRefresh`) are CPU trace scopes: run with `-trace=cpu` and look for `ItemForge::`.
- **Console:** `ItemForge.Dump` logs every inventory in the world as the machine running it sees it.
- **Tests:** 100 automation tests (`Automation RunTests ItemForge` in the console or Session Frontend) check the rules in your project.

## Packaging

Item definitions referenced from your levels, Blueprints or components are cooked automatically. Ones only reached through save
games are not: list their folder in **Project Settings -> Asset Manager** (primary asset type `IFItem`) or under
**Additional Asset Directories to Cook**.

## Extending

See [Extending](Extending.md).

## Status

Pre-release.
