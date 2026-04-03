# Collision Detection with Bump.lua

This tutorial assumes you have completed the [Introduction to STI](01-introduction-to-sti.md) and have a working player moving around a map. If you haven't done that yet, go back and complete it first!

In the previous tutorial, our player could move freely around the map. That's great for ghosts, but most games need walls, obstacles, and solid objects. In this tutorial, we'll add collision detection using [bump.lua](https://github.com/kikito/bump.lua), a simple and fast AABB (Axis-Aligned Bounding Box) collision library.

## What is Bump.lua?

Bump.lua is not a physics engine. It doesn't simulate gravity, momentum, or bouncing. Instead, it answers a simple question: "If I try to move from here to there, what's in my way?" This makes it perfect for top-down games, platformers with simple physics, or any game where you just need solid obstacles.

## Setting Up Tiled

Before we write any code, we need to tell STI which parts of our map should be solid. Open your map in Tiled and find the layer (or tiles) that should block the player. This might be a "Walls" tile layer or an object layer with collision shapes.

Select your collision layer and open the Properties panel. Add a new custom property called `collidable` and set it to `true`. That's it! STI will automatically pick up any layer, tile, or object marked as collidable.

You can also mark individual tiles as collidable in your tileset. This is useful when only some tiles in a layer should block movement (like trees on a grass layer).

## Loading the Bump Plugin

Let's modify our code to load STI with the bump plugin. We also need to require bump.lua itself, so make sure you have it in your project.

```lua
-- Include Simple Tiled Implementation into project
local sti = require "sti"
local bump = require "bump"

function love.load()
	-- Load map file with bump plugin
	map = sti("map.lua", { "bump" })

	-- Create bump world with 32 pixel cell size
	world = bump.newWorld(32)

	-- Initialize bump world with map's collidable objects
	map:bump_init(world)
end
```

The second argument to `sti()` is a table of plugins to load. By passing `{ "bump" }`, STI will load the bump plugin and add new methods to our map object.

The `bump.newWorld(32)` creates a new bump world. The `32` is the cell size used for spatial hashing. A good rule of thumb is to use your tile size.

The `map:bump_init(world)` scans all layers, tiles, and objects for the `collidable` property and adds them to the bump world automatically.

## Adding the Player to the World

Our player needs to exist in the bump world too. Let's modify our player creation code:

```lua
function love.load()
	-- Load map file with bump plugin
	map = sti("map.lua", { "bump" })

	-- Create bump world
	world = bump.newWorld(32)

	-- Initialize bump world with map's collidable objects
	map:bump_init(world)

	-- Create new dynamic data layer called "Sprites" as the 8th layer
	local layer = map:addCustomLayer("Sprites", 8)

	-- Get player spawn object
	local player
	for k, object in pairs(map.objects) do
		if object.name == "Player" then
			player = object
			break
		end
	end

	-- Create player object
	local sprite = love.graphics.newImage("sprite.png")
	layer.player = {
		sprite = sprite,
		x      = player.x,
		y      = player.y,
		w      = 16,  -- Collision box width
		h      = 16,  -- Collision box height
		ox     = sprite:getWidth() / 2,
		oy     = sprite:getHeight() / 1.35
	}

	-- Add player to bump world
	-- The collision box is centered at the player's feet
	world:add(
		layer.player,
		layer.player.x - layer.player.w / 2,
		layer.player.y - layer.player.h / 2,
		layer.player.w,
		layer.player.h
	)

	-- ... rest of layer setup
end
```

Notice we added `w` and `h` properties to define our collision box. The collision box doesn't have to match the sprite size. In fact, it's usually smaller. A 16x16 box at the player's feet feels much better than a huge box covering the whole sprite.

When we call `world:add()`, we pass the player object as an identifier, then the x, y, width, and height of the collision box. We offset the x position by half the width so the box is centered on the player's position.

## Collision-Aware Movement

Here's where the magic happens. Instead of directly changing the player's position, we ask bump to move us and handle any collisions:

```lua
	-- Add controls to player
	layer.update = function(self, dt)
		-- 96 pixels per second
		local speed = 96 * dt

		-- Calculate desired movement
		local dx, dy = 0, 0

		if love.keyboard.isDown("w", "up") then
			dy = dy - speed
		end

		if love.keyboard.isDown("s", "down") then
			dy = dy + speed
		end

		if love.keyboard.isDown("a", "left") then
			dx = dx - speed
		end

		if love.keyboard.isDown("d", "right") then
			dx = dx + speed
		end

		-- Get current position in bump world
		local px, py = world:getRect(self.player)

		-- Move with collision detection
		local actualX, actualY, cols, len = world:move(
			self.player,
			px + dx,
			py + dy
		)

		-- Update player position based on collision result
		self.player.x = actualX + self.player.w / 2
		self.player.y = actualY + self.player.h / 2
	end
```

Instead of `self.player.x = self.player.x + dx`, we call `world:move()`. This function takes our player, the desired new position, and returns the actual position after resolving collisions. If there's a wall in the way, the player stops at the wall instead of passing through it.

The `cols` table contains information about each collision that occurred, and `len` is the number of collisions. We'll use these later for more advanced collision handling.

## Debug Drawing

Bump doesn't draw anything by itself, but STI provides a handy debug function:

```lua
function love.draw()
	-- Scale world
	local scale = 2
	local screen_width  = love.graphics.getWidth()  / scale
	local screen_height = love.graphics.getHeight() / scale

	-- Translate world so that player is always centred
	local player = map.layers["Sprites"].player
	local tx = math.floor(player.x - screen_width  / 2)
	local ty = math.floor(player.y - screen_height / 2)

	-- Draw world
	map:draw(-tx, -ty, scale)

	-- Draw collision boxes (for debugging)
	love.graphics.push()
	love.graphics.scale(scale)
	love.graphics.translate(-tx, -ty)
	map:bump_draw(world)
	love.graphics.pop()
end
```

The `map:bump_draw(world)` function draws red rectangles around all collidable objects. This is incredibly useful for debugging. You can see exactly what the player is colliding with and adjust your collision shapes as needed.

Note that we need to apply the same scale and translation manually for the debug drawing since `bump_draw` doesn't take those as arguments.

## Handling Different Collision Types

Sometimes you want different responses to different collisions. Maybe walls should stop you, but coins should be collected. Bump supports this with collision filters:

```lua
-- Move with collision filtering
local filter = function(item, other)
	if other.properties and other.properties.collectible then
		return "cross"  -- Pass through but detect collision
	end
	return "slide"  -- Default: slide along walls
end

local actualX, actualY, cols, len = world:move(
	self.player,
	px + dx,
	py + dy,
	filter
)

-- Check for collectible collisions
for i = 1, len do
	local col = cols[i]
	if col.other.properties and col.other.properties.collectible then
		-- Collect the item
		world:remove(col.other)
	end
end
```

The filter function returns a collision type:
- `"slide"` - Stop at the obstacle and slide along it (default)
- `"touch"` - Stop at the obstacle, no sliding
- `"cross"` - Pass through but still detect the collision
- `"bounce"` - Bounce off the obstacle

## Putting It All Together

Here's the complete code with bump collision:

```lua
-- Include Simple Tiled Implementation into project
local sti = require "sti"
local bump = require "bump"

function love.load()
	-- Load map file with bump plugin
	map = sti("map.lua", { "bump" })

	-- Create bump world
	world = bump.newWorld(32)

	-- Initialize bump world with map's collidable objects
	map:bump_init(world)

	-- Create new dynamic data layer called "Sprites" as the 8th layer
	local layer = map:addCustomLayer("Sprites", 8)

	-- Get player spawn object
	local player
	for k, object in pairs(map.objects) do
		if object.name == "Player" then
			player = object
			break
		end
	end

	-- Create player object
	local sprite = love.graphics.newImage("sprite.png")
	layer.player = {
		sprite = sprite,
		x      = player.x,
		y      = player.y,
		w      = 16,
		h      = 16,
		ox     = sprite:getWidth() / 2,
		oy     = sprite:getHeight() / 1.35
	}

	-- Add player to bump world
	world:add(
		layer.player,
		layer.player.x - layer.player.w / 2,
		layer.player.y - layer.player.h / 2,
		layer.player.w,
		layer.player.h
	)

	-- Add controls to player
	layer.update = function(self, dt)
		local speed = 96 * dt
		local dx, dy = 0, 0

		if love.keyboard.isDown("w", "up") then
			dy = dy - speed
		end

		if love.keyboard.isDown("s", "down") then
			dy = dy + speed
		end

		if love.keyboard.isDown("a", "left") then
			dx = dx - speed
		end

		if love.keyboard.isDown("d", "right") then
			dx = dx + speed
		end

		local px, py = world:getRect(self.player)
		local actualX, actualY = world:move(self.player, px + dx, py + dy)

		self.player.x = actualX + self.player.w / 2
		self.player.y = actualY + self.player.h / 2
	end

	-- Draw player
	layer.draw = function(self)
		love.graphics.draw(
			self.player.sprite,
			math.floor(self.player.x),
			math.floor(self.player.y),
			0,
			1,
			1,
			self.player.ox,
			self.player.oy
		)
	end

	-- Remove unneeded object layer
	map:removeLayer("Spawn Point")
end

function love.update(dt)
	map:update(dt)
end

function love.draw()
	local scale = 2
	local screen_width  = love.graphics.getWidth()  / scale
	local screen_height = love.graphics.getHeight() / scale

	local player = map.layers["Sprites"].player
	local tx = math.floor(player.x - screen_width  / 2)
	local ty = math.floor(player.y - screen_height / 2)

	map:draw(-tx, -ty, scale)

	-- Debug: draw collision boxes
	love.graphics.push()
	love.graphics.scale(scale)
	love.graphics.translate(-tx, -ty)
	map:bump_draw(world)
	love.graphics.pop()
end
```

With about 100 lines of code, we've added full collision detection to our game. The player now respects walls and obstacles, and we have debug visualization to help us tune our collision shapes.

## Next Steps

Bump.lua is great for games that don't need realistic physics. In the next tutorial, we'll look at the Box2D plugin for when you need gravity, momentum, and complex physics interactions.
