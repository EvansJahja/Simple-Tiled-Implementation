# Working with Objects and Entities

This tutorial assumes you have completed the [Introduction to STI](01-introduction-to-sti.md). We'll build on that foundation to explore how to work with Tiled objects and turn them into game entities.

In Tutorial 01, we used a single object named "Player" as a spawn point. We found it, grabbed its coordinates, and created our player there. Simple! But if you've been playing around with Tiled, you've probably noticed that objects can do so much more. What if we want multiple enemies? Treasure chests? NPCs with dialogue? Trigger zones that load the next level?

That's what this tutorial is all about. By the end, your Tiled maps will be doing most of the heavy lifting for you.

## What Can Objects Do?

Tiled supports several object shapes, and each one is useful for different things.

**Rectangle** is your bread and butter. Most entities - spawn points, chests, doors, enemies - work great as rectangles. The position and size translate directly into game coordinates.

**Ellipse** is perfect for circular areas. Think of an NPC's "notice radius" or a campfire's warmth zone. You get a center point and radii, which maps nicely to distance-based calculations.

**Point** is the simplest - just an X and Y. Great for spawn markers when you don't need size information. "Put an enemy here. Done."

**Polygon** lets you draw custom shapes. Irregular trigger zones, complex collision areas, or that weirdly-shaped pond the player shouldn't walk through.

**Polyline** is a series of connected points that don't close into a shape. These are fantastic for patrol paths! Draw the route an enemy should walk, and your code can follow it point by point. We'll get to that later - it's really cool.

## Two Ways to Find Your Objects

STI gives you two ways to access objects, and understanding when to use each will save you headaches down the road.

### The Global Approach: map.objects

Every object in your map gets a unique ID from Tiled. STI collects all of them into `map.objects`, keyed by that ID:

```lua
for id, object in pairs(map.objects) do
	print(id, object.name, object.x, object.y)
end
```

This is great when you're looking for one specific object (like our Player spawn in Tutorial 01) or when you need to process every object regardless of which layer it's on. But there's a catch: the IDs are assigned by Tiled and aren't predictable. You can't just say "give me object 5" and know what you'll get.

### The Organized Approach: Layer Iteration

Objects live in Object Layers, and you can iterate through a specific layer's objects:

```lua
local enemyLayer = map.layers["Enemies"]
for _, object in ipairs(enemyLayer.objects) do
	print(object.name, object.type)
end
```

This is usually the better approach. Why? Because you're probably already organizing your Tiled map with layers like "Enemies", "Collectibles", "Triggers", and so on. Your code can mirror that organization. Need to spawn all enemies? Loop through the Enemies layer. Need to set up all triggers? Loop through the Triggers layer. Clean and predictable.

## The Magic of Object Types

Here's where things get really powerful. In Tiled, every object has a `Type` field (called "Class" in newer versions of Tiled). This is your secret weapon for spawning different kinds of entities from the same layer.

Picture this: you have an "Entities" layer in Tiled with a bunch of objects. Some are enemies, some are chests, some are NPCs. Instead of creating separate layers for each (which gets messy fast), you just set their Type appropriately and let your code sort them out:

```lua
function love.load()
	map = sti("map.lua")

	local entityLayer = map.layers["Entities"]

	for _, object in ipairs(entityLayer.objects) do
		if object.type == "Enemy" then
			spawnEnemy(object)
		elseif object.type == "Chest" then
			spawnChest(object)
		elseif object.type == "NPC" then
			spawnNPC(object)
		elseif object.type == "Trigger" then
			createTrigger(object)
		end
	end
end
```

Now your level designer (which might also be you, wearing a different hat) can place enemies and chests on the same layer without your code getting confused. The Type field keeps everything sorted out.

## Custom Properties: Your Data Pipeline from Tiled

Alright, this is where Tiled transforms from "map editor" into "game data editor". Custom properties let you attach any data you want to objects, and it flows straight into your Lua code.

In Tiled, select an object and look at the Properties panel. Click the plus sign to add custom properties. Want this enemy to have 100 health? Add `health = 100`. Want it to move at a specific speed? Add `speed = 50`. Want it to say something when the player approaches? Add `dialogue = "Halt! Who goes there?"`.

In your code, all of this lives in `object.properties`:

```lua
local function spawnEnemy(object)
	local enemy = {
		x        = object.x,
		y        = object.y,
		width    = object.width,
		height   = object.height,
		health   = object.properties.health or 100,
		speed    = object.properties.speed or 30,
		dialogue = object.properties.dialogue,
		hostile  = object.properties.hostile or false
	}

	table.insert(enemies, enemy)
end
```

Notice how we use `or` to provide defaults? That's important. If someone forgets to set the health property, we don't want the game to crash - we want a reasonable default. This also means you don't have to set every property on every object. Only override the defaults when you need something different.

"But wait," you might be thinking, "couldn't I just store all this data in my code?" You absolutely could! But then every time you want to tweak an enemy's health or add a new patrol route, you're editing code, reloading the game, and testing again. With custom properties, you (or your level designer friend) can iterate on game feel without touching a single line of Lua. Open Tiled, change a number, save, reload. That's powerful, especially when you're in the "is this enemy too hard?" phase of development.

