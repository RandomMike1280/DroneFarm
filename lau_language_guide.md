# LAU Language Guide for Drone Farming

This guide covers the basics of the `.lau` scripting language used for automating drone farming based on the available examples.

## Basic Syntax

### Variables
Declare variables using the `varol` keyword:
```lau
varol item = player.getItem(1)
varol x, z = drone.getPosition() -- Multiple assignment is supported
varol count, maxCount = 0, 20
```

Use `varol` only when declaring new variables. When updating an existing variable, assign to it directly:
```lau
varol seedBudget = 10
seedBudget = seedBudget - 1
```

### Comments
Use `--` for single-line comments. They must be on a separate line or at the end of a statement.
```lau
-- This is a comment
varol result = math.abs(-50) -- Prints 50
```

### Strings
Strings are joined (concatenated) using the `+` operator, not the `..` operator typically found in Lua.
```lau
print("Date: " + task.date())
print("Budget: " + seedBudget)
```

Numbers can usually be concatenated with strings directly using `+` in this environment.

### Lists and Dictionaries
`.lau` uses a unified structure for both ordered lists (arrays) and key-value pairs (dictionaries). Note that lists are **1-indexed** (the first element is at index 1).

**Defining Lists:**
```lau
-- Dictionary style
varol droneData = {
    ["Speed"] = 15,
    ["Mode"] = "Automatic"
}

-- Array style (Ordered List)
varol fruits = {"Apple", "Pear", "Banana"}
```

**Accessing and Modifying:**
You do not use `varol` when updating an existing list.
```lau
varol inventory = {"Wheat", "Corn", "Tomato"}
print(inventory[2]) -- Retrieves "Corn"
print(#inventory) -- The '#' operator gets the length of the list

inventory.new = "Potato" -- Add/update using dot notation
inventory[1 + 2] = "Watermelon" -- Add/update dynamically using index (updates 3rd item)

-- Deleting elements uses 'null' instead of 'nil'
droneData.Speed = null
```

Table indexes must be numbers or strings. Avoid using an object/table as a key:
```lau
varol fieldState = {}
fieldState["-1,2"] = "planted" -- Valid
fieldState[3] = "ready" -- Valid

-- Invalid if coord is a table/object:
-- fieldState[coord] = "planted"
```

You can use string keys with either dot notation or bracket notation. Use bracket notation when the key contains special characters:
```lau
stats.Restocks += 1
stats["Watering_Can"] += 1
stats["MaxStock"]["SprinklerV3"] = 5
```

### Control Flow
The language primarily uses Lua-like block structures (`if / then / end`, `while true do`). While C-like syntax elements (`if (condition) { ... }`) are supported, they are known to be poorly implemented and buggy. **Always prefer the Lua style for stability.**

**Logical and Relational Operators:**
*   Use uppercase `AND`, `OR`, and `NOT` for logical conditions.
*   Use `~=` for "not equal" (e.g., `if count ~= 5 then`).
*   Use `==` for equality and normal comparisons like `<`, `<=`, `>`, and `>=`.
*   `true`, `false`, and `null` are the common boolean/empty values.
*   `+=` and `-=` are supported for incrementing/decrementing variables.
*   `break` exits the current loop.

```lau
if item AND it.Type == "Seed" then
    -- code
elseif NOT item then
    -- code
end

while true do
    -- code
    if count >= 10 then break end
end

count += 1
count -= 1

-- Numeric For Loop
for i = 1, 5 do 
    print("Number:", i) 
end 

-- For Loop over Lists (use 'inpairs')
varol fruits = {"Apple", "Pear", "Cherry"} 
for index, fruit inpairs(fruits) do 
    print(index + ". fruit: " + fruit) 
end
```

Short one-line blocks are supported and used in this project:
```lau
if buyingSeeds then return end
if market.buyGear(Enum.Gear.Watering_Can) == false then break end
```

### Defining Functions
You can define standard functions or assign anonymous functions to variables using the `func` keyword and closing with `end`.

