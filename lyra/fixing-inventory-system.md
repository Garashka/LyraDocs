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

In either case, there is a much simpler solution available to us with a little C++. The `GA_Interaction_Collect` ability already receives the target actor in its payload, so we can instead retrieve the `IPickupable` interface directly from here.

Open your `IPickupable.h` header file and look for the UPickupableStatics class. Add the following as a public function declaration:
```
	UFUNCTION(BlueprintPure, meta = (WorldContext = "Ability"))
	static TScriptInterface<IPickupable> GetIPickupableFromTargetObject(AActor* Target);
```

Next, open the `IPickupable.cpp` cpp file and add the following:
```
TScriptInterface<IPickupable> UPickupableStatics::GetIPickupableFromTargetObject(AActor* Target)
{
	// If the actor is directly pickupable, return that.
	TScriptInterface<IPickupable> PickupableActor(Target);
	if (PickupableActor)
	{
		return PickupableActor;
	}

	// If the actor isn't pickupable, it might have a component that has a pickupable interface.
	TArray<UActorComponent*> PickupableComponents = Target ? Target->GetComponentsByInterface(UPickupable::StaticClass()) : TArray<UActorComponent*>();
	if (PickupableComponents.Num() > 0)
	{
		ensureMsgf(PickupableComponents.Num() == 1, TEXT("We don't support multiple pickupable components yet."));

		return TScriptInterface<IPickupable>(PickupableComponents[0]);
	}

	return TScriptInterface<IPickupable>();
}
```

Finally, press compile.

Once that's done, open up the `GA_Interaction_Collect` gameplay ability again and replace the `GetIPickupableFromActorInfo` node with our new `GetIPickupableFromTargetActor` node, giving it the `Target` from the `GameplayEventData` as an input. The result should look something like this:
![image](https://user-images.githubusercontent.com/8943296/177227504-71c6200e-50fc-4d31-aa92-2ee403fa57aa.png)

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

`OnListItemObjectSet` is called when the tile is updated within its parent container and receives an object which the tile is meant to represent. If you open the parent (`W_InventoryGrid`) and look at its `Construct` function you can see it adds each item in our inventory to the `TileViewWidget` as a `ULyraInventoryItemInstance`.So if we return to our `W_InventoryTile` and cast the ListItemObject to a `ULyraInventoryItemInstance`, we should be able to retrieve any `InventoryFragment_QuickBarIcon` attached to an item instance.

If you've dug around through other parts of Lyra, you may have seen fragments retrieved from items using a `FindFragmentByClass` function. This is implemented in C++ and as suggested by the name, returns the first fragment it finds matching the given class. Using this we can retrieve the QuickBarIcon we implemented earlier and set our brush. You should end up with something like this:
![image](https://user-images.githubusercontent.com/8943296/177232189-93c98388-b791-4808-ba85-9436b8b8a5af.png)

Now when you hit play and collect a rock, you should find it has a texture in your inventory (as well as a default pistol that is spawned by default and had been a mysterious green square until now).

## 6. Toasts don't trigger - Broadcast messages not received by server
Coming soon™

## 7. Item tiles in inventory don't have quantity
Coming soon™

## 8. Duplicate tile warnings
Coming soon™
