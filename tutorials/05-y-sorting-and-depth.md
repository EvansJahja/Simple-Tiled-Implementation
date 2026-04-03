# Y-Sorting and Depth

This tutorial assumes you have completed the [Introduction to STI](01-introduction-to-sti.md) and ideally [Working with Objects and Entities](04-objects-and-entities.md).

Let me set the scene: you've got a beautiful top-down game world. Trees dot the landscape, fences line the paths, and your player character is running around exploring. But something's wrong. When the player walks behind a tree, they're still visible - drawn right on top of it. When they walk in front, same deal. It's like your character is always floating on a layer above everything else.

This breaks the illusion. In a real world, when you walk behind a tree, the tree blocks your view. Walk in front, and you're visible. This is called **depth** or **draw order**, and getting it right is what separates "that looks like a game" from "that looks like paper cutouts on a board."

The solution is called **Y-sorting**, and once you understand it, you'll wonder how you ever lived without it.

## The Drawing Problem

Let's think about how 2D rendering works. In LÖVE (and most 2D engines), whatever you draw last appears on top. Draw the background first, then the trees, then the player, and the player will be visible in front of everything.

STI draws layers in order by their index. Layer 1 first, then layer 2, and so on. Within a tile layer, tiles are drawn in a consistent order (top-to-bottom, left-to-right). This works great for flat scenery.

But entities on a custom layer? They draw in whatever order they're stored in your array. If you added the player first and the tree second, the tree is always on top. Swap the order, and the player is always on top. Neither is correct! We need the order to *change* based on where things are.

## The Insight: Y Position = Depth

Here's the key insight that makes everything click. In a top-down game viewed from a slight angle (the classic 2D RPG perspective), things lower on the screen are "closer" to the camera. Things higher on the screen are "further away."

Think about it: if you're looking at a field, trees in the distance (top of screen) are behind trees in the foreground (bottom of screen). And when your character is at the top of the screen, they're in the distance too - so trees lower than them should be drawn on top.

The Y coordinate on screen corresponds directly to depth in the game world. Lower Y = further back. Higher Y = further forward.

So the rule is simple: **sort everything by Y position, then draw in that order.** Things with lower Y get drawn first (they're in the back), things with higher Y get drawn last (they're in the front, visible on top).

## The Basic Implementation

Let's make it happen. If you have an entities array in your custom layer, just sort it before drawing:

```lua
layer.draw = function(self)
	-- Sort entities by Y position
	table.sort(self.entities, function(a, b)
		return a.y < b.y
	end)

	-- Draw in sorted order
	for _, entity in ipairs(self.entities) do
		love.graphics.draw(entity.sprite, entity.x, entity.y)
	end
end
```

That's it. That's Y-sorting. Every frame, we sort the array so things with smaller Y values come first, then we draw them in order. When your player walks up past a tree (lower Y), they'll be drawn before the tree, so the tree appears on top. Walk down past it (higher Y), and the player is drawn later, appearing on top.

"But wait," you might be thinking, "sorting every frame sounds expensive!" It can be, but for anything less than thousands of entities, modern computers chew through it without breaking a sweat. Lua's `table.sort` is actually quite fast, and games have been doing this since the SNES days with far less computing power. Don't optimize until you actually have a problem.

## The Sprite Anchor Gotcha

Alright, there's a catch, and it's an important one. Where exactly is your entity's position?

Most sprites aren't anchored at their top-left corner for gameplay purposes. Think about a character sprite - the "position" should be at their feet, not their head. When they stand on a tile, their feet are on that tile, even if their head extends above it.

If you're sorting by `entity.y` but that Y represents the sprite's top edge, you'll get weird results. A tall character standing next to a short bush will appear behind the bush even when they should be in front, because the character's top is higher on screen than the bush's top.

The fix is to sort by where the entity's feet are - the bottom of the sprite:

```lua
table.sort(self.entities, function(a, b)
	return (a.y + a.height) < (b.y + b.height)
end)
```

Or, if you've been following along with our tutorials and your entities use origin offsets (where the Y position already represents the feet), then `a.y` is already the right value. This is why we set up our player with the origin at their feet back in Tutorial 01 - it makes Y-sorting work naturally.

## Trees and Players, Living in Harmony

Let's build a practical example. We have a player who can move around, and some trees placed in Tiled that the player should be able to walk behind and in front of.

First, let's set up our map and create both the player and trees as entities in the same array:

