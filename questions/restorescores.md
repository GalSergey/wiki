# Restoring scoreboard's after changing a nickname

**Note:** This will work on versions as of 1.20.2 and below, and requires the use of a datapack as it uses [macros](https://minecraft.wiki/w/Function_(Java_Edition)#Macros).

* [Checking player UUID](#checking-player-uuid)
* [Adding a new player to storage](#adding-a-new-player-to-storage)
* [Restoring scoreboards](#restoring-scoreboards)

When a player changes his nickname, this will be considered a new player for the scoreboards. Therefore, you may want this player to return all his scores, and, if you want, anything else automatically after changing his nickname.

This article provides an example for restoring scoreboard `ID`, `coins` and `play_time` scores.

If the inventory was not transferred automatically for any reason, then you can also [return the player's inventory](/wiki/questions/storeinventory#putting-things-in-the-original-slot-from-storage) (if you [stored the inventory](/wiki/questions/storeinventory#in-storage) before).

This method uses storage to store some data about the player, namely:

* Scoreboard ID - used for the [scoreboard ID](/wiki/questions/linkentity) of the system<sup>[1]</sup>;
* Player UUID - when changing a nickname, the player's UUID does not change, so this is the main way to find player data;
* Nickname - used in a macro to copy scores from the old to the new player nickname;

\[1\] - Storing the scoreboard ID is optional and is presented in the article as an example.

As a result, storage `example:database` with a `players` list will be created with this data for each player with like the following content:

    # storage example:database
    {players:[{ID:1,UUID:[I;-1166469461,-1413460234,-1975305955,-1073877043],Name:"GalSergey"}]}

When a [player joining to the server](/wiki/questions/playerjoin) for the first time (including the first time after changing his nickname), the player does not have any values in the scoreboard, which can be easily detected with the following command:

    execute as @a unless score @s <objective> = @s <objective> run say I don't have <objective>.

So you can run a function that will check if there is a player with this UUID in storage `example:database`. But before we do this, let's first create a function (`example:uuid/new`) in which we will add the new player's data to this storage.

# Adding a new player to storage

So we will first write all the data to be stored in `this` compound tag, so that later it will be easier to add all this data to the storage `example:database players` list.

If you are using a scoreboard ID system, in this function we will immediately give the new player a scoreboard `ID` and also set this value in storage `example:macro this.ID`:

    # function example:uuid/new
    execute store result score @s ID store result storage example:macro this.ID int 1 run scoreboard players add #new ID 1

The player UUID can simply be copied from the player data:

    data modify storage example:macro this.UUID set from entity @s UUID

But getting a player's nickname as a simple string (not a JSON object) is not so easy. The easiest way is to create a player head using [this loot table](https://misode.github.io/loot-table/?share=XUBWyFQl9g) and read the `"minecraft:profile".name` component  (1.20.5 and above) or the `SkullOwner.Name` tag (1.20.2 - 1.20.4).

To do this in an optimal way, you can create an item\_display with a previously known UUID in the load function, but make sure that this position is always loaded. This will avoid using a target selector.

    # function example:load
    execute unless entity 3ecf96f6-5342-4ab1-a629-10926cea8230 run summon item_display ~ ~ ~ {UUID:[I;1053791990,1396853425,-1507258222,1827308080]}

Now you can use /loot to create a player head and write the `item.components."minecraft:profile".name` tag to storage:

    # function example:uuid/get_nickname
    loot replace entity 3ecf96f6-5342-4ab1-a629-10926cea8230 container.0 loot example:player_head
    data modify storage example:macro this.Name set from entity 3ecf96f6-5342-4ab1-a629-10926cea8230 item.components."minecraft:profile".name

Now let's return to the previous function (`example:uuid/new`) and add the received `ID`, `UUID` and `Name` of the player to `storage example:database players`:

    # function example:uuid/new
    data modify storage example:database players append from storage example:macro this

# Checking player UUID

Now that we have a function for adding data of new players to storage, let's create a function to check whether the “new” player is really new, or whether this player has changed his nickname.

First, let's delete any data that may be in `storage example:macro this` and run the macro function (`example:uuid/find`) in which we will look for the UUID of the selected player:

    # function example:uuid/check
    data remove storage example:macro this
    function example:uuid/find with entity @s

In this function, we will insert the player's UUID into the /data modify command to get the player's data object stored in storage:

    # function example:uuid/find
    $data modify storage example:macro this set from storage example:database players[{UUID:$(UUID)}]

So, for example, if the player has `UUID:[I;1053791990,1396853425,-1507258222,1827308080]` then this command will be executed:

    data modify storage example:macro this set from storage example:database players[{UUID:[I;1053791990,1396853425,-1507258222,1827308080]}]

And if there is an object with this UUID in storage, then this entire object will be copied to `storage example:macro this`.

But if this is not the case, then `example:macro this` will not be created in storage, and you can check whether this object exists and, depending on this, run the function of adding new player data to storage (`example:uuid/new`), or the macro function of restoring scoreboards (`example:uuid/restore`):

    # function example:uuid/check
    execute if data storage example:macro this run function example:uuid/restore with storage example:macro this
    execute unless data storage example:macro this run function example:uuid/new

# Restoring scoreboards

Now all that remains is to create a function for restoring scoreboards from the old nickname.

After running this function (`example:uuid/restore`), you need to update the nickname in storage for this player.

*Note: If you wish, you can also keep a history of nickname changes, additionally storing the old nickname if you want.*

To update a player's nickname, run the get player nickname function (`example:uuid/get_nickname`) created when you created the function to add a new player's data to storage and replace the received nickname in the main storage `example:database`:

    # function example:uuid/restore
    function example:uuid/get_nickname
    $data modify storage example:database players[{UUID:$(UUID)}].Name set from storage example:macro this.Name

*Note: The function* `example:uuid/get_nickname` *will change* `example:macro this` *which was used when running the macro function* `example:uuid/restore`\*, but this will not affect the inserted data in this function in any way, since the insertion of data into the function occurs when the macro function is run, but before any commands are executed.\*

Next you need to add copy commands for each scoreboard you want to restore for that player. This have the following pattern:

    $scoreboard players operation @s <objective> = $(Name) <objective>

The only exception is `ID` score, since scoreboard ID is directly stored in storage:

    # function example:uuid/restore
    $scoreboard players set @s ID $(ID)
    $scoreboard players operation @s coins = $(Name) coins
    $scoreboard players operation @s play_time = $(Name) play_time

And there is one last detail left - you need to reset all the scores of the player with the old nickname:

    # function example:uuid/restore
    $execute unless entity $(Name) run scoreboard players reset $(Name)

_Note: It is worth checking that the saved player is offline so as not to reset your scores during debugging._

Ready. Now any player who changes their nickname will automatically restore all the scoreboards they had before changing their nickname.

Below is a complete example of the entire datapack without comments. You can edit this code to suit your needs:

    # function example:load
    execute unless entity 3ecf96f6-5342-4ab1-a629-10926cea8230 run summon item_display ~ ~ ~ {UUID:[I;1053791990,1396853425,-1507258222,1827308080]}
    scoreboard objectives add ID dummy
    scoreboard objectives add coins dummy
    scoreboard objectives add level level
    scoreboard objectives add play_time custom:play_time
    
    # function example:tick
    execute as @a unless score @s ID = @s ID run function example:uuid/check
    
    # function example:uuid/check
    data remove storage example:macro this
    function example:uuid/find with entity @s
    execute if data storage example:macro this run function example:uuid/restore with storage example:macro this
    execute unless data storage example:macro this run function example:uuid/new
    
    # function example:uuid/find
    $data modify storage example:macro this set from storage example:database players[{UUID:$(UUID)}]
    
    # function example:uuid/new
    execute store result score @s ID store result storage example:macro this.ID int 1 run scoreboard players add #new ID 1
    data modify storage example:macro this.UUID set from entity @s UUID
    function example:uuid/get_nickname
    data modify storage example:database players append from storage example:macro this
    
    # function example:uuid/get_nickname
    loot replace entity 3ecf96f6-5342-4ab1-a629-10926cea8230 container.0 loot example:player_head
    data modify storage example:macro this.Name set from entity 3ecf96f6-5342-4ab1-a629-10926cea8230 item.components."minecraft:profile".name
    ## For 1.20.2 - 1.20.4 use:
    ## data modify storage example:macro this.Name set from entity 3ecf96f6-5342-4ab1-a629-10926cea8230 item.tag.SkullOwner.Name
    
    # function example:uuid/restore
    function example:uuid/get_nickname
    $data modify storage example:database players[{UUID:$(UUID)}].Name set from storage example:macro this.Name
    $scoreboard players set @s ID $(ID)
    $scoreboard players operation @s coins = $(Name) coins
    $scoreboard players operation @s play_time = $(Name) play_time
    ## Add here more scoreboards that you want to restore.
    $execute unless entity $(Name) run scoreboard players reset $(Name)
    
    # loot_table example:player_head
    {
      "pools": [
        {
          "rolls": 1,
          "entries": [
            {
              "type": "minecraft:item",
              "name": "minecraft:player_head",
              "functions": [
                {
                  "function": "minecraft:fill_player_head",
                  "entity": "this"
                }
              ]
            }
          ]
        }
      ]
    }