```lau
-- Standard definition
func add(a, b) 
    return a + b 
end 

-- Anonymous function assigned to a variable
varol multiply = func(x, y) 
    return x * y 
end 
```

### Functions and Calling
To call a function, you must use parentheses. If you omit parentheses, you are referencing the function, not calling it.
```lau
drone.move(Enum.Direction.East) -- CORRECT: Calls the function
varol myMove = drone.move -- Assigns the function reference to a variable
myMove(Enum.Direction.South) -- Calls the referenced function
```

### Modules and Imports
You can split your code into separate `.laum` module scripts and import them into your main script using the `req()` function.
```lau
-- Import a module
varol farmingModule = req("FarmingOperations.laum")
```

Modules return values with `return`. A common pattern is to return a table of functions:
```lau
-- Movement.laum
func init(config)
    -- setup
end

func moveTo(x, z)
    droneV2.goto(x, z)
end

return {
    init = init,
    moveTo = moveTo,
}
```

The caller may be able to access returned functions by name:
```lau
varol movement = req("Movement.laum")
movement.init(config)
movement.moveTo(0, 0)
```

This project uses numeric access (`movement[1]`, `farming[6]`) because returned module tables are known to work that way in this runtime. Named access is easier to read, but verify it in-game before converting existing modules.

Only main `.lua` / `.lau` scripts should use pragmas like `--!ndrone`. Module files (`.laum`) are loaded with `req()` and should just return their functions/data.

## Events

Events are listeners that run a function when something happens. They use callback functions inside parentheses:
```lau
player.chatted:connect(func(message)
    print("Player said: " + message)
end)
```

An event has three parts:
*   The event to listen to, such as `player.chatted`.
*   The listener type after the colon, such as `:connect` or `:once`.
*   The callback function, such as `func(message) ... end`.

### `:connect`
`:connect` permanently listens and runs the callback every time the event fires.
```lau
player.chatted:connect(func(message)
    if message == "status" then
        player.alert("Running")
    end
end)
```

Do not create permanent listeners inside a `while true do` loop. That creates a new active listener each loop iteration and can overflow the number of active events.

### `:once`
`:once` listens a single time, runs the callback once, then stops listening.
```lau
player.chatted:once(func(item)
    if string.find(item, "Apple") then
        market.buySeed(Enum.Seed.Apple)
    end
end)
```

Use `:once` when you only want one response to a prompt or one future event.

### Event Timing
With `--!ndrone`, event callbacks can run while the drone is still moving or working. This is useful for market refresh handlers:
```lau
market.changedSeedStock:connect(func()
    player.alert("Market seed stocks refreshed!")
    market.buySeed(Enum.Seed.Lotus)
end)

market.changedGearStock:connect(func()
    player.alert("New gears arrived!")
    market.buyGear(Enum.Gear.Watering_Can)
end)
```

Use guard flags if an event handler might take time and you want to prevent overlapping runs:
```lau
varol buyingSeeds = false

market.changedSeedStock:connect(func()
    if buyingSeeds then return end
    buyingSeeds = true
    market.buySeed(Enum.Seed.Lotus)
    buyingSeeds = false
end)
```

## Pragmas and Asynchronous Execution

Pragmas are special instructional commands that tell the `.lau` engine how to process your code. They must be placed at the **very top** of your main `.lau` script (they do not work inside `.laum` module scripts).

### The `--!ndrone` Pragma
By default, `.lau` operates synchronously. When you issue a drone command (like `drone.doFlip()`), the script pauses until the animation finishes. 

Adding `--!ndrone` at the top of your script enables **Asynchronous (Non-Blocking) mode**. The engine will issue the command and instantly skip to the next line.

```lau
--!ndrone
drone.doFlip()
print("hi") -- Prints instantly while the drone is still flipping!
```

### The Overlapping Problem
In async mode, if you issue a command while the drone is busy, **the new command is completely ignored.**

```lau
--!ndrone
drone.doFlip() -- Starts flipping
drone.doFlip() -- IGNORED! Drone is already busy.
```

