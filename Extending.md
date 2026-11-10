---
title: Extending ItemForge
---

# Extending ItemForge

## Your own item data

Add a fragment class and put it on your definitions:

```cpp
UCLASS(DisplayName = "Food")
class UMyFoodFragment : public UIFItemFragment
{
	GENERATED_BODY()

public:
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Food")
	float Calories = 200.f;
};

// Reading it back:
if (const UMyFoodFragment* Food = Item.Definition->FindFragment<UMyFoodFragment>())
{
	Eat(Food->Calories);
}
```

In Blueprint, make a child class of `IFItemFragment` and use **Find Fragment By Class**.

Fragments are per definition, not per item: they hold what never changes. Per-item state is in `FIFItem`: count, place,
rotation, loaded rounds, Stats and Data (README, "Per-item state").

## Your own item actions

Subclass `UIFItemAction` (C++ or Blueprint) and list it in the item's **Actions** fragment. `Execute` runs on the server, then
`ConsumeCount` is used up; read `EffectTag` and `Magnitude` there, or ignore them and do anything:

```cpp
UCLASS()
class UMyEatAction : public UIFAction_Consume
{
	GENERATED_BODY()

public:
	virtual bool CanExecute_Implementation(UIFInventoryComponent* Inventory, const FIFItem& Item, AActor* User) const override
	{
		return Super::CanExecute_Implementation(Inventory, Item, User) && GetHunger(User) > 0.f;
	}

	virtual void Execute_Implementation(UIFInventoryComponent* Inventory, const FIFItem& Item, AActor* User) const override
	{
		Feed(User, Magnitude);
	}
};
```

Actions are class defaults: one instance serves every item, so keep per-use state in the user or the item's Stats.

## Your own HUD

`UIFHotbarWidget` and `UIFNotificationsWidget` build a plain default tree. Make a Blueprint child with your own tree (a
`Slots` horizontal box, a `Messages` vertical box) and set it in the interaction component's `HotbarClass` / `NotificationsClass`,
or turn them off and bind your widgets to `OnInventoryChanged`, `OnItemUsed`, `OnItemBroken` and `OnOperationFailed`.

## Your own economy rules

Prices and trades are `FIFTrade` (in `IFInventoryModel.h`), over two plain models like every other rule: the price of one is
the definition's `Value` times the vendor's multiplier (and condition). Different vendors, different multipliers; a vendor whose
prices change (reputation, haggling) sets `BuyMultiplier` / `SellMultiplier` on the server. For a currency carried as items (coins in the grid), give coins a `Value` of 1 and sell them to a vendor, or
convert them with `AddCurrency` when they are picked up (`OnInventoryChanged`).

Crafting results can do more than land in the inventory: bind `OnCrafted` on the server (experience, achievements).

## Your own access rules

```cpp
bool UMyChestInventory::CanBeAccessedBy(const AActor* Requester) const
{
	return Super::CanBeAccessedBy(Requester) && !bLocked;
}
```

## Equipment slots

A slot is a root container with **Kind** `Slot`. It holds one item of any size. Fill **Required Tags** (for example
`ItemForge.Item.Weapon`) and turn off **Auto Fill** so pickups don't equip things on their own. React to equipping in
`OnInventoryChanged`: an item is equipped when its `ContainerId` is the slot's id (`GetRootContainerId(SlotName)`).

Two-handed items: list the slots they keep empty in the Equippable fragment's **Blocks Slots**. The rule lives in the model
(`CheckSlotBlocking`), so prediction, the drop highlight and the server agree.

Equipped items and their attachments: an item is fitted when its `ContainerId` is an attachment slot of another item
(`FIFContainerInfo::bAttachment`, from `GetContainer` or `FIFInventoryModel::GetItemContainers`). Its owner is
`FindContainerOwner(ContainerId)`. The GAS module's `UIFAbilityEquipmentComponent::CollectActiveItems` walks the same way, a
good start for your own stat system.

## Your own presentation

- **Inspect window, character preview:** subclass `UIFInspectWidget` / `UIFCharacterPreviewWidget` in Blueprint (optional
  bindings `PreviewImage`, `NameText`, `DescriptionText`, `HintText`, `Frame`) and set them in the UI settings' classes.
- **Rendered items:** `AIFItemStudio` puts an item and what is fitted to it on a stage seen only by its capture; use it for your
  own screens (`ShowItem`, `SetView`, `SetLive`, `RenderPixels`). `IFRender::ConfigureIsolatedCapture` sets up any capture to film
  one actor alone with sensible exposure.
- **Sounds and effects:** bind the screen's `OnFeedback` (a Blueprint child of the screen, or after `OpenInventoryScreen`).
- **Prompts:** `UIFInputLibrary::OnInputDeviceChanged` (C++) or `GetLastInputDevice` (Blueprint) for your own widgets.

## Your own container rules

Subclass `UIFContainerFilter`, override `CanContain(Item)` (C++ or Blueprint), give the subclass its settings as class defaults,
and set it as a container's **Filter**. It runs on the class default object on the server and on predicting clients, so it must
depend on the item alone (definition, tags, stats, data): no randomness, no server-only state. With **Filter Nested** it also judges
what goes into containers inside items in this one.

## Using the rules without a component

`FIFInventoryModel` is the whole rule set over plain data, with no UObject or world: vendors, crafting previews, AI planning or
your own tests can run it. The component is a replicated wrapper around it.

## Gameplay tags

The plugin declares `ItemForge.Item.Weapon`, `ItemForge.Item.Armor`, `ItemForge.Item.Ammo`, `ItemForge.Item.Magazine`, `ItemForge.Item.Container`,
`ItemForge.Item.Consumable`, `ItemForge.Item.Material`, `ItemForge.Item.Tool`, `ItemForge.Item.Attachment`, `ItemForge.Ammo.Rifle`, `ItemForge.Ammo.Pistol`, `ItemForge.Stat.Durability`,
`ItemForge.Stat.Charges`, `ItemForge.Rarity.*`, `ItemForge.Effect.Heal`, `ItemForge.Currency.Credits` and `ItemForge.Station.Workbench` for its examples. Containers and magazines match any tag, so use
your own.