## Scaling Up: The Entity Factory

Alright, let's tackle something more ambitious. Our if/elseif chain works fine for a few entity types, but what happens when your game grows? Twenty different enemy variants, ten types of collectibles, a dozen interactive objects... that chain becomes a nightmare.

What we want is a cleaner way to say "here's how to create each type of entity" without a giant pile of conditionals. In programming circles, this is called a "factory pattern" - which sounds fancy but is really just "a table that tells us how to make things."

The idea is simple: we make a table where the keys are entity types (like "Enemy" or "Chest") and the values are functions that know how to create that type. When we encounter an object in Tiled, we look up its type in our table and call the corresponding function:

```lua
local sti = require "sti"

-- This table defines how to create each entity type
local EntityTypes = {
	Enemy = {
		create = function(object)
			return {
				type     = "Enemy",
				x        = object.x,
				y        = object.y,
				health   = object.properties.health or 50,
				speed    = object.properties.speed or 40,
				sprite   = love.graphics.newImage("enemy.png"),
				update   = function(self, dt)
					-- Enemy AI goes here later
				end
			}
		end
	},

	Chest = {
		create = function(object)
			return {
				type     = "Chest",
				x        = object.x,
				y        = object.y,
				contents = object.properties.contents or "gold",
				opened   = false,
				sprite   = love.graphics.newImage("chest.png"),
				open     = function(self)
					if not self.opened then
						self.opened = true
						return self.contents
					end
				end
			}
		end
	},

	NPC = {
		create = function(object)
			return {
				type     = "NPC",
				x        = object.x,
				y        = object.y,
				name     = object.name,
				dialogue = object.properties.dialogue or "...",
				sprite   = love.graphics.newImage(object.properties.sprite or "npc.png")
			}
		end
	}
}
```

The beauty here is that each entity type is completely self-contained. Want to add a new type? Just add another entry to the table. Want to change how NPCs are created? You know exactly where to look. No scrolling through a hundred-line if/elseif chain trying to find the right branch.

Now our loading code becomes wonderfully simple:

```lua
function love.load()
	map = sti("map.lua")
	entities = {}

	local layer = map:addCustomLayer("Sprites", 8)
	layer.entities = entities

	local objectLayer = map.layers["Entities"]
	if objectLayer then
		for _, object in ipairs(objectLayer.objects) do
			local factory = EntityTypes[object.type]
			if factory then
				local entity = factory.create(object)
				table.insert(entities, entity)
			else
				print("Unknown entity type: " .. tostring(object.type))
			end
		end
	end

	layer.draw = function(self)
		for _, entity in ipairs(self.entities) do
			if entity.sprite then
				love.graphics.draw(entity.sprite, entity.x, entity.y)
			end
		end
	end

	layer.update = function(self, dt)
		for _, entity in ipairs(self.entities) do
			if entity.update then
				entity:update(dt)
			end
		end
	end

	map:removeLayer("Entities")
end
```

That warning message for unknown types is a lifesaver during development. If you typo "Enemi" instead of "Enemy" in Tiled, you'll immediately know something's wrong instead of scratching your head wondering why your enemy didn't spawn.

## Making Enemies Walk: Polyline Paths

Remember those polylines we mentioned? Let's put them to work. This is one of my favorite Tiled tricks because it lets you design enemy behavior visually.