To safely use async mode, you must manually check the drone's status using `drone.status()`:
```lau
--!ndrone

while true do
    -- Only send commands if the drone is resting
    if drone.status() == Enum.DroneStatus.Sleep then
        drone.doFlip()
    end
    
    -- You can run other background calculations here!
    
    task.wait(0.1) -- Always include a small wait to prevent crashes in tight loops
end
```

In async scripts, wrap physical drone commands with a status wait if the next line depends on that action finishing:
```lau
func waitDrone()
    while drone.status() ~= Enum.DroneStatus.Sleep do
        task.wait(0.05)
    end
end

waitDrone()
droneV2.goto(2, -1)
waitDrone()
drone.harvest()
waitDrone()
```

## Core Objects and APIs

### The `drone` Object
Controls the automation drone's actions and retrieves data about its current tile.

*   **Farming Actions**
    *   `drone.plant(Enum.Seed.[Type])`: Plants a specific seed on the current tile.
    *   `drone.canCrop()`: Returns a boolean indicating if the plant on the current tile can be cropped (cut from root).
    *   `drone.crop()`: Collects crops like Pumpkin, Wheat, Potato, etc.
    *   `drone.canHarvest()`: Returns a boolean indicating if a fruit-bearing tree can be harvested.
    *   `drone.harvest()`: Collects fruit from fruit-bearing trees.
*   **Plant Data**
    *   `drone.getPlant()`: Returns a plant object for the current tile, or `null` if no plant is present. Known properties include `HasFruit`.
    *   `drone.getPlantHasFruit()`: Returns a boolean indicating if the plant has fruit.
    *   `drone.getPlantPercent()`: Returns the growth percentage of the plant itself.
    *   `drone.getFruitPercent()`: Returns the growth percentage of the fruit.
*   **Movement & Position**
    *   `drone.move(Enum.Direction.[Direction])`: Moves the drone one unit in the specified direction (only North, South, East, West).
    *   `drone.doFlip()`: Makes the drone perform a backflip.
    *   `drone.getPosition()`: Returns both X and Z coordinates (`varol x, z = drone.getPosition()`).
    *   `drone.getPositionX()`: Returns only the X coordinate.
    *   `drone.getPositionZ()`: Returns only the Z coordinate, if available in the current runtime.
*   **State & Status**
    *   `drone.status()`: Returns the drone's current state (e.g., `Enum.DroneStatus.Busy` or `Enum.DroneStatus.Sleep`).
    *   `drone.useItem(Enum.Gear.[GearType])`: Commands the drone to use an item, such as a Watering Can.

### The `droneV2` Object
The V2 drone has advanced movement and tile inspection capabilities that read machine and soil buff data.
*   **Advanced Movement**
    *   `droneV2.goto(x, z)`: Commands the drone to travel directly to the specified X and Z coordinates.
    *   `droneV2.swap(Enum.Direction)`: Swaps positions with the plant (or empty space) on the adjacent tile.
*   **Tile Inspection & Gear Data**
    *   `droneV2.isLocked()`: Returns a boolean indicating if the plant on the tile is locked (cannot be swapped).
    *   `droneV2.hasGear()`: Returns a boolean indicating if a machine is currently placed on the tile.
    *   `droneV2.getGear()`: Returns a comprehensive object containing all machine details and soil buff data.
    *   `droneV2.getGearName()`: Returns the specific name of the gear.
    *   `droneV2.getGearDuration()`: Returns its remaining active duration in seconds.
*   **Soil Buffs**
    *   `droneV2.getFertilizer()` / `droneV2.getManualWater()` / `droneV2.getMachineWater()`: Returns an object with `Duration` (remaining seconds) and `Multi` (effectiveness multiplier).
    *   `droneV2.getLightning()`: Returns the remaining duration of the lightning rod protection effect in seconds.

Example water check:
```lau
varol water = droneV2.getManualWater()
if water AND water.Duration == 0 then
    drone.useItem(Enum.Gear.Watering_Can)
end
```

### The `player` Object
Accesses player inventory, stats, and UI interactions.

