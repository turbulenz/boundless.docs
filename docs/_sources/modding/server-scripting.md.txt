# Server Scripting

## Setup of Lua

* Using a slightly modified LuaJit v2.1.0-beta3.
* JIT is currently disabled! (Bugs that lead to hard-exits on memory allocation failures within Lua scripts.)
* FFI and dynamic-linking of C libraries is disabled, pure Lua scripts only.
* 5.2 Compatibility flags disabled, Lua 5.1 only.
* Modifications/Restrictions of the Lua standard library:
  * `package.searchpath` which was exposed even without the 5.2 compatibility flag is also disabled.
  * `package.loaders` for dynamic-linking are unset.
  * `package.path` and `package.cpath` are unset and unused (effectively hardcoded to "?;?.lua")
  * `stdin`, `stdout` and `stderr` are unset and inaccessible (are effectively already-closed file handles)
  * `loadfile`, `dofile` with no argument (stdin) will fail, as will all other cases where would be by-default using the stdio streams
  * `io.popen` is disabled
  * `io.tmpfile` is disabled
  * `os.tmpname` is disabled
  * `os.getenv` is disabled
  * `os.exit` is disabled
  * `os.setlocale` is disabled
  * `collectgarbage` is available, but with no arguments taken; will only ever do a full garbage collection
  * `jit` (from LuaJit) and `debug` standard libraries are disabled/uncompiled; debug is too dangerous!