```lua
function love.load()
	map = sti("map.lua")

	local layer = map:addCustomLayer("Sprites", 8)
	layer.entities = {}

	-- Load sprites
	local playerSprite = love.graphics.newImage("player.png")
	local treeSprite = love.graphics.newImage("tree.png")

	-- Create player from Tiled object
	local playerObj
	for _, object in pairs(map.objects) do
		if object.name == "Player" then
			playerObj = object
			break
		end
	end

	layer.player = {
		sprite = playerSprite,
		x      = playerObj.x,
		y      = playerObj.y,
		ox     = playerSprite:getWidth() / 2,
		oy     = playerSprite:getHeight() -- Anchor at feet
	}
	table.insert(layer.entities, layer.player)

	-- Create trees from Tiled objects
	for _, object in pairs(map.objects) do
		if object.type == "Tree" then
			local tree = {
				sprite = treeSprite,
				x      = object.x + object.width / 2,
				y      = object.y + object.height,  -- Bottom of tree
				ox     = treeSprite:getWidth() / 2,
				oy     = treeSprite:getHeight()     -- Anchor at trunk base
			}
			table.insert(layer.entities, tree)
		end
	end

	-- Hide original object layer
	for _, l in ipairs(map.layers) do
		if l.type == "objectgroup" then
			l.visible = false
		end
	end
end
```

Notice how both the player and trees use origin offsets that place the anchor at their feet (or trunk base). The Y position represents where they're "standing." This is crucial for correct sorting.

Now the draw function with Y-sorting:

```lua
	layer.draw = function(self)
		-- Sort by Y position (feet/base)
		table.sort(self.entities, function(a, b)
			return a.y < b.y
		end)

		-- Draw everyone in sorted order
		for _, entity in ipairs(self.entities) do
			love.graphics.draw(
				entity.sprite,
				math.floor(entity.x),
				math.floor(entity.y),
				0, 1, 1,
				entity.ox,
				entity.oy
			)
		end
	end
```

And the update function to move the player:

```lua
	layer.update = function(self, dt)
		local speed = 96 * dt
		if love.keyboard.isDown("w", "up") then self.player.y = self.player.y - speed end
		if love.keyboard.isDown("s", "down") then self.player.y = self.player.y + speed end
		if love.keyboard.isDown("a", "left") then self.player.x = self.player.x - speed end
		if love.keyboard.isDown("d", "right") then self.player.x = self.player.x + speed end
	end
```

Run this, and watch the magic happen. Walk your player up past a tree - they disappear behind it. Walk down past it - they pop out in front. The world suddenly feels *real*.

## Tiled Has Y-Sorting Too!

Here's a nice surprise: STI actually respects Tiled's built-in draw order setting for object layers. In Tiled, select an Object Layer and look at its properties. You'll find "Drawing Order" which can be set to "Top Down."

When you set this, STI automatically sorts objects in that layer by their Y position:

```lua
-- This is what STI does internally for "topdown" object layers:
if layer.draworder == "topdown" then
	table.sort(layer.objects, function(a, b)
		return a.y + a.height < b.y + b.height
	end)
end
```

This is fantastic for decorative objects that never move - position them in Tiled, set the layer to top-down draw order, and forget about it. But for dynamic entities like your player that move every frame, you still need manual sorting in your custom layer.

## The Layer Sandwich Alternative

"What if I don't want to sort every frame? Isn't there another way?"

There is! You can structure your layers to fake depth. Imagine this layer setup:

```
Layer 1: Ground (grass, dirt, paths)
Layer 2: Ground decorations (flowers, rocks)
Layer 3: Entities (player, NPCs, enemies)
Layer 4: Tree trunks (bottom half of trees)
Layer 5: Entities... wait, we already have that layer
```

Hmm, that doesn't work. The problem is that the player needs to be both behind AND in front of trees depending on position, and you can't be in two layers at once.

You could split trees into "trunk" and "canopy" layers, where trunks are below the player layer and canopies are above:

```
Layer 1: Ground
Layer 2: Tree trunks (always behind player)
Layer 3: Player/Entities
Layer 4: Tree canopies (always in front of player)
```

This creates a weird disconnect though - the player would always appear between the trunk and canopy. It kind of works for very specific art styles, but it's a hack. Real Y-sorting is more flexible and looks better.

The layer sandwich approach is really only useful when you want EVERYTHING at a certain height to be in front or behind - like having all rooftops above the player layer so you can walk "inside" buildings.

## When Sorting Gets Slow: Optimization Tips

If you've got hundreds of entities and performance becomes a concern (profile first!), here are some tricks:

### Only Sort When Something Moves

If most of your entities are stationary, only re-sort when positions change:

```lua
local needsSort = true

layer.update = function(self, dt)
	-- Movement code here
	if playerMoved then
		needsSort = true
	end
end

layer.draw = function(self)
	if needsSort then
		table.sort(self.entities, function(a, b)
			return a.y < b.y
		end)
		needsSort = false
	end

	for _, entity in ipairs(self.entities) do
		-- draw
	end
end
```