*   **Inventory & Wealth**
    *   `player.getItem(slotNumber)`: Returns the item in the specified inventory slot (e.g., slot 1 is the first hotbar slot). The item object has properties like `Type`, `Name`, `Amount`, and sometimes `OriginalKey`.
    *   `player.getInventory()`: Returns the entire inventory as a list (table).
    *   `player.getInventorySize()`: Returns the total number of all (Fruit, Seeds, etc.) item types in the inventory.
    *   `player.getFruitCount()`: Returns the total number of fruit item types in the inventory.
    *   `player.getFruitCapacity()`: Returns the max number of fruit item types in the inventory.
    *   `player.scrap()`: Returns the total amount of scrap (currency) the player owns.
    *   `player.calculateFinalScrap(basePrice)`: Returns the actual scrap earned after multipliers (e.g., events).
    *   `player.getTileNumber()`: Returns the player's land size (upgrade level).
*   **UI & Events**
    *   `player.alert("Message")`: Displays an alert message to the player.
    *   `player.chatted:connect(func(message) ... end)`: Event listener triggered when the player types a chat command.
    *   `player.chatted:once(func(message) ... end)`: Event listener triggered by the next chat message only.
*   **Location and Positioning**
    *   `player.getCurrentTile()`: Returns the grid coordinates (X, Z) of the tile the player is currently standing on. If the player is outside the farm area, it returns `null`.
*   **Camera and Player Control**
    *   The following functions act as both Getters and Setters. If you provide an argument, they change the state. If you leave the parentheses empty `()`, they return the current state.
    *   `player.camera(Enum.Camera?)`: Sets or gets the camera's target. Example Set: `player.camera(Enum.Camera.Drone)` Example Get: `varol currentCam = player.camera()`
    *   `player.cameraMode(Enum.CameraMode?)`: Sets or gets the camera's behavioral mode. Example Set: `player.cameraMode(Enum.CameraMode.Follow)` Example Get: `varol currentMode = player.cameraMode()`
    *   `player.controlEnabled(Boolean?)`: Disables (false) or enables (true) the player's movement controls. If called without arguments, returns whether controls are currently active.

**Examples:**
```lau
-- Location check
varol x, z = player.getCurrentTile()

if x ~= null then
	print("Player is on Tile -> X: " + x + " Z: " + z)
else
    player.alert("You are not standing on a tile!")
end

-- Camera manipulation
-- Check the current camera target
varol target = player.camera()

-- If the camera is not on the drone, move it to the drone!
if target ~= Enum.Camera.Drone then
    player.camera(Enum.Camera.Drone)
    player.cameraMode(Enum.CameraMode.Follow)
    
    -- Disable player movement while watching the drone
    player.controlEnabled(false)
end
```

### The `playerV2` Object
The Player V2 module introduces advanced events for market interactions and a daily gift system. Accessed with 'playerV2.' prefix.

*   **Daily Gifts**
    *   `playerV2.getGift()`: Attempts to claim the daily login gift. Displays a notification and opens the gift menu if successful.
*   **Interaction Events**
    *   These events allow your script to react to specific player actions in the game world and market.
    *   `playerV2.clicked:connect(func(button, x, z) ... end)`: Triggered when the player clicks in the world. Parameters: `Enum.ClickType`, `PositionX`, `PositionZ`. IMPORTANT: Only triggers on owned tiles or plants. Returns `null` for unowned areas.
    *   `playerV2.boughtSeed:connect(func(seed) ... end)`: Triggered when a seed is purchased. Returns the purchased `Enum.Seed` as a parameter.
    *   `playerV2.boughtGear:connect(func(gear) ... end)`: Triggered when a gear is purchased. Returns the purchased `Enum.Gear` as a parameter.
*   **UI and Navigation**
    *   `playerV2.mainScreenEnable(Boolean)`: Enables (true) or disables (false) the main screen UI.
    *   `playerV2.tpToDrone()`: Instantly teleports your character directly to the drone's current position.
    *   `playerV2.distanceToDrone()`: Returns the numerical distance between your character and the drone.
