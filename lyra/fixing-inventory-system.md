# Fixing Lyra's Inventory System
If you've played around with Unreal Engine 5's [Lyra sample project](https://docs.unrealengine.com/5.0/en-US/lyra-sample-game-in-unreal-engine/) you may have noticed it includes a prototype inventory system. If you've played around with that inventory system, you may have found there's a good reason it's a prototype - it's incomplete.

This page will document some improvements that can be made to get it to a more usable state.

## Foreword
 - This isn't going to give you a fully working inventory system, it's only aiming to patch the very obvious holes in the Lyra inventory system. Project-specific customisations (weight requirements, slot number limitations) are beyond the scope of this page.
 - It is also not going to be a deep dive into Lyra. I am also not going to explain the inner workings of the inventory framework, but will give a brief explanation of how I found the cause of the issues to begin with.
 - This will require some C++, though to get it to a usable state you should be able to copy/paste the excerpts below.
 - Inventories are very game specific and it's unlikely a sample project is going to give you a one-size-fits-all system.
 - This is not necessarily the best or more efficient inventory system, just my attempt to plug the holes in a way that feels in-line with Epic's intended usages of the Lyra framework.

# 1. The Inventory Experience doesn't load
When starting PIE on `L_InventoryTestMap` one of the first things you'll probably notice is that... nothing happens. This is because the `DefaultGameplayExperience` configured in the level's World Settings is not able to load all of the components it defines as part of the experience.
If you open he Output Log you will find a message similar to this:

```
LogAssetManager: Warning: Invalid Primary Asset Id LyraExperienceActionSet:LAS_InventoryTest: ChangeBundleStateForPrimaryAssets failed to find NameData
```

Experience data are `UPrimaryDataAsset`'s and must be explicitly loaded. In Lyra, the GameFeatureData for each plugin defines where the `UAssetManager` will scan for referenced assets.

To fix this we need to add an entry to the plugin's GameFeatureData file telling it where any `LyraExperienceActionSet` can be located. Open the `ShooterMaps` GameFeatureData file in the root of the ShooterMaps Content plugin and add an entry under `Primary Asset Data Types to Scan` like this:  
![image](https://user-images.githubusercontent.com/8943296/177223919-fcfa5932-f11e-43d2-9c70-14980c290e86.png)  
Where Directories should contain the location your LyraExperienceActionSet can be found. For me, I am going to move LAS_InventoryTest to the `ShooterMaps Content/System/Experiences` directory to be consistent with usages in other plugins.

You may also need to reload the map or restart the editor to get the assets to rescan, but should now find something different happens when you walk up to one of the cubes placed in the map:
![image](https://user-images.githubusercontent.com/8943296/177224054-7669c3bd-3c20-4625-8930-c90b4704bc90.png)

# 2. Interact keybind doesn't work
The next thing you'll probably discover is that the interact keybind (default 'E') plays an attack animation but doesn't collect the item. This is because the key is also used by the melee attack input action our `B_TestInventoryExperience` grants us as part of the `DefaultPawnData`.

To fix this, change the input mapping of either `IA_Melee` or `IA_Interact` such that they are no longer duplicates.
 - Input mappings for IA_Melee are located in `IMC_ShooterGame_KBM` and `IMC_ShooterGame_Gamepad`
 - Input mappings for IA_Interact are located in `IMC_InventoryTest`

You should know find that when you walk up to a rock and use your interact key, it plays a placeholder animation and disappears.

# 3. Items are not added to inventory
The next thing you might notice is that even though we can now collect the item, they are never actually added to our inventory. And this is where we're going to have to get hands on with some C++.

Items are granted to the interacting character's `LyraInventoryManagerComponent` in the ability `GA_Interaction_Collect`. In particular, we are interested in this section of the graph where the `AddPickupInventory` node pictured is intended to grant us the item:  
![image](https://user-images.githubusercontent.com/8943296/177225091-cd097972-7298-436b-b85c-f7ea953b2067.png)  

This node is implemented in C++ in the `Inventory/IPickupable.h`/`Inventory/IPickupable.cpp` files. There is nothing actually incorrect with the `AddPickupInventory` implementation, however if you step through with breakpoints you will discover that the IPickupable it receives is invalid. The culprit is actually this node:  
![image](https://user-images.githubusercontent.com/8943296/177225301-438346af-da6e-478b-9a2b-24cf9da6662a.png)  

The node attempts to retrieve an `IPickupable` interface from the `CurrentActorInfo` of the activated ability. Unfortunately, this doesn't work as the `CurrentActorInfo` is actually us! `CurrentActorInfo` is a struct within each instanced GameplayAbility representing the owner of the ability. It contains a lot of useful information relevant to instanced abilities such as the PlayerController, AvatarActor (physical representation in the game world) and others.

There are two possible ways this implementation was intended to work:
1. The instanced ability would belong to the interactible object, thus the CurrentActorInfo would contain the `IPickupable` interface as required. However that would result in a lot of rather useless ASC's in any sizable game.
2. The interact ability is triggered by an event (`TriggerInteraction` in `LyraGameplayAbility_Interact.cpp`) in which they construct a new `ActorInfo` instance and pass it to the `TriggerAbilityFromGameplayEvent`. It is possible they expected activating the ability in this method to update the instance's `CurrentActorInfo`. Unfortunately it doesn't work like this.

Whatever the case, it's not very helpful to us. Fortunately, there is another way to get this information within our ability. Return to the `GA_Interaction_Collect` blueprint.

Assuming this ability is called correctly, the `Target` we receive in our `EventData` payload should be an actor implementing the `IPickupable` interface. This means that we can instead cast our `Target` actor to the `Pickubable` class (the lack of an `I` prefix is intentional) and instead pass that result to our `AddPickupInventory` call. The result should look something like this:  

![image](https://user-images.githubusercontent.com/8943296/177438680-82ae9b0d-9404-4ee9-9ac7-1da88477e8f5.png)

Now when you open the InventoryTestMap and collect a rock, open your inventory (default 'I') and you should notice you now have two green squares for items. One of these is your brand new rock.

## 4. Inventory doesn't toggle off
It might be arguable if this one is a hole, since you can still close the inventory by clicking on the widget and pressing escape (configured as a default back button for the Common UI components), but if you're like me it will still be enough for a minor inconvenience to infuriate you while testing.

The Inventory UI element is displayed by the `GA_ToggleInventory` ability, with all the logic implemented in its parent class `GAB_ShowWidget_WhenInputPressed`. We're going to change the parent class into a toggle so that activating the ability again will hide the widget.

To do this we need to add a `WaitInputPress` node after the widget has been displayed that will then deactivate the displayed widget, like so:  
![image](https://user-images.githubusercontent.com/8943296/177229174-fbdbe303-8169-4f67-8442-8dd7e345d881.png)  

If you press play now, you will discover that the widget flickers as it rapidly appears and disappears. The culprit this time is the InputAction associated with our ability. Open `IA_ToggleInventory`.

Input Actions provide a lot of customization to inputs that without the `EnhancedInputComponent` would need to be implemented through code. For example, the Input Action's `Trigger` property can change under what circumstances functions associated with an Input Action will actually be executed (e.g. when held, when tapped etc). The default behaviour without any triggers defined is to use the trigger `Down`, which will activate continuously while the input is held. A more appropriate trigger for this action might be `Pressed`, which will trigger only once when the input is activated. Change it to something like the following:  
![image](https://user-images.githubusercontent.com/8943296/177229496-01aa1366-67d6-4447-9b87-53956eb38097.png)  

Now the Inventory doesn't flicker, but we can't get rid of it! What gives? Well it turns out that when a `CommonActivatableWidget` is displayed, it defaults to blocking game input. Fortunately there is an intended way around this, it just once again was not implemented.

While the `GAB_ShowWidget_WhenInputPressed` has no functional usages, the similar `GAB_ShowWidget_WhileInputHeld` has several such as the `W_MatchScoreboard_CP`. Opening that scoreboard, we can see that it inherits from `LyraActivatableWidget`, a `CommonActivatableWidget` with a few extra properties for handling input. In particular the `InputConfig` can be used to determine the input mode used when this widget is active.

Open your `W_InventoryScreen` and reparent it to the `LyraActivatableWidget` class (open the widget graph, select `Class Settings` then change `Parent Class`). Then in its `Class Defaults` change `Input Config` to either `Game` or `Game and Menu`.

Now when you play, you should have a toggleable inventory widget!

## 5. Item tiles in inventory don't have icon
Well this one's just another example of good-old unfinished prototyping.

Now, all we have left to do is give our rock an icon. Lyra uses a composable structure for its items. This means that rather than having all properties configured for every inventory item defined, you pick and choose which properties it has available by adding one or more `ULyraInventoryItemFragment`'s to each `ULyraInventoryItemDefinition`.

To add an icon to our rock, we're going to need to add one of these fragments. Open up the `TestID_Rock` blueprint file and add a new `Inventory Fragment Quick Bar Icon`. Set its `Brush` to any texture you like.

We're not done yet. An item is represented in the inventory with a `W_InventoryTile`. If you open this you're going to find it's completely blank, except for a `OnListItemObjectSet` event with no implementation. This is what we need to complete.

`OnListItemObjectSet` is called when the tile is updated within its parent container and receives an object which the tile is meant to represent. If you open the parent (`W_InventoryGrid`) and look at its `Construct` function you can see it adds each item in our inventory to a `CommonTileView` as a `ULyraInventoryItemInstance`.So if we return to our `W_InventoryTile` and cast the ListItemObject to a `ULyraInventoryItemInstance`, we should be able to retrieve any `InventoryFragment_QuickBarIcon` attached to an item instance.

If you've dug around through other parts of Lyra, you may have seen fragments retrieved from items using a `FindFragmentByClass` function. This is implemented in C++ and as suggested by the name, returns the first fragment it finds matching the given class. Using this we can retrieve the QuickBarIcon we implemented earlier and set our brush. You should end up with something like this:
![image](https://user-images.githubusercontent.com/8943296/177232189-93c98388-b791-4808-ba85-9436b8b8a5af.png)

Now when you hit play and collect a rock, you should find it has a texture in your inventory (as well as a default pistol that is spawned by default and had been a mysterious green square until now).

## 6. Toasts don't trigger/Inventory doesn't update while open
One problem you might not have noticed is that when playing as Standalone or a Listen Server your inventory will not currently update if items are added to it while the inventory is open, but if you open and close it it will contain all items. Similarly on Standalone/Listen Server you will not receive the inventory toast intended to appear whenever an item is added to your inventory.

In both these cases we expect updates to be handled by the following Async task:  
![image](https://user-images.githubusercontent.com/8943296/177278475-6ae9fc5c-711b-40cf-9fd7-5766e9f7d6d2.png)  
We know the items are added to our inventory, so time to go and check the event is dispatched correctly. Time for some more C++.

Searching for the GameplayTag we are subscribed to (`Lyra.Inventory.Message.StackChanged`) we can find a single usage of it, within the `LyraInventoryManagerComponent.cpp` file where it is declared as a `FNativeGameplayTag` using a macro. This native tag is then used by the `FLyraInventoryList::BroadcastChangeMessage` function. Alright, looks good so far. If you've dug around in Lyra elsewhere you know this `MessageSystem` is being used successfully, so our issue must be here. You could add a breakpoint here and see if it's the `BroadcastChangeMessage` function is ever hit, but it won't be.

Checking where the `BroadcastChangeMessage` function is called, you might notice a pattern. It's called in the `PreReplicatedRemove`, `PreReplicatedAdd` and `PostReplicatedChange` functions. These are all related to replication provided by the `FFastArraySerializer` structure our InventoryList inherits from, and if you've done much with replication before you can probably guess at this point; they are only called on the remote clients. You could confirm this by testing the feature while running as a client.

To fix this then, we need to ensure that `BroadcastChangeMessage` is also called when we are modifying the inventory list on a Listen Server. The functionality provided to modify the array is again pretty barebones (and some of it not even implimented correctly - looking at you `ConsumeItemsByDefinition`), so for demonstration purposes we'll correct the `FLyraInventoryList::AddEntry(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)` function. Add a call to `BroadcastChangeMessage` after the array is modified and marked as dirty, and you should end up with something like this:

```
ULyraInventoryItemInstance* FLyraInventoryList::AddEntry(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)
{
	ULyraInventoryItemInstance* Result = nullptr;

	check(ItemDef != nullptr);
 	check(OwnerComponent);

	AActor* OwningActor = OwnerComponent->GetOwner();
	check(OwningActor->HasAuthority());

	FLyraInventoryEntry& NewEntry = Entries.AddDefaulted_GetRef();
	NewEntry.Instance = NewObject<ULyraInventoryItemInstance>(OwnerComponent->GetOwner());
	NewEntry.Instance->SetItemDef(ItemDef);
	for (ULyraInventoryItemFragment* Fragment : GetDefault<ULyraInventoryItemDefinition>(ItemDef)->Fragments)
	{
		if (Fragment != nullptr)
		{
			Fragment->OnInstanceCreated(NewEntry.Instance);
		}
	}
	NewEntry.StackCount = StackCount;
	Result = NewEntry.Instance;
	
	MarkItemDirty(NewEntry);

	BroadcastChangeMessage(NewEntry, 0, NewEntry.StackCount);
	
	return Result;
}
```
Now compile, re-open the test map and try collecting an item with your inventory window open. You should now see the items added to your inventory as you collect them, as well as the toasts.

## 7. Duplicate tile warnings
If you open and close your inventory multiple times before collecting an item, you might notice the following warning in your output log: `LogScript: Warning: Script Msg: Cannot add duplicate item into ListView.`. This one's an easy fix.

Remember that Async task we use to listen to change events in the previous step? Well it turns out they don't clean always clean themselves up. Each time you open your inventory, a new `ListenForGameplayMessages` task is created. However when you close the inventory it is not destroyed. Because of this, for each time you have opened your inventory this play session, it will attempt to add a separate tile to or InventoryGrid, causing the above warning.

To resolve this open `W_InventoryGrid` and promote the `ListenForGameplayTask`'s `AsyncAction` pin to a variable. This stores a reference to the task. Then override the Destruct function of the widget and cancel that task. This will cause our task to be cleaned up every time the inventory is closed. The result should look something like this:  
![image](https://user-images.githubusercontent.com/8943296/177283644-3f842211-2e3e-4398-a3da-acd1fff181f7.png)  
The same will need to be done to the `W_ItemAcquiredList` widget.

## 8. Item tiles in inventory don't have quantity
Okay, if you've been following along you should have the Lyra Inventory at the starting line of what an inventory should be. There's one more thing that the majority of inventory systems are going to want, before we start getting into project specifics.

If you've looked through the C++ much, you may have noticed the storage structure for inventory items (`FLyraInventoryItem`) has a StackCount property to track the number of items in each stack. Currently this isn't exposed to Blueprints and so we are unable to display it, so we're going to need to do some C++.

There's multiple ways you could do this - you could for example update the visibility of the StackCount property to be `BlueprintReadOnly` and pass a `FLyraInventoryItem` from your `W_InventoryGrid` to each `W_InventoryTile` (rather than the `ULyraInventoryItem` that is currently used). This might be a better approach if you were to scrap the current widget grid, but unfortunately this won't work so well with the `CommonTileView` currently used as the `AddItem` function used to add tiles only accepts UObjects.

Instead we're going to take some inspiration from Lyra's handling of ammo counts. Open your `LyraInventoryManagerComponent.cpp` file and add the following near the top, just below the existing `UE_DEFINE_GAMEPLAY_TAG_STATIC` we will define a new gameplay tag that will be used to represent the stack count of each item by adding the following:
```UE_DEFINE_GAMEPLAY_TAG_STATIC(TAG_Lyra_Inventory_Item_Count, "Lyra.Inventory.Item.Count");```

Finally inside the `FLyraInventoryList::AddEntry(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)` function add the following line after the `NewEntry.Instance` property is assigned:
`NewEntry.Instance->AddStatTagStack(TAG_Lyra_Inventory_Item_Count, StackCount);`

Your end result should be something like this:
```

ULyraInventoryItemInstance* FLyraInventoryList::AddEntry(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)
{
	ULyraInventoryItemInstance* Result = nullptr;

	check(ItemDef != nullptr);
 	check(OwnerComponent);

	AActor* OwningActor = OwnerComponent->GetOwner();
	check(OwningActor->HasAuthority());


	FLyraInventoryEntry& NewEntry = Entries.AddDefaulted_GetRef();
	NewEntry.Instance = NewObject<ULyraInventoryItemInstance>(OwnerComponent->GetOwner());  //@TODO: Using the actor instead of component as the outer due to UE-127172
	NewEntry.Instance->SetItemDef(ItemDef);
	for (ULyraInventoryItemFragment* Fragment : GetDefault<ULyraInventoryItemDefinition>(ItemDef)->Fragments)
	{
		if (Fragment != nullptr)
		{
			Fragment->OnInstanceCreated(NewEntry.Instance);
		}
	}
	NewEntry.StackCount = StackCount;
	// Add item count as a GameplayTag so it can be retrieved from the ULyraInventoryItemInstance
	NewEntry.Instance->AddStatTagStack(TAG_Lyra_Inventory_Item_Count, StackCount);
	
	Result = NewEntry.Instance;

	MarkItemDirty(NewEntry);

	BroadcastChangeMessage(NewEntry, 0, NewEntry.StackCount);
	
	return Result;
}
```

Now press compile.

Lastly, we need to update the `W_InventoryTile` widget to display this quantity. Currently there is no text component to display this, so your first step is to add one. Then, in the `OnListItemObjectSet` function retrieve the quantity from our `ULyraInventoryItemInstance` using `GetStatTagStackCount` (using the same tag we just defined in C++ `Lyra.Inventory.Item.Count`) and set the text value of your text component. You should end up with something like this:

![image](https://user-images.githubusercontent.com/8943296/177464902-ce40564b-56fc-43dc-af65-c01b8ba3f0cc.png)

Now to test this, start your game and pick up an item. Note that items will not currently combine into the same stack, so if you want to confirm the quantity is displayed correctly for higher quantities you will need to increase the `Stack Count` property in the `Static Inventory` template attached to your level's rock-cubes.

# Limitations
## No manual re-ordering of tile widgets
The `CommonTileView` widget doesn't appear to support manual ordering of widgets. This is going to be an issue for any inventory system that requires such behaviour (e.g. Factorio style).

# Credits
 - Shoptop of the Unreal Slackers Discord for providing a simple Blueprint-only solution to section 3.
