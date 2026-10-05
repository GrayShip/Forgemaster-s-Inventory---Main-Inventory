# Forgemaster-s-Inventory---Main-Inventory
OPENSTARBOUND ENGINE CHANGES
============================

Base:     OpenStarbound v0.1.15.1, commit e604e078e89310f1212dd20f5cd01bd59d25300b
Patch:    patches\opensb-virtual-inventory.patch (git diff against the base commit)

Why
---
To let the Forgemaster's Inventory mod act as the player's main inventory.
Without engine changes a script cannot:
  - receive items when they are picked up (the engine puts them straight into
    the bags);
  - make crafting stations, quests and other mods' scripts see and spend the
    items kept in the mod's storage;
  - replace the inventory button, show its own window next to a container, etc.

Every change is a generic hook: the engine asks the player's scripts
(genericScriptContexts in player.config) and behaves as before when no script
answers. Without mods that use the hooks the game behaves like stock
OpenStarbound. The network protocol is unchanged, so the build can play on
regular OpenStarbound servers.


1. VIRTUAL INVENTORY
--------------------

source/game/StarPlayer.hpp, source/game/StarPlayer.cpp

  New Player methods:
    virtualInventoryItems()    - items the player's scripts declare part of the
                                 inventory. Cached per "revision" reported by
                                 the script.
    virtualInventoryCount()    - how many of those match a descriptor.
    virtualInventoryConsume()  - asks the scripts to remove items.

  A player script may define:
    virtualInventoryRevision()            -> any value; the item list is
                                             queried again when it changes
    virtualInventoryItems()               -> array of item descriptors
    virtualInventoryConsume(desc, exact)  -> how many were removed

  Re-entrancy guard (m_inVirtualInventoryCall): while the engine calls these
  functions, nested queries do not see the virtual inventory. The flag is reset
  with finally(), even if a script throws.

  hasItem / hasCountOfItem / takeItem got an includeVirtual parameter
  (default true).

source/game/StarPlayerInventory.hpp, source/game/StarPlayerInventory.cpp

  hasItem, hasCountOfItem, consumeItems, takeItems and availableItems include
  the virtual inventory (includeVirtual parameter, default true).
  consumeItems spends the real slots first and takes the rest from the virtual
  inventory. The previous consumption code moved to consumeRealItems().
  These functions back crafting stations (CraftingInterface), quests
  (QuestManager), player.hasItem / hasCountOfItem / consumeItem in all scripts,
  and world.entityHasCountOfItem.

  takeItems now returns an empty result when consumption fails (the result
  used to be ignored).


2. PICKUP INTERCEPTION
----------------------

source/game/StarPlayer.cpp (pickupItems, itemsCanHold)

  Before items go into the bags, the engine calls the player's scripts:
    canInterceptItemPickup(desc) -> true if the script will take the item
                                   (the item is then picked up even with
                                   full bags)
    interceptItemPickup(desc)    -> nil: took everything; false: declined;
                                   a descriptor: the remainder for the bags
  Currency items (pixels etc.) are never intercepted.

  New method giveItemWithoutIntercept(): gives an item bypassing interception
  (m_suppressPickupIntercept flag).

source/game/scripting/StarPlayerLuaBindings.cpp

  player.giveItem(item [, allowIntercept]) - items given by scripts are NOT
    intercepted by default and go to the regular inventory. Pass true as the
    second argument to treat the item as a pickup.
    Why: Forgemaster keeps its state in memory while handing out an item and
    saves it after giveItem; if the item was intercepted back into storage,
    that save overwrote it and the item was lost.
  player.hasItem / hasCountOfItem / consumeItem - new optional last argument
    includeVirtual (default true).
  player.virtualInventorySupported() - marks the patched engine.

source/frontend/StarContainerInterface.cpp

  Items taken from a container with shift + click on a slot go through
  Player::pickupItems (they used to be put straight into the bags), so they can
  be intercepted as well.


3. INTERFACE HOOKS
------------------

The engine sends messages to the player's scripts (handled with
message.setHandler). If a handler returns true, the default action is skipped.

source/frontend/StarActionBar.cpp

  "osb.actionBarSlotRightClick" (slot 1-6, "primary"/"alt") - right click on an
  action bar slot. Why: stock behaviour clears the slot only while the vanilla
  inventory is open; the mod also clears it while Forgemaster is open and
  returns the item to storage properly.

source/frontend/StarMainInterface.hpp, source/frontend/StarMainInterface.cpp

  "osb.mainBarInventoryClick" - click on the inventory button of the main bar
  (the mod opens Forgemaster; shift + click opens the vanilla inventory).

  "osb.containerOpened" (container entity id) / "osb.containerClosed" - a
  container was opened / closed. If the script returns true, the vanilla
  inventory is not opened next to the container (the mod opens Forgemaster).

  "osb.merchantOpened" (merchant entity id) / "osb.merchantClosed" - the same
  for merchant (shop) windows that open together with the inventory.

  Window placement: when a script pane whose config has
  "containerCompanion": true is shown instead of the vanilla inventory, the
  container or merchant window is placed next to it with bringPaneAdjacent (right, left,
  above or below), immediately or as soon as that pane opens. The mod's window
  is not moved, so Forgemaster's locked window position is kept.

  New helper methods: playerScriptHook(), placeContainerBeside(),
  placeMerchantBeside(),
  openContainerId(), addToOpenContainer(), addToOpenMerchant().

source/frontend/StarInterfaceLuaBindings.cpp

  interface.openContainerId()        -> id of the open container, or nil
  interface.addToOpenContainer(item) -> puts an item into the open container;
                                        whatever does not fit comes back to the
                                        player as a pickup
  interface.addToOpenMerchant(item)  -> puts an item into the open merchant's
                                        sell window (only while the sell tab is
                                        shown); returns what was not accepted,
                                        or nil
  Why: shift + click in the Forgemaster window puts the item into the open
  merchant's sell window or the open container, like shift + click in the
  vanilla inventory. The existing world.containerAddItems does not work for this on
  the client: the container answers asynchronously, so the function reports
  that nothing was added while the items still reach the container, which can
  duplicate them.


4. ENGINE BUG FIX
-----------------

source/windowing/StarListWidget.cpp (setSelected)

  Crash "Access violation ... ListWidget::setSelected" on shift + left click
  in the Forgemaster window. Cause: selecting a row calls the pane's script,
  the script rebuilds the list, and the engine then accessed the removed row.
  Added a check that the selected position is still inside the list. This is a
  bug in OpenStarbound itself and could affect other mods with lists.


NOT CHANGED
-----------
  - Network protocol and save format.
  - Server logic: everything above runs on the client for the main player
    (isMaster).
  - Functions that iterate items directly (player.itemsWithTag,
    player.inventoryTags, consumeItemWithParameter, consumeTaggedItem) do not
    see the virtual inventory.


HOW TO REVERT
-------------
  Copy the files from backup into win\.