* All file access is sandboxed and paths restricted at the Lua-vm level:
  * All file paths provided may only be relative paths to the sandboxed scripting directory; ensure mods cannot mess with non-mod files!!
  * All file paths must be basic ascii char-set only
  * Characters `<>:"|?*\` as well as the "space" character may not be used
  * All paths are auto-lowercased.
  * These restrictions apply to module path/names in `require` calls of other modules also
  * **Note:** these extra restrictions apply only to the relative-path provided to the api, the sandboxed scripting directory; aka installation directory has no such restrictions.
  * These extra restrictions are to make cross-platform mods simpler; eg windows file-access is not case-sensitive unlike linux, and have different restrictions on what valid file paths can contain etc.

## Additional "Boundless" standard-library additions and modifications

The following APIs have been customised or added into the Boundless version of Lua.

* `print` will auto-redirect to the standard game-loggers.

### extended file-system access:

* `os.exists(path:string) -> boolean` ; return true if path exists; will always return false for a path outside the scripting sandbox
* `os.mkdir(path:string) -> boolean` ; create new directory (parent directory must exist), return true if directory was created or already existed, will always return false for a path outside the scripting sandbox
* `os.isfile(path:string) -> boolean` ; will always return false for a path outside the scripting sandbox
* `os.isdir(path:string) -> boolean` ; will always return false for a path outside the scripting sandbox
* `os.dir(path:string) -> iterator` ; returning 2 values `{ name:string, isdir:boolean }` ; behaviour "undefined" if modifying the directory during iteration (may/may not iterate changes etc)

  Example:

  ```lua
  -- print the names of all files/folders in the root directory of the scripting sandbox
  -- will never never iterate directories outside the scripting sandbox
  for name, isdir in os.dir(".") do
    print(name, isdir)
  end
  ```

### extended "clock" access:

* `os.hrtime() -> number` ; returns the number of seconds since the server started running; this differs from the Lua stdlib os.clock() which is cpu-time since the server started instead.
* `os.utctime() -> number` ; returns the number of seconds since the unix-epoch
* `os.worldtime() -> number` ; returns the world-time of the server (at sub-frame resolution)

### intervals/timeouts:

* `os.setInterval(function, milliseconds:number) -> id:number` ; execute the function every "millisecond"s; returns an `id` that can be passed to os.clearInterval
* `os.setTimeout(function, milliseconds:number) -> id:number` ; invoke the function only once; returns an `id` that can be passed to os.clearTimeout
* `os.clearInterval(id:number) -> boolean` ; return true if there was an interval with the given id stopped.
* `os.clearTimeout(id:number) -> boolean` ; return true if there was a timeout with the given id stopped.
  * the function callbacks will be called from a C-context via a `pcall` so any errors will only terminate the function execution, and not the whole script.

  Example:

  ```lua
  -- running the start function will start an interval logging "hi" every second, with the end function clearing the interval.
  local id
  function end()
    os.clearInterval(id)
    id = nil
  end
  function start()
    if not id then
      id = os.setInterval(function () print "hi" end, 1000)
    end
  end
  ```

* `os.intervals() -> iterator` of `{ id:number, function }` of registered intervals. This iterates a copy of the intervals so additions made after start of iteration will not be seen. Removals however, will not be iterated. This makes it safe to use this to iterate and remove all intervals.
* `os.timeouts() -> iterator` of `{ id:number, function }` of registered timeouts. As per `os.intervals()` regarding details.

## Boundless exposed library

### Event system

* `boundless.addEventListener(event:integer, function, priority=0.0) -> boolean` ; return true if the function was not already registered. Registers the function to be called every time the given event fires. Order that handlers will be executed is based on priority, and within an equal-priority class, by order of registration.
* `boundless.removeEventListener(event:integer, function) -> boolean` ; return true if the function was registered (and now removed); this must be the exact same function/closure.
* `boundless.events.onEnterFrame : number` ; event fires every "frame" of the server before any game systems run; receives a single argument for the worldTime of the current execution frame (wont be exactly the same as `os.worldtime()` which is sub-frame resolution)
* `boundless.events.onBlockPlaced : playerEntity, blockCoord` ; fires when a player places a block
* `boundless.events.onBlockRemoved : playerEntity, blockCoord, blockTypeData, blockColorIndex, toolUsedItemTypeData` ; fires when a player breaks a block

  Example:

  ```lua
  function tick(worldTime)
    print("Server is about to start a frame w/ worldTime " .. worldTime)
  end
  function init()
    boundless.addEventListener(boundless.events.onEnterFrame, tick)
  end
  function deinit()
    boundless.removeEventListener(boundless.events.onEnterFrame, tick)
  end
  ```

  * running the `init` function will register the tick function to be called at the start of each server frame, and the `deinit` will unregister it again; calling `init()` multiple times will have no effect as the handler is already registered, and `deinit()` will similarly have no effect if the handler is not registered.
  * **Note:** if you reload the module containing this code, the handler will not be automatically unregistered, loading a module in the scripting environment is the equivalent of a `dofile()` Lua call, you must unregister things yourself.

* `boundless.eventListeners(event:integer) -> iterator` of `{ function, priority }` in the order that they will be called for the given event. This iterates a copy of the registered events so that it can be easily used to unregister all events without worrying about iterating whilst unregistering. On the other hand, this means it will not iterate events added after iteration has begun. If however during iteration one of the events yet to be iterated is removed, it will not be returned by the iterator.

### Coordinate system types

* Boundless wrapping-worlds requires some more careful use of coordinate systems that make sense in a wrapping-world; these are exposed to the Lua scripts (Partially! its an on-going task to expose all features)
* So far exposed, are the coordinate/position types and delta(vector) types

  they all share the same templated apis:
  * `ChunkCoord` and `ChunkDelta` for chunk coordinates (integer)
  * `BlockCoord` and `BlockDelta` for block coordinates (integer)
  * `WorldPosition` and `WorldDelta` for world coordinates (floating point)
  * `PlotCoord` and `PlotDelta` for (3d) plot coordinates (integer)
  * `PlotXZ` and `PlotXZDelta` for (2d) plot coordinates (integer)

  these all have "Unwrapped" versions, like `UnwrappedChunkCoord` which represent a coordinate that may not be valid for the wrapped coordinate system, eg a position larger than the size of the world or a delta which is too long.
  a delta represents the difference of two coordinates, defined uniquely as the shortest difference of two coordinates in the range `[-worldsize/2, worldsize/2)`.

* `boundless.ChunkCoord()` et.al act as "constructors", and either take no-arguments (default-construct 0's), a single argument of some other equal-or-lower-type (eg for `ChunkCoord`, accepts any other wrapped-position type `BlockCoord`, `WorldPosition`, etc, but wrapped-types cannot be constructed from an unwrapped type directly), and for the unwrapped versions can set the x(y)z values directly. "Cast" construction will do appropriate conversions/scalings.

  Example:

  ```lua
  -- assuming we have a WorldPosition with values { -1.5, 0, 17.35 } named "pos" then:
  print(boundless.ChunkCoord(pos)) -- prints { -1, 1 } as "pos" is within the chunk at coordinate { -1, 1 }
  print(boundless.BlockCoord(pos)) -- prints { -2, 0, 17 } as "pos" is within the block at coordinate { -2, 0, 17 }
  print(boundless.WorldPosition(boundless.ChunkCoord(pos))) -- prints { -16, 0, 16 } as the lowest valid WorldPosition within the chunk at { -1, 1 }
  ```

* you cannot construct a wrapped-type with explicit x(y)z values as doing so would allow producing an invalid coordinate/delta, you may only directly construct an "Unwrapped" version.
* all vector-like types allow indexing as a Lua table, or with x/y/z named properties. y is unwrapped and always writable; for Unwrapped types xz are also writable.

  Example:

  ```lua
  local p = boundless.UnwrappedWorldPosition(10, 20, 30) -- { 10, 20, 30 }
  p[1] = 40 -- p is now { 40, 20, 30 }
  p.y = 50  -- p is now { 40, 50, 30 }
  ```

* `vector:copy()` produces a copy of the vector
* `vector:withY(y:number)` can be used on all types, giving a copy of the type with a different y value
* `vector:withYOffset(offset:number)` is equivalent to `coord.withY(coord.y+offset)`

  Example:

  ```lua
  local p = boundless.WorldPosition() -- { 0, 0, 0 }
  p.y = 10 -- p is now { 0, 10, 0 }; even through wrapped, y is always mutable
  print(p:withYOffset(10)) -- prints { 0, 20, 0 }, and p is still { 0, 10, 0 }
  ```

* deltas can be negated, and always produces an unwrapped delta (in the general case, a delta negated is not necessarily a valid wrapped delta in the extreme case of a most negative wrapped delta whose negation is itself, not the positive version)
* (x+y) vectors can be added subject to rules:
  * types must match; a world-position cannot be added to chunk-delta
  * you cannot add two positions together
  * adding a delta and a position of wrapped or unwrapped types always produces an unwrapped position type
  * adding a delta to another delta of wrapped or unwrapped type always produces an unwrapped delta type
* (x-y) vectors can be subtracted subject to rules:
  * types must match
  * you cannot subtract a position from a delta (positions cannot be negated)
* (x*s, s*x) scaling of deltas is subject to more intricate rules:
  * wrapped or unwrapped deltas can be scaled by integers, producing an unwrapped delta (cannot, in general, scale by a floating point value without breaking the rules of the wrapped universe)
  * wrapped deltas can be scaled by floating point values (-1,1] producing a wrapped delta; only for World-deltas which are floating point to begin with.
  * unit-length wrapped deltas can be scaled by any floating point value producing an unwrapped delta; only for World-deltas which are floating point to begin with.

* `boundless.wrap(coord/delta)->wrapped-coord/delta` ; is responsible for taking unwrapped positions/deltas and producing correctly wrapped versions that can then be provided to the boundless apis.

  Example:

  ```lua
  local p = boundless.UnwrappedChunkCoord(1000, 1500)
  print(boundless.wrap(p)) -- prints the ChunkCoord { 40, -36 } [ assuming a worldSize=192 chunks which is the case for creative servers ]
  ```

* `boundless.sqlength(WorldDelta) -> number` ; returns the squared length of the given wrapped-worldposition delta; cannot be used on unwrapped types.
* `boundless.distance(WorldPosition,WorldPosition) -> number` ; returns a distance between two positions that respects the wrapped coordinate system (aka minimal distance). equivalent to `math.sqrt(boundless.sqlength(boundless.wrap(pos1 - pos2)))`

  Example:

  ```lua
  -- assuming a WorldPosition p1 = { 1535, 0, 0 } and a WorldPosition p2 = { -1536, 0, 0 }
  print(boundless.distance(p1, p2)) -- prints 1 [ assuming a worldSize=192 chunks!]
  ```

  * noting this is different than what you would get with a standard `sqrt((x1-x2)^2+(z1-z2)^2)` geometric distance due to the wrapped coordinate system as the two positions are actually only 1 metre apart for a world of that size.

### Chunk access

* `boundless.getChunk(ChunkCoord) -> chunk-weak-ref | nil` ; returns nil if the chunk is unloaded, the returned value is a weak-ref which may "expire" and must be "locked" before use.
* `boundless.loadChunk(ChunkCoord, function)` ; will call the function (possibly synchronously!!) with the chunk once loaded (synchronous if already loaded or in the deleted-cache); again a weak-chunk reference; however note that it is guaranteed that for the duration of the function callback, the chunk will not become expired as the server will hold a strong-reference to the chunk internally until the function execution has completed.
* `chunk:expired() -> boolean` ; return true if the chunk is known to be unloaded (but may unload at any point due to garbage-collection!)
* `chunk:lock() -> chunk-ref|nil` ; returns a "strong" reference to the chunk allowing the Lua script to keep the chunk in memory. Due to the presence of strong references in the Lua environment, weak references may be invalidated at almost any point in time due to GC operations which can occur between or Lua statements. If the chunk was unloaded by the time the `lock()` call occurs, then nil will be returned. Due to garbage collection, its not impossible that `chunk:lock()` would return `nil`, even if `chunk:expired()` returned true immediately before!
* `chunk:release()` ; for a "strong" reference to the chunk, will release the reference immediately without waiting for garbage-collection to occur and reap the reference automatically.

* `boundless.getChunkAnd8Neighbours(ChunkCoord) -> table of 9 weak-chunk-refs | nil` ; return nil if any of the 9 chunks is unloaded
* `boundless.loadChunkAnd8Neighbours(ChunkCoord, function)` ; function provided with a table of 9 weak-chunk refs once loaded (may occur synchronously); as with the normal boundless.loadChunk() these weak-chunk references are guaranteed to remain valid for the duration of the callback execution.

* **Note:** as with the interval/timeout api, callback functions are invoked via a "pcall" and any error will not abort the script, but only that particular function.

  Example:

  ```lua
  local strongref
  boundless.loadChunk(chunkpos, function (chunk)
    print("hello")
    print(chunk:expired()) -- always prints false; as stated the chunk may be a weak-ref here, but the scripting environment will ensure the chunk remains valid from the server-side executing the callback
    strongref = chunk:lock() -- strongref now holds a Lua-side strong reference to the chunk, so the chunk will be kept in memory until strongref is garbage collected (aka never!)
    -- strongref = nil -- this would allow the strongref to be garbagecollected, but will not occur at any well-defined point in time so may be a while!
    strongref:release() -- releases the reference immediately, so that assuming no-other strong reference exist, the chunk will be unloaded as soon as this function ends
  end)
  print("hi") -- this may be printed *after* "hello" is printed if the chunk was actually already in memory on the server!
  ```

### Block access

* **Note on block colors:** with the exception of some "Metal" kind props like doors; all blocks in Boundless have either a single color, or 255 colors available. For those blocks without color variations, the block-color is always "0". For those that do (even the metal special case), the "0" value encodes a "world-default color" with values [1,255] encoding explicit colors. This is all handled automatically, so any access to blocks from the Lua environment will always resolve 0 colors into the true color index for colored blocks, and setting block values will deal with mapping to the 0 color index where appropriate.

* `boundless.blockTypes` is a table of all blockType ids
* `boundless.liquidTypes` is a table of all liquidType ids
* (also `boundless.itemTypes` for itemType ids though there is no api for items yet)

* `boundless.BlockValues(type=0,meta=0,color=0,liquid=0) -> BlockValues` ; acts as a constructor of a set of block values. range-checking will occur, but there is no check of "validity" here, so you can freely construct BlockValues corresponding to invalid blocks that don't exist etc.
* BlockValue objects have freely writable fields `blockType`, `blockMeta`, `blockColorIndex` and `liquidMeta`
* `boundless.getBlockValues(p:BlockCoord) -> BlockValues|nil` ; read the block-values at the given coordinate. If the coordinate is out-of-bounds (above or below the the world) then will always return the default-constructed BlockValues corresponding to an empty block of AIR (even if the chunk is unloaded). nil will be returned if the chunk at this position is not loaded.
* `boundless.setBlockValues(p:BlockCoord, val:BlockValues, playerEntity=nil) -> boolean` ; returns true if the block at this position was updated
  * **Note:** will always return false if p.y is <= 0 or > 255 (does not allow removal of bedrock at the bottom of the world)
  * **Note:** will always return false if the chunk at this position * as well as its 8 neighbours * are unloaded, as changes to the block world will always require neighbouring chunks to ensure validity as many changes to the world must synchronously operate on neighbour blocks that may be in those neighbour chunks.
  * **Note:** will check for the "validity" of the block-values, so will not allow setting blocktypes that do not exist, or invalid blockMeta values for that particular block, or invalid block-colors for that particular block, or invalid liquidMeta. The system will however allow you to set "valid" liquidMeta, even if the resultant block does not permit containing any liquid (eg a full cube block of rock); in these cases it will just ignore the liquidMeta instead.
  * **Note:** beacon consoles, or campfires cannot be created with this function unless a valid player instigator is provided to do plot management, trying to set these types will always return false otherwise. If a player entity is provided, may still fail to set the block if it would require creating a new beacon, and the player has insufficient plots available
  * **Note:** this largely emulates players placing blocks, so cannot be used to create blocks like "ready to break" portal/warp blocks, and cannot be used to create invalid arrangements of blocks like extra-long pipes or very large portals/signs or floating torches etc.
  * **Note:** in the case that the blockType and blockMeta remain "unchanged", you are free to modify the block colors and the liquid at any position in the world even if the blockType/blockMeta correspond to an otherwise disallowed combination for the API.
  * **Note:** restrictions on blockMeta are to ensure scripts cannot insert "undefined-behaviour" data that may break systems or lead to breaks later in time if systems change and previously unused meta becomes meaningful etc.
  * **Note:** restrictions on liquidMeta are much more "free" as liquids are a stateless system so even if you can do "weird things" by spawning free-floating non-source liquids, nothing breaks as a result.
* `boundless.getBlockTypeData(type) -> BlockTypeData`. For looking up names and related Block-ids.
  * `bdata.name` : string ; the name-id of block, corresponds to an entry in boundless.blockTypes
  * fields: `id`, `rootType`, `embeddedType`, `analyticsType`, `craftedType`, `tetraType`, `octaType`, `bevelType`, `slopedType`, `hoeType`, `unhoeType`, `gleambowBase`, `undugBase`, `dugup`.

* `boundless.describeBlock(BlockValues) -> table`. Provides a high-level description of the blockType/blockMeta/blockColorIndex provided, mapping blockTypes to a "canonical" type and converting the blockType/blockMeta into a more descriptive set of properties dependent on the type of block. This acts to abstract away the implementation details of how data can be split between multiple blockTypes and the blockMeta. This function has "undefined" results (may be nil, may be just a bit bollocks) if provided with an invalid set of data in the BlockValues, but should always be valid for any blockValues gotten from the world itself.
  * `blockType` integer matching a value in boundless.blockTypes. In particular this is the "analyticsType" representing the fundamental block type (Discards tags like DUGUP/GLEAMBOW or variations like for FRIEZE blocks or LEDS, or texture-rotation variations, or all the variations of grass-block ids)
  * `embeddedBlockType` integer matching a value in boundless.blockTypes. If present, represents the block embedded within. This only applies to resource-seam blocks and grass blocks, corresponding to the rock/glacier type, or soil type.
  * `blockColorIndex` integer, present for coloured blocks; represents the default-mapped colour of the block (is never 0)
  * `embeddedBlockColorIndex` integer, present when embeddedBlockType is present; the colour of the embedded block.
  * `grassHeight` integer, present for grass blocks, a 0-3 values representing the long-grass height.
  * `dugup` boolean, present for blocks that have a dug-up variant; whether the block provided was dug-up
  * `gleambow` boolean, present for blocks that have a gleambow variant; whether the block provided was from a gleambow meteorite
  * `textureRotation` integer, present for blocks that have texture rotation; a 0-3 value.
  * `rotation` integer, present for mesh-blocks that have rotation; a 0-3 value.
  * `placeOnFace` integer, present for blocks like torches that can be put onto walls, and any plant that gets put down onto the ground. A 0-5 value representing which axial-face it has been put onto.
  * `connections` integer, present with different meanings depending on the type. For a spark-link (`MACHINE_PIPE_BASE`), represents which of the 6 directions is connected visually. For a modular sign, represents an encoding of what sign pieces are around it that it connects to. For a Beam represents whether the end pieces of the beam have an extra arm-connector created visually. For a Pole, represents which of the 4 side directions have a beam connecting to it.
  * `powered` boolean, present for spark links, whether the spark link can provide spark.
  * `crate` boolean, present for modular sign pieces, distinguishing the unjoined "crate" type versus the joined interactable type.
  * `wild` boolean, present for crops. Whether the crop is "wild" or not.
  * `stage` integer, present for non-wild crops. What integral stage of growth the crop is at, a 0-31 value with 31 representing "fully simulated" (eg withered for ornamental crops)
  * `fertilizerSkill` boolean, present for non-fully-simulated non-wild crops. Whether the crop has been fertilised by a user with the specific fertilising skill for extra boost.
  * `fertilizerGrade` integer, present for non-fully-simulated non-wild crops. A 0-2 value, 0 meaning not fertilised.
  * `auto` boolean, present for doors and trap-doors. Whether the door will auto-close/auto-open after user interaction.
  * `autoClose` boolean, present if `auto` is true. Distinguish between auto-open and auto-close doors.
  * `open` boolean, present for doors and trap-doors. Whether the door is currently open.
  * `right` boolean, present for doors, whether the door opens on its right axis instead of the left.
  * `high` boolean, present for trap-doors, whether the trap-door sits high in the block space or not (opening down, rather than up)
  * `slopeChiselled` integer, present for slope-chiselled blocks. A bit mask of which corners have yet to be chiselled.
  * `active` boolean, present for meteorite blocks to distinguish active/inactive meteorite blocks (indestructible/breakable)
  * `squareChiselled` integer, present for square-chiselled blocks and square size-chiselled blocks. A bit mask of which corners have yet to be chiselled, or for size-chiselled blocks, which arms are present as a bit-mask.
  * `bevelChiselled` integer, present for bevel-chiselled blocks and bevel size-chiselled blocks. A bit mask of which corners have yet to be chiselled, or for size-chiselled blocks, which arms are present as a bit-mask.
  * `sizeChiselled` integer, present for size-chiselled blocks representing the size of the chiselling as a 0-3 value. In addition, one of squareChiselled or bevelChiselled will be present.
  * `portalDirection` integer, present for open portal and warp blocks, representing the orientation of the portal plane as a 0-2 value.
  * `portalState` string, one of `idle`, `complete`, `loading`, `breakable`, `open` or `dormant` distinguishing the block-states of the portal/warp blocks.
  * `portalComplete` boolean, represents if the portal is rectangular or not in its idle state, for portal conduits only, not warps.
  * `index` integer, present for multi-block machines (other than furnace which is distinguished by two block types already for the crucible and furnace separately). Is a mesh index, with 0 representing the "crates" and 1-N representing which mesh piece is used for rendering.
  * `variation` integer, present for types like the LED blocks and Marble Frieze which have variation types that can only be reached via change-type chiselling.
* `boundless.undescribeBlock(nil|table) -> BlockValues w/ liquidMeta of 0 | nil`. Opposite of describeBlock: constructing a BlockValues from description. Will not throw any errors, only returning nil if input is invalid (does not exhaustively check validity). For an input of `nil` will return the "air-block". Even if this function returns a valid BlockValues, this does not mean it is valid for a boundless.setBlockValues which has extra restrictions around what kind of data is allowed to be set which can include neighbouring blocks also.

* `boundless.describeLiquid(BlockValues|liquidMeta number) -> table`. Similarly, but for liquid part of the BlockValues.
  * `liquidType` integer matching a value in boundless.liquidTypes
  * `source` boolean for whether this is a source-block of liquid
  * `userPlaced` boolean for whether this is a source-block placed by a user.
  * `distance` integer representing the distance from the source block of the liquid (source blocks are always 0, as are vertical falling non-source-blocks of liquid)
* `boundless.undescribeLiquid(nil|table) -> liquidMeta number | nil`. Opposite of describeLiquid; constructing a liquidMeta from description. Will not throw any errors, only returning nil if input is invalid (does not exhaustively check validity). for an input of nil will return the "no-liquid" liquidMeta value.

### Item access

* `boundless.Item(itemType, count=1, color=0) -> Item` can only accept a subset of all the boundless.itemTypes; specifically if there is a "DUGUP" version, then that is the one that is allowed to be used. This is a more restrictive API than for blocks, and wont allow creation of Items for types that are not "in-game", so cannot create items that exist in the gamedata but are not yet released (This is not necessarily on purpose, but a side-effect of the way we avoid creating invalid Items (like non-dugup block items)). This function will throw an error if the item type is not valid, the count is too large for that particular type of item, or the color is out-of-bounds for the palette of the item. the 0 color is always valid and for coloured items will map to the world-default color.
* `boundless.Item(Item | read-only inventory item) -> Item` does a deep-copy of given item.

* Items have the following fields
  * `id` : number, the ItemType
  * `count` : number, the primitive count of the item, may be set but will throw an error if the count is too high for the item type (and cannot be set to a value less than 1)
  * `color` : number, the colour of the item, may be set but will throw an error if the colour is out-of-bounds for the item's palette (most items have either no colour variations, or a single fixed variation (only 0 and possibly 1 is valid), or 255 color variations, some special cases like metal doors have 5 instead). Setting to a value of 0 for coloured items will set to the world-default so is always a valid argument to the constructor / setter.

* Smart-stacks should use the `ItemType boundless.itemTypes.COLLECTION`, with access to the sub-items via array-indexing with indices 1 to 9 inclusive. Rules about the validity of a smart-stack are checked on usage of the Item (storing into an inventory, spawning as a drop in the world), so it should not be possible to create mixed-smart stacks not normally possible in-game, or nested smart-stacks, or empty/singular smart-stacks (smart-stacks must have more than 1 contained item).

* `item:copy() -> Item` ; produces a deep-copy of the item or inventory-read-only-item, same as calling `boundless.Item(theItemToDeepCopy)`
* `item:capacity() -> number` ; the number of items that can be stored (0 if not a smart-stack), this is different than `#item` which is the number of non-nil sub-items in the smart-stack.

  Example:

  ```lua
  stack = boundless.Item(boundless.itemTypes.COLLECTION)
  stack[1] = boundless.Item(boundless.itemTypes.SOIL_SILTY_BASE_DUGUP, 50)
  stack[2] = boundless.Item(boundless.itemTypes.SOIL_PEATY_BASE_DUGUP, 50)
  copy2 = stack[2]:copy() -- same as boundless.Item(stack[2])
  stack[2].color = 10 -- copy2.color will still be 50
  stack[1] = nil -- remove from the smart stack

  -- the lua-side Items largely act like normal lua objects, so the following is expected behaviour
  stack[1] = stack[2]
  stack[1].color = 10 -- stack[2].color is now also equal to 10 since stack[1] and stack[2] reference the same lua Item object
  ```

  the API will not prevent creating nested smart-stacks directly etc, but uses of the items will throw errors eg if you try to set an inventory slot to such an item.