### Separate Static and Dynamic

Keep entities that never move in a separate, pre-sorted array. Only Y-sort the moving entities, then merge during drawing:

```lua
layer.draw = function(self)
	-- Only sort moving entities
	table.sort(self.dynamicEntities, function(a, b)
		return a.y < b.y
	end)

	-- Merge sorted arrays while drawing
	local si, di = 1, 1
	while si <= #self.staticEntities or di <= #self.dynamicEntities do
		local static = self.staticEntities[si]
		local dynamic = self.dynamicEntities[di]

		if not dynamic or (static and static.y < dynamic.y) then
			drawEntity(static)
			si = si + 1
		else
			drawEntity(dynamic)
			di = di + 1
		end
	end
end
```

This is more complex but can help if you have lots of static objects.

### Only Sort Visible Entities

If your world is huge, consider only Y-sorting entities currently on screen. Why sort things you're not drawing?

But honestly? For most games, just sorting the whole array every frame is totally fine. A modern computer can sort thousands of items in well under a millisecond. Don't prematurely optimize.

## The Complete Example

Here's everything put together - a player walking around a world with trees that correctly obscure the player based on depth:

```lua
local sti = require "sti"

function love.load()
	map = sti("map.lua")

	local layer = map:addCustomLayer("Sprites", 8)
	layer.entities = {}

	-- Sprites
	local playerSprite = love.graphics.newImage("player.png")
	local treeSprite = love.graphics.newImage("tree.png")

	-- Create player
	local spawnX, spawnY = 200, 200
	for _, object in pairs(map.objects) do
		if object.name == "Player" then
			spawnX, spawnY = object.x, object.y
			break
		end
	end

	layer.player = {
		sprite = playerSprite,
		x      = spawnX,
		y      = spawnY,
		ox     = playerSprite:getWidth() / 2,
		oy     = playerSprite:getHeight()
	}
	table.insert(layer.entities, layer.player)

	-- Create trees
	for _, object in pairs(map.objects) do
		if object.type == "Tree" then
			table.insert(layer.entities, {
				sprite = treeSprite,
				x      = object.x + object.width / 2,
				y      = object.y + object.height,
				ox     = treeSprite:getWidth() / 2,
				oy     = treeSprite:getHeight()
			})
		end
	end

	-- Player movement
	layer.update = function(self, dt)
		local speed = 96 * dt
		if love.keyboard.isDown("w", "up") then self.player.y = self.player.y - speed end
		if love.keyboard.isDown("s", "down") then self.player.y = self.player.y + speed end
		if love.keyboard.isDown("a", "left") then self.player.x = self.player.x - speed end
		if love.keyboard.isDown("d", "right") then self.player.x = self.player.x + speed end
	end

	-- Y-sorted drawing
	layer.draw = function(self)
		table.sort(self.entities, function(a, b)
			return a.y < b.y
		end)

		for _, entity in ipairs(self.entities) do
			love.graphics.draw(
				entity.sprite,
				math.floor(entity.x),
				math.floor(entity.y),
				0, 1, 1,
				entity.ox,
				entity.oy
			)
		end
	end

	-- Hide original object layer
	for _, l in ipairs(map.layers) do
		if l.type == "objectgroup" then
			l.visible = false
		end
	end
end

function love.update(dt)
	map:update(dt)
end

function love.draw()
	local player = map.layers["Sprites"].player
	local scale = 2
	local sw = love.graphics.getWidth() / scale
	local sh = love.graphics.getHeight() / scale
	local tx = math.floor(player.x - sw / 2)
	local ty = math.floor(player.y - sh / 2)

	map:draw(-tx, -ty, scale)
end
```

Run this with a map that has a Player spawn point and some Tree objects, and you've got proper depth sorting. Walk behind trees, walk in front of trees. The illusion of a 2D world with depth is complete.

## Wrapping Up

Y-sorting is one of those techniques that seems almost too simple when you first hear it. "Just... sort by Y? That's it?" Yep. That's it. And it transforms your game from "sprites on a flat plane" to "a world you can walk around in."

The key points to remember:

1. **Sort entities by Y position** before drawing each frame
2. **Anchor sprites at their feet** so Y position represents where they're standing
3. **Use origin offsets** in LÖVE's draw function to handle the anchor
4. **Include everything that needs depth** in the same sorted array - player, enemies, trees, NPCs, everything

With bump.lua or Box2D for collision, custom properties for entity data, and Y-sorting for depth, you've got all the core pieces of a polished 2D game. Go make something amazing!