*   **Leaderboard Data**
    *   `playerV2.getScrapLeaderboardRank()`: Returns your current rank on the Top 50 Scrap Leaderboard as a number. If you are not in the top 50, it returns `null`.

**Examples:**
```lau
-- React to world clicks
playerV2.clicked:connect(func(button, x, z)
    if x ~= null then
        print("Clicked on: " + x + ", " + z)
    end
end)

-- Track seed purchases
playerV2.boughtSeed:connect(func(seed)
    if seed == Enum.Seed.Apple then
        print("Bought Apple seed")
    end
end)

-- Track gear purchases
playerV2.boughtGear:connect(func(gear)
    print("New gear added to inventory: " + gear)
end)

-- Check distance to drone, teleport if it is too far away
varol dist = playerV2.distanceToDrone()
if dist > 50 then
    playerV2.tpToDrone()
    print("Teleported to drone! Distance was: " + dist)
end

-- Check Scrap Leaderboard Rank
varol rank = playerV2.getScrapLeaderboardRank()
if rank ~= null then
    print("I am currently rank " + rank + " on the leaderboard!")
else
    print("I need more scrap to reach the top 50!")
end
```

### The `market` Object
Handles purchasing seeds, selling items, and market events.

*   **Market Data**
    *   `market.getSeedStock()`: Returns current seed stock.
    *   `market.getSeedPrice(Enum.Seed.[Type])`: Returns the price of a specific seed.
    *   `market.getSeedStockTime()` / `market.getGearStockTime()`: Returns time remaining for seed or gear stock refresh.
    *   `market.whatValue(slotNumber)`: Returns the market value of the item in the specified inventory slot.
*   **Transactions**
    *   `market.buySeed(Enum.Seed.[Type])` / `market.buyGear(Enum.Gear.[GearType])`: Buys a specific seed or gear. Some runtimes return `false` when the item cannot be bought.
    *   `market.sellItem(slotNumber)`: Sells the item in the specified inventory slot.
    *   `market.sellAllItem()`: Sells all sellable items from the inventory at once.
*   **Events**
    *   `market.changedSeedStock:connect(func() ... end)`: Triggered immediately when seed stocks refresh.
    *   `market.changedGearStock:connect(func() ... end)`: Triggered immediately when gear stocks refresh.

Example bounded purchase loop:
```lau
varol maxBuys = 20
while maxBuys > 0 do
    if market.buyGear(Enum.Gear.Watering_Can) == false then break end
    maxBuys -= 1
    task.wait(0.05)
end
```

### The `garden` Object
Provides functions for scanning the farm and retrieving plant data.

*   **Scanning the Garden**
    *   `garden.getGardenPositions()`: Scans the entire garden and returns a dictionary list of all active plants. The keys are the string coordinates (e.g., `"X,Z"`) and the values are strictly the `PlantName`s.
    *   `garden.getPlantEnum(Enum.Seed)`: Scans the field and lists ONLY the plants that match the specified seed type (e.g., `Enum.Seed.Apple`). Returns detailed data for each matched plant.
*   **Coordinate Specific Data**
    *   `garden.getPlantPosition(x, z)`: Returns highly detailed data about the plant at the specified X and Z coordinates. The returned object contains properties like `PlantName`, `PlantWeight`, `PlantPercent`, and if it bears fruit, it also includes `HasFruit`, `FruitName`, `FruitWeight`, `FruitPercent`.

When using `garden.getGardenPositions()`, validate keys before using them as table indexes. Table indexes must be numbers or strings.

### The `task` Object
Provides utility functions for time and yielding. Note that loops do *not* strictly require yielding to prevent crashes, but `task.wait()` is available if needed.

*   `task.wait(seconds)`: Pauses the script for the specified number of seconds.
*   `task.date()`: Returns the current date and time as a string.
*   `task.clock()`: Returns a high-precision timestamp (useful for benchmarking code performance: `varol start = task.clock()`).

## Enums