* `boundless.getItemTypeData(type) -> ItemTypeData`. For looking up names and details of Item types, noting every block is also an item
  * `idata.name` : string ; the name-id of item, corresponds to an entry in boundless.itemTypes
  * fields: `id`, `maxStackSize` (the maximum valid count for the item), `colors` (the number of colors the item has, 0 if uncoloured)

* `boundless.spawnItem(Item, position:WorldPosition) -> Entity` ; creates a "block-drop" entity using the giving Item and position

### Connection access

* a connection exists for each connected player (whether they are on the world, or just looking at the world through a portal)

* `boundless.getConnection(connectionId | entityId | entity | characterId) -> connection-weakref | nil` ; access the connection with the given id, corresponding to the entityId of the given player on the world, or corresponding to the given characterId of a connected player. this is a weak-reference that may expired, but there is no ":lock()" method like with chunks, and no worries about the reference expiring in between Lua statements as the Lua environment will never hold garbage-collectable strong references.
* `connection:expired() -> boolean` ; true if the connection is no longer available
* connections have accessible fields, which will all be `nil` if the connection is no longer available
  * `id`
  * `username`
  * `characterId`

  Example:

  ```lua
  function logplayers()
    for c in boundless.connections() do
      print(c.username)
    end
  end
  ```

* `boundless.ConnectionId(number) -> ConectionId` ; acts as a constructor for a ConnectionId from a raw number.
* `boundless.CharacterId(number) -> CharacterId` ; acts as a constructor for a CharacterId from a raw number.