In Tiled, create a polyline object (it's the squiggly line tool). Draw the path you want your enemy to follow - click to add waypoints. Give it a name like "guard_route_1". The polyline data comes through as a list of points, each relative to the object's position:

```lua
-- When you encounter a polyline object, build a path from it
local path = {}
for _, point in ipairs(object.polyline) do
	table.insert(path, {
		x = object.x + point.x,
		y = object.y + point.y
	})
end
```

Now you can reference this path from an enemy. In Tiled, give your enemy a custom property like `patrol = "guard_route_1"`. Then in your enemy creation code, find that path and store it:

```lua
Enemy = {
	create = function(object)
		local enemy = {
			x = object.x,
			y = object.y,
			pathIndex = 1,
			path = nil,
			speed = object.properties.speed or 50
			-- ... other properties
		}

		-- Find the patrol path if specified
		if object.properties.patrol then
			for _, obj in pairs(map.objects) do
				if obj.name == object.properties.patrol and obj.polyline then
					enemy.path = {}
					for _, point in ipairs(obj.polyline) do
						table.insert(enemy.path, {
							x = obj.x + point.x,
							y = obj.y + point.y
						})
					end
					break
				end
			end
		end

		enemy.update = function(self, dt)
			if not self.path then return end

			local target = self.path[self.pathIndex]
			local dx = target.x - self.x
			local dy = target.y - self.y
			local dist = math.sqrt(dx * dx + dy * dy)

			if dist < 2 then
				-- Reached waypoint, move to next
				self.pathIndex = self.pathIndex + 1
				if self.pathIndex > #self.path then
					self.pathIndex = 1  -- Loop back to start
				end
			else
				-- Move toward waypoint
				self.x = self.x + (dx / dist) * self.speed * dt
				self.y = self.y + (dy / dist) * self.speed * dt
			end
		end

		return enemy
	end
}
```

Now your level designer can draw patrol routes visually in Tiled, and enemies will follow them automatically. No coordinate math by hand, no guessing where the waypoints should be. Draw the path, link it to an enemy, done. When you want to change the route, just edit it in Tiled. This is the kind of workflow that makes game development actually fun.

## Adding and Removing Entities at Runtime

Games aren't static. Enemies die, coins get collected, new monsters spawn from portals. We need to handle entities coming and going without breaking our game loops.

Here's a gotcha that trips up a lot of people: if you're iterating through a table with a for loop and you remove something from that table mid-loop, bad things happen. The iteration gets confused, items get skipped, and you end up with bugs that only happen sometimes. Those are the worst bugs.

The solution is to queue up additions and removals, then process them between frames:

```lua
local entities = {}
local toAdd = {}
local toRemove = {}

function addEntity(entity)
	table.insert(toAdd, entity)
end

function removeEntity(entity)
	table.insert(toRemove, entity)
end

function updateEntities(dt)
	-- First, add any pending entities
	for _, entity in ipairs(toAdd) do
		table.insert(entities, entity)
	end
	toAdd = {}

	-- Update everyone who's still alive
	for _, entity in ipairs(entities) do
		if entity.update then
			entity:update(dt)
		end
	end

	-- Finally, remove the dead ones
	for _, entity in ipairs(toRemove) do
		for i, e in ipairs(entities) do
			if e == entity then
				table.remove(entities, i)
				break
			end
		end
	end
	toRemove = {}
end
```

Now when an enemy dies, you call `removeEntity(enemy)`. It won't actually disappear until the update loop finishes, which means your iteration stays safe. Same with spawning - `addEntity(newEnemy)` queues it up for the next frame.

Is this the only way to solve this problem? Nope! You could iterate backwards through the table, or mark entities as "dead" and skip them during iteration. But the queue approach is clean, predictable, and hard to mess up.

## To Remove or Not to Remove?

Once you've extracted all the data from an object layer, you have a choice to make. In Tutorial 01, we just killed the layer outright:

```lua
map:removeLayer("Entities")
```

Gone. No more ugly debug boxes. Nice and clean.

But sometimes you want to keep that layer around. Maybe you want to respawn enemies at their original positions when the player dies. Maybe you're debugging and want to see where things were supposed to spawn. In that case, just hide the layer:

```lua
map.layers["Entities"].visible = false
```

The layer still exists in memory, you can still access its objects, but it doesn't draw. Best of both worlds during development. You can always switch to `removeLayer` once you're confident everything's working.

## Putting It All Together

Here's a complete example with multiple entity types - a player, enemies, and coins:

```lua
local sti = require "sti"

local entities = {}
local player = nil

function love.load()
	map = sti("map.lua")

	local layer = map:addCustomLayer("Sprites", 8)

	for id, object in pairs(map.objects) do
		if object.name == "Player" then
			player = {
				x      = object.x,
				y      = object.y,
				speed  = 96,
				sprite = love.graphics.newImage("player.png")
			}
		elseif object.type == "Enemy" then
			table.insert(entities, {
				type   = "enemy",
				x      = object.x,
				y      = object.y,
				health = object.properties.health or 50,
				sprite = love.graphics.newImage("enemy.png")
			})
		elseif object.type == "Coin" then
			table.insert(entities, {
				type   = "coin",
				x      = object.x,
				y      = object.y,
				value  = object.properties.value or 1,
				sprite = love.graphics.newImage("coin.png")
			})
		end
	end

	layer.update = function(self, dt)
		if love.keyboard.isDown("w", "up") then player.y = player.y - player.speed * dt end
		if love.keyboard.isDown("s", "down") then player.y = player.y + player.speed * dt end
		if love.keyboard.isDown("a", "left") then player.x = player.x - player.speed * dt end
		if love.keyboard.isDown("d", "right") then player.x = player.x + player.speed * dt end
	end

	layer.draw = function(self)
		for _, entity in ipairs(entities) do
			love.graphics.draw(entity.sprite, entity.x, entity.y)
		end
		love.graphics.draw(player.sprite, player.x, player.y)
	end

	-- Hide object layers so we don't see the debug boxes
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
	local tx = player.x - love.graphics.getWidth() / 2
	local ty = player.y - love.graphics.getHeight() / 2
	map:draw(-tx, -ty)
end
```

We've got a player, we've got enemies with configurable health, we've got coins with configurable values, and it all comes from Tiled. No hard-coded positions, no magic numbers scattered through your code. Need to move an enemy? Open Tiled, drag it, save. Want more coins? Copy, paste, done.

That's the power of treating Tiled as your game's data source. Your level designer (even if that's just future-you) will thank you.

In the next tutorial, we'll tackle Y-sorting - making sure all these entities draw in the right order so your player can walk behind trees and buildings like they're actually part of a real world.
