# Physics with Box2D

This tutorial assumes you have completed the [Introduction to STI](01-introduction-to-sti.md). If you haven't, go back and complete it first!

In the previous tutorial on [Collision Detection with Bump](02-collision-with-bump.md), we used bump.lua for simple AABB collision. But what if you want realistic physics? Gravity, momentum, bouncing balls, swinging ropes? That's where Box2D comes in.

## When to Use Box2D vs Bump

Before we dive in, let's clarify when to use each:

**Use Bump.lua when:**
- You need simple "stop at walls" collision
- Top-down games (RPGs, Zelda-likes)
- Simple platformers without complex physics
- You want maximum control over movement

**Use Box2D when:**
- You need gravity and realistic physics
- Objects should bounce, slide, or roll naturally
- You want physics puzzles (Angry Birds-style)
- Platformers with momentum-based movement
- You need sensors, joints, or ragdolls

## Box2D Concepts

Box2D uses a few key concepts:

- **World**: The physics simulation. Has gravity and manages all physics objects.
- **Body**: A physics object (static, dynamic, or kinematic).
- **Shape**: The collision geometry (polygon, circle, edge).
- **Fixture**: Attaches a shape to a body with physical properties (friction, density).
- **Sensor**: A fixture that detects overlap but doesn't cause physical collision.

## Setting Up Tiled

Just like with bump, we mark collision objects in Tiled using custom properties. The Box2D plugin supports several properties:

| Property | Description |
|----------|-------------|
| `collidable` | Set to `true` to include in physics world |
| `dynamic` | Creates a dynamic body (affected by forces) |
| `static` | Creates a static body (immovable, default) |
| `kinematic` | Creates a kinematic body (moves by velocity, not forces) |
| `sensor` | Makes the fixture a sensor (detects but doesn't collide) |
| `friction` | Surface friction (0 to 1, default 0.2) |
| `restitution` | Bounciness (0 to 1, default 0) |
| `categories` | Collision filter category |
| `mask` | Collision filter mask |
| `group` | Collision filter group |

For a basic platformer, mark your ground and wall tiles/objects as `collidable = true`. They'll be static by default, which is what we want for level geometry.

## Loading the Box2D Plugin

Let's set up our physics world:

```lua
-- Include Simple Tiled Implementation into project
local sti = require "sti"

function love.load()
	-- Set up physics
	love.physics.setMeter(32) -- 32 pixels = 1 meter

	-- Load map file with Box2D plugin
	map = sti("map.lua", { "box2d" })

	-- Create physics world with gravity
	-- 0 horizontal gravity, 9.8 * 32 vertical (downward)
	world = love.physics.newWorld(0, 9.8 * 32)

	-- Initialize Box2D bodies from map
	map:box2d_init(world)
end

function love.update(dt)
	-- Update physics world
	world:update(dt)

	-- Update map (animations, etc.)
	map:update(dt)
end

function love.draw()
	map:draw()
end
```

The `love.physics.setMeter(32)` call tells Box2D how many pixels equal one meter. This affects how gravity and forces feel. 32 pixels per meter works well for 32x32 tile games.

The `map:box2d_init(world)` scans all layers for collidable objects and creates Box2D bodies for them automatically.

## Creating a Dynamic Player

Unlike bump where we control position directly, with Box2D we create a physics body and let the simulation handle movement. Here's how to create a dynamic player:

```lua
function love.load()
	love.physics.setMeter(32)

	map = sti("map.lua", { "box2d" })
	world = love.physics.newWorld(0, 9.8 * 32)
	map:box2d_init(world)

	-- Create player physics body
	local layer = map:addCustomLayer("Sprites", 8)

	-- Get spawn position from map
	local spawnX, spawnY = 100, 100
	for k, object in pairs(map.objects) do
		if object.name == "Player" then
			spawnX = object.x
			spawnY = object.y
			break
		end
	end

	-- Create player
	local sprite = love.graphics.newImage("sprite.png")
	layer.player = {
		sprite   = sprite,
		body     = love.physics.newBody(world, spawnX, spawnY, "dynamic"),
		ox       = sprite:getWidth() / 2,
		oy       = sprite:getHeight() / 2,
		onGround = false
	}

	-- Create collision shape (rectangle)
	local shape = love.physics.newRectangleShape(16, 32)
	layer.player.fixture = love.physics.newFixture(layer.player.body, shape, 1)

	-- Prevent rotation (optional, makes platforming easier)
	layer.player.body:setFixedRotation(true)

	-- Set player fixture user data for collision detection
	layer.player.fixture:setUserData({ type = "player" })

	-- Draw player
	layer.draw = function(self)
		local x, y = self.player.body:getPosition()
		love.graphics.draw(
			self.player.sprite,
			math.floor(x),
			math.floor(y),
			self.player.body:getAngle(),
			1,
			1,
			self.player.ox,
			self.player.oy
		)
	end

	map:removeLayer("Spawn Point")
end
```

The player body is created as `"dynamic"`, meaning it's affected by gravity and forces. We also call `setFixedRotation(true)` to prevent the player from spinning when hitting walls, which usually feels better for platformers.

## Platformer Movement

With Box2D, we don't set positions directly. Instead, we apply forces or set velocities:

```lua
	layer.update = function(self, dt)
		local body = self.player.body
		local vx, vy = body:getLinearVelocity()

		-- Horizontal movement
		local moveSpeed = 200

		if love.keyboard.isDown("a", "left") then
			vx = -moveSpeed
		elseif love.keyboard.isDown("d", "right") then
			vx = moveSpeed
		else
			-- Apply friction when not moving
			vx = vx * 0.9
			if math.abs(vx) < 1 then vx = 0 end
		end

		-- Apply horizontal velocity
		body:setLinearVelocity(vx, vy)
	end
```

For jumping, we need to know if the player is on the ground. We'll set up collision callbacks for that:

```lua
function love.load()
	-- ... previous setup code ...

	-- Set up collision callbacks
	world:setCallbacks(beginContact, endContact)
end

function beginContact(a, b, contact)
	local player = map.layers["Sprites"].player
	local aData = a:getUserData()
	local bData = b:getUserData()

	-- Check if player touched ground
	if aData and aData.type == "player" or bData and bData.type == "player" then
		-- Get collision normal
		local nx, ny = contact:getNormal()
		-- If normal points up, we hit ground
		if ny < -0.5 then
			player.onGround = true
		end
	end
end

function endContact(a, b, contact)
	-- We'll use a simple approach: assume we're not grounded
	-- until the next beginContact proves otherwise
	local player = map.layers["Sprites"].player
	local aData = a:getUserData()
	local bData = b:getUserData()

	if aData and aData.type == "player" or bData and bData.type == "player" then
		player.onGround = false
	end
end
```

Now we can add jumping:

```lua
function love.keypressed(key)
	local player = map.layers["Sprites"].player

	if key == "space" or key == "w" or key == "up" then
		if player.onGround then
			local vx, vy = player.body:getLinearVelocity()
			player.body:setLinearVelocity(vx, -300) -- Jump!
			player.onGround = false
		end
	end
end
```

## Debug Drawing

The Box2D plugin provides debug visualization:

```lua
function love.draw()
	local player = map.layers["Sprites"].player
	local px, py = player.body:getPosition()

	local tx = math.floor(px - love.graphics.getWidth() / 2)
	local ty = math.floor(py - love.graphics.getHeight() / 2)

	map:draw(-tx, -ty)

	-- Debug draw physics bodies
	love.graphics.push()
	love.graphics.translate(-tx, -ty)
	map:box2d_draw()
	love.graphics.pop()
end
```

This draws outlines of all physics bodies, making it easy to debug collision issues.

## Using Sensors

Sensors are fixtures that detect overlap without causing physical collision. Perfect for triggers, collectibles, or damage zones. In Tiled, simply add the `sensor` property set to `true` on your collidable object.

You can detect sensor collisions in your callbacks:

```lua
function beginContact(a, b, contact)
	local aData = a:getUserData()
	local bData = b:getUserData()

	-- Check if either fixture is a sensor
	if a:isSensor() or b:isSensor() then
		-- Handle trigger/collectible collision
		if aData and aData.type == "coin" then
			-- Collect the coin
		end
	end
end
```

## Putting It All Together

Here's a complete platformer example:

```lua
local sti = require "sti"

function love.load()
	love.physics.setMeter(32)

	map = sti("map.lua", { "box2d" })
	world = love.physics.newWorld(0, 9.8 * 32)
	map:box2d_init(world)

	world:setCallbacks(beginContact, endContact)

	local layer = map:addCustomLayer("Sprites", 8)

	-- Find spawn point
	local spawnX, spawnY = 100, 100
	for k, object in pairs(map.objects) do
		if object.name == "Player" then
			spawnX, spawnY = object.x, object.y
			break
		end
	end

	-- Create player
	local sprite = love.graphics.newImage("sprite.png")
	layer.player = {
		sprite   = sprite,
		body     = love.physics.newBody(world, spawnX, spawnY, "dynamic"),
		ox       = sprite:getWidth() / 2,
		oy       = sprite:getHeight() / 2,
		onGround = false
	}

	local shape = love.physics.newRectangleShape(16, 32)
	layer.player.fixture = love.physics.newFixture(layer.player.body, shape, 1)
	layer.player.body:setFixedRotation(true)
	layer.player.fixture:setUserData({ type = "player" })

	layer.update = function(self, dt)
		local body = self.player.body
		local vx, vy = body:getLinearVelocity()
		local moveSpeed = 200

		if love.keyboard.isDown("a", "left") then
			vx = -moveSpeed
		elseif love.keyboard.isDown("d", "right") then
			vx = moveSpeed
		else
			vx = vx * 0.9
		end

		body:setLinearVelocity(vx, vy)
	end

	layer.draw = function(self)
		local x, y = self.player.body:getPosition()
		love.graphics.draw(
			self.player.sprite,
			math.floor(x), math.floor(y),
			0, 1, 1,
			self.player.ox, self.player.oy
		)
	end

	map:removeLayer("Spawn Point")
end

function beginContact(a, b, contact)
	local player = map.layers["Sprites"].player
	local nx, ny = contact:getNormal()
	if ny < -0.5 then
		player.onGround = true
	end
end

function endContact(a, b, contact)
	map.layers["Sprites"].player.onGround = false
end

function love.keypressed(key)
	local player = map.layers["Sprites"].player
	if (key == "space" or key == "w" or key == "up") and player.onGround then
		local vx, vy = player.body:getLinearVelocity()
		player.body:setLinearVelocity(vx, -300)
		player.onGround = false
	end
end

function love.update(dt)
	world:update(dt)
	map:update(dt)
end

function love.draw()
	local player = map.layers["Sprites"].player
	local px, py = player.body:getPosition()
	local tx = math.floor(px - love.graphics.getWidth() / 2)
	local ty = math.floor(py - love.graphics.getHeight() / 2)

	map:draw(-tx, -ty)

	-- Debug: uncomment to see physics bodies
	-- love.graphics.push()
	-- love.graphics.translate(-tx, -ty)
	-- map:box2d_draw()
	-- love.graphics.pop()
end
```

## Top-Down Physics (Alternative)

If you're making a top-down game but still want physics (for pushing objects, etc.), just set gravity to zero:

```lua
world = love.physics.newWorld(0, 0) -- No gravity
```

Then use `body:applyForce()` or `body:setLinearVelocity()` for movement. Objects will still collide and push each other around, but won't fall.

## Next Steps

You now know how to use both collision systems in STI! In the next tutorial, we'll dive deeper into working with objects and entities, learning how to spawn different entity types from your Tiled map.