### Entity access

* `boundless.getEntity(entityId | connectionId | characterId | connection) -> entity-weakref | nil` ; access the entity with the given id, or corresponding to the player of the given connectionId/characterId. As with connections is a weak-ref that may expire, and as there are no strong-references garbage-collectable in the Lua environment, do not need to worry about expiration between Lua statements.
* `boundless.getEntity(blockCoord, damage-entities=false) -> entity-weakref | nil` ; access a block-entity by any blockCoord that it covers (eg machines/portals cover multiple blocks). The damage-entities parameter allows for access to "transient damage block-entities", eg when you hit a soil block an entity is created for that block until its damage is reset and are normally not accessible. In the case of "most" multi-block entities, damage is still done at the per-block level, and this is managed by creation of a damage-block entity for that individual block in addition to the multi-block entity underneath, in these cases you would only be able to access that damage-entity by setting the boolean parameter to true, or else you would access the multi-block entity instead.
* `entity:expired() -> boolean` ; true if the entity is no longer available
* `entity:isDead() -> boolean` ; returns true if the entity is dead. This is not strictly the same as checking health==0 for current health, as in the middle of an update an entity may temporarily go to 0 health before gaining health due to some other effect. The entity only dies towards the end of an update when no more effects can occur that may bring the entity back from the brink of death.
* entities have the following accessible fields, which will all be nil if the entity is no longer available
  * `id` : EntityId
  * `name` : string | nil, name of the entity
  * `position` : WorldPosition, may also be set. setting the position is equivalent to a "forced teleport" and can move entities inside of blocks invalidly etc etc. Can only be set for entities which move around, and not entities like portals or blocks. If set on a player will cause a mis-prediction for that players client. Note that the position vector itself is immutable, so you cannot directly do `entity.position.y = 10` (will throw an error), but that you should instead do something like `entity.position = entity.position:withY(10)` or otherwise `p = entity.position:copy(); p.y = 10; entity.position = p`
  * `facing` : number, the (xz-plane) angle of the entity in the world
  * `movementState` : number | nil, (flying,walking,creeping,running) see boundless.movementStates
  * `health` : number | nil, may also be set. nil if the entity has no health at all
  * `maxHealth` : number | nil, nil if the entity has no health at all
  * `text` : string | nil, for a modular-sign, the text/additionalText will be nil if the sign has not been "locked" yet (by setting text at least once). A locked sign (normal boundless game rules) is such that it will no longer re-shape/merge with neighbouring signs and allows signs to be placed side-by-side and can only be entirely destroyed. Internally, locking a sign for the first time replaces its Entity with a new one, but the lua API will silently update the provided entity reference to point to the new one automatically in this case.
  * `additionalText` string | nil, the additional text on a sign when interacted with; as above for modular signs and locking.
  * `textStyle` number | nil,  `0,1,2` value representing "normal", "bold", "italic"; usable even if sign has no text assigned yet. if the sign is not yet locked the textStyle/textAlign will be reset if the entity is then destroyed/re-created by removing/adding sign blocks in the group.
  * `textAlign` number | nil,  `0,1,2` value representing "centre", "left", "right"; as per textStyle

  Example:

  ```lua
  function logplayerpositions()
    for c in boundless.connections() do
      local e = boundless.getEntity(c.id)
      if e then
        print(c.username .. " is at position " .. e.position)
      end
    end
  end
  ```

  * `inventory` : table, a table of all the inventories the entity has (eg players have a backPack, furnaces have a fuelInput, machineInput and a machineOutput inventory) this table is iterable by either pairs or ipairs, and can be indexed numerically or by name. If the entity is no longer present, will return nil. Access to fields of the inventory table will also return nil if the entity is no longer present.
    * entity inventories are numerically indexed tables, and extra checking of the item types will be done on insertion into the inventory to prevent cases like adding soil into a portal-token inventory.
    * access to items within the inventory are "read-only" copies of the data owned by the game-engine, access to sub-items (smart-stacks/ammo) of these items will also be "read-only" copies; if you want to modify the data, either construct a lua-Item via boundless.Item(theReadOnlyItem) or use the :copy() member function to get an editable lua-Item copy to use. The API is otherwise the same, so if you are only reading/iterating the item data, you don't need to take lua-copies of the Items.

    * `inventory.type:capacity() -> number` ; the number of slots in the inventory available for use. This is different than `#inventory.type` which is the number of non-nil slots. If the entity is no longer available, will return 0

    Example:

    ```lua
    for name,inventory in ipairs(entity.inventory) do
        print("Entity has an inventory "..name.." with "..#inventory.." items in it, and space for a total of "..inventory:capacity().." items")
        print(inventory[4].color) -- print the color of the 4th item (assuming not nil)
        -- inventory[4].color = 20 -- not allowed, inventory[4] is a read-only copy of the 4th item in the inventory
        local newv = inventory[4]:copy() -- newv is now a "proper" lua copy of the Item that can be manipulated
        newv.color = 20
        inventory[4] = newv -- copy the lua Item back into the inventory to update
        print(inventory[4].color) -- prints 20 now
    end
    ```