### `Enum.Seed`
Available seed types for planting and purchasing:
`Apple`, `Bamboo`, `Banana`, `Blueberry`, `Bush`, `Cacao`, `Cactus`, `Carrot`, `Coconut`, `Corn`, `Dragon`, `Garlic`, `Glttch`, `Grape`, `Kiwi`, `Lemon`, `Lotus`, `Mango`, `Mushroom`, `Onion`, `Pear`, `Pepper`, `Pineapple`, `Pomegranate`, `Potato`, `Pumpkin`, `Strawberry`, `Tomato`, `Tree`, `Watermelon`, `Wheat`

*(Note: Use the standard seed name like `Enum.Seed.Apple`. Variations like `Enum.Seed.AppleTree` are incorrect and will not work).*

### `Enum.Direction`
Used for drone movement. There are exactly 4 available directions:
`Enum.Direction.North`, `Enum.Direction.East`, `Enum.Direction.South`, `Enum.Direction.West`.

### `Enum.DroneStatus`
Used to check if the drone is ready for a command, especially in async mode.
*   `Enum.DroneStatus.Busy`: The drone is currently performing an action (like moving or flipping).
*   `Enum.DroneStatus.Sleep`: The drone is idle and ready to receive a new command.

### `Enum.Gear`
Represents tools or machines.
Known gear enum names used by this project:
*   `Enum.Gear.Fertilizer`
*   `Enum.Gear.Lightning_Rod`
*   `Enum.Gear.Sprinkler`
*   `Enum.Gear.SprinklerV2`
*   `Enum.Gear.SprinklerV3`
*   `Enum.Gear.Watering_Can`: Used to manually water tiles via `drone.useItem()`.

## Built-in Functions
*   `print("Message")`: Prints text to the console.
*   `tonumber(string)`: Converts a string to a numeric value.
*   `req("Module.laum")`: Loads a module file and returns its exported value.

### The `string` Module
*   `string.find(str, substring)`: Returns the starting index of the substring within the string (1-indexed).
*   `string.match(str, pattern)`: Returns a match for the pattern, if supported by the runtime.
*   `string.sub(str, startIndex)`: Returns a substring starting from the specified index.

### The `math` Module
*   `math.random(min, max)`: Generates a random integer between the two specified numbers.
*   `math.round(number)`: Rounds a decimal number to the nearest integer (e.g., 4.6 -> 5).
*   `math.abs(number)`: Returns the absolute value (positive form) of the number. Useful for distances.
*   `math.pi`: Returns the mathematical constant Pi (3.1415...).

### The `list` Module
*   `list.find(listObject, item)`: Searches for `item` in `listObject` and returns its index (1-based). If the item is not found, it returns `null`.

## Automation Patterns From This Project

### Safe Drone Movement Wrapper
Direct movement with `droneV2.goto()` is fast, but in `--!ndrone` mode you should wait before and after physical commands:
```lau
func waitDrone()
    while drone.status() ~= Enum.DroneStatus.Sleep do
        task.wait(0.05)
    end
end

func moveTo(x, z)
    waitDrone()
    droneV2.goto(x, z)
    waitDrone()
end
```

### Field State Keys
Use stable coordinate strings for field-state tables:
```lau
func stateKey(x, z)
    return x + "," + z
end

varol key = stateKey(-1, 3)
fieldState[key] = "empty"
```

Avoid indexing field-state tables with plant objects, coordinate objects, or other tables.

### Seed Inventory Matching
Inventory items can use different names for seeds. This project checks several fields:
```lau
if item AND (item.Type == "Seed" OR item.Type == "Seeds") then
    if item.Name == "Lotus" OR item.OriginalKey == "Lotus" then
        print("Found Lotus seeds")
    end
end
```

### Market Refresh Automation
Use market events for immediate buying instead of checking stock timers only after a long sweep:
```lau
varol buyingGears = false

market.changedGearStock:connect(func()
    if buyingGears then return end
    buyingGears = true
    varol maxBuys = 20
    while maxBuys > 0 do
        if market.buyGear(Enum.Gear.Watering_Can) == false then break end
        maxBuys -= 1
        task.wait(0.05)
    end
    buyingGears = false
end)
```