* `boundless.EntityId(number) -> EntityId` ; acts as a constructor for a EntityId from a raw number.

* `boundless.archetypes` ; table of all entity archetype ids in Boundless
* `boundless.movementStates` ; table with flying,walking,creeping,running movement state values
* `boundless.spawnCreature(archetypeID, position:WorldPosition, facing:number=0) -> Entity-weakref | nil` ; can only be used to create creature entities, eg block entities cannot be created this way, and only via placing blocks instead.
* `boundless.killEntity(entity:Entity-weakref)` ; deals deathly damage to the entity (assuming is an entity that can be damaged)

### Chat access

* `boundless.guiIcons` a table of all the icons available to use (excluding glyphs)
* `boundless.guiGlyph(bitset:integer) -> table|nil` given a 12-bit integer of which "arms" should be present in the 3x3 boundless-glyph, map to the icon and rotation/flip required to render it. Not all glyphs (only around half) are present in the game-data as only those used by the game are available to be rendered.
* `boundless.guiColors` a table of named colours used in the GUI.
* `boundless.showPlayerLog(entity|entityId|connection|connectionId|characterId, msgType=boundless.guiActionLogMsgTypes.CUSTOM, msgArgs...:string, customisation={})` ; show an "action log" to the player of the given type using the given arguments. The possible log message types are listed in `boundless.guiActionLogMsgTypes` and arguments "undocumented" as it stands. `boundless.guiActionLogMsgTypes.CUSTOM` may be used with 2 additional arguments for a fully customised log (unlocalised) using the 2 arguments for the title text and sub-title text; as long as the second argument is a string, you can omit the msgType parameter and it will be implied as CUSTOM to simplify general usage. The customisation table must come last and may have the following fields:
  * `icon : string|table` either string texture, or table of `{ glyph: string, rotate: integer=0, flip: boolean=false }` to set the rotation and flip at the same time (eg using `boundless.guiGlyph(..)`)
  * `iconRotate : number` angle in degrees to rotate the icon (can be used to access more "glyphs" as rotations are not present in the set of available glyphs), only accepts 90-degree rotation increments.
  * `iconFlip : boolean` flip on x-axis, in combination with iconRotate can get all meaningful transformations of a square icon; noting that the flip occurs before the rotation. Note that iconRotate/iconFlip are "in addition" to whatever flip/rotate value may exist in the icon table field, so iconFlip=true will always flip the icon on the x-axis, after if it has possibly already been flip/rotated due to the values in the icon table.
  * `iconColor` the colour of the icon to use, this may be a named colour from `boundless.guiColors` table, or an integer sRGB value, eg: `0xff0000` for full-red.
  * `titleFont : string` the font to use for the title, one of `boundless.guiFonts.italic`, `boundless.guiFonts.normal`, `boundless.guiFonts.bold` only
  * `titleColor`
  * `subtitleFont : string`
  * `subtitleColor`
  * `link : boolean` if true, a small bar will be shown above the log entry joining it to the log line above (eg the log messages when joining a world that show info about the world as separate log lines)
  * `linkColor` only makes sense if "link" is set to true

  The default styling differs between different `guiActionLogMsgTypes` types, but for the case of a CUSTOM (default) message type, the defaults are equivalent to a customisation object:

  ```lua
  { icon = "", -- no icon
    iconFlip = false,
    iconRotate = 0,
    iconColor = boundless.guiColors.colorMidGrey,
    titleColor = boundless.guiColors.colorWH, -- white
    titleFont = boundless.guiFonts.normal,
    subtitleColor = boundless.guiColors.colorWH,
    subtitleFont = boundless.guiFonts.italic,
    link = false,
    linkColor = boundless.guiColors.colorWH }
  ```
  setting a field in the customisation object will override whatever defaults exist for that message type.

  Example:

  ```lua
  boundless.showPlayerLog(c, "Welcome", "to the bright side...",
          { icon = boundless.guiIcons.boundless,
            iconColor = boundless.guiColors.boundlessred })
  boundless.showPlayerLog(c, "But mainly", "to the dark side!",
          { icon = boundless.guiGlyph(0xec4),
            iconColor = 0, -- black
            titleColor = 0,
            subtitleColor = 0,
            subtitleFont = boundless.guiFonts.bold,
            link = true });
  ```
