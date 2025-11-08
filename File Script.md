--[[
	CART MANAGEMENT SYSTEM
	This module handles the core cart functionality for a racing/combat game including:
	- Cart spawning and positioning
	- Player-to-cart assignment
	- Cart customization (skins, cosmetics, stickers)
	- Perk attachments (boosters, bumpers/shields)
	- Combat mechanics (bumper collision damage)
	- Cart lifecycle management
--]]

-- Initialize spawn points from workspace and cache their positions
local Spawns = {}
for _, v in workspace.Spawns:GetChildren() do table.insert(Spawns, v.Position) end
workspace.Spawns:Destroy() -- Clean up spawn folder after caching positions

local PlayArea = workspace.PlayArea

return function(RemoteEventFunctions, UnreliableRemoteFunctions, GameplayFunctions, ProcessFunctions, GameState, GameData)
	-- STATE MANAGEMENT
	-- Main cart registry - maps Cart instances to their data (driver, parts, perks, etc.)
	local Carts = {}

	-- Quick access lookups for better performance
	local CartDataCache = {} -- Maps Player -> Cart Data
	local CartCache = {} -- Maps Player -> Cart Instance
	local Counter = 1 -- Tracks which spawn point to use next (resets at end of round)


	--[[
		CART CREATION
		Spawns a new cart for a player at the next available spawn point
		Applies customization and sets up anti-camping detection
	--]]
	ProcessFunctions.CreateCart = function(Player, CartWanted : string, CartData : {cartProperties})
		-- Clone the requested cart type from storage
		local Cart = game.ReplicatedStorage.carts[CartWanted]:Clone()
		Cart.Parent = workspace.Carts
		repeat task.wait() until Cart.PrimaryPart

		-- Position cart at spawn point facing forward (Z-axis)
		Cart:PivotTo(CFrame.lookAt(Spawns[Counter], Spawns[Counter] + Vector3.zAxis))
		task.wait()
		Cart:MoveTo(Spawns[Counter])

		-- Register cart in the main registry
		Carts[Cart] = {
			Driver = Player,
		}

		-- ANTI-CAMPING SYSTEM
		-- After 12 seconds, if player leaves their cart they are defeated
		task.delay(12, function()
			if Cart.Parent and GameState.GameInProgress then
				Cart.Seat:GetPropertyChangedSignal("Occupant"):Once(function()
					task.wait(1.5) -- Grace period for getting back in
					if Carts[Cart] and GameState.GameInProgress then
						ProcessFunctions.HandleDefeat(Player,"is a chicken!")
					end
				end)
				-- Check if player never got in their cart
				if not Cart.Seat.Occupant and GameState.GameInProgress then
					ProcessFunctions.HandleDefeat(Player,"is a chicken!")
				end
			end
		end)

		-- CART CUSTOMIZATION
		local CartParts = {}
		local CartPaint = ProcessFunctions.GetCartSkin(Player) -- Get player's selected skin

		-- Listen for new parts being added to cart and apply skin to them
		local c = Cart.ChildAdded:Connect(function(Cart)
			for each, part in Cart:GetDescendants() do 
				if CartPaint[part.Name] then
					local ColorVals = CartPaint[part.Name][1]
					local MaterialName = CartPaint[part.Name][2]
					if ColorVals and MaterialName then
						part.Color = Color3.new(ColorVals[1], ColorVals[2], ColorVals[3])
						part.Material = Enum.Material[MaterialName]
					end
				end
				if part:IsA("BasePart") then
					table.insert(CartParts, part)
				end
			end
		end)

		Carts[Cart].CartParts = CartParts
		Cart.Seat:SetNetworkOwner(Player) -- Give player control of cart physics

		-- Clean up the ChildAdded listener after 20 seconds
		task.delay(20, function()
			c:Disconnect() c = nil
		end)

		-- Add cosmetic items and stickers to the cart
		GameplayFunctions.AddCosmeticsAndStickersToCart(Player, Cart)

		-- Apply skin to all existing cart parts
		for each, part in Cart:GetDescendants() do 
			if CartPaint[part.Name] then
				local ColorVals = CartPaint[part.Name][1]
				local MaterialName = CartPaint[part.Name][2]
				if ColorVals and MaterialName then
					part.Color = Color3.new(ColorVals[1], ColorVals[2], ColorVals[3])
					part.Material = Enum.Material[MaterialName]
				end
				if part ~= Cart.PrimaryPart then
					table.insert(CartParts, part)
				end
			end
		end

		-- Update quick-access caches
		CartDataCache[Player] = Carts[Cart]
		CartCache[Player] = Cart
		Counter += 1 -- Move to next spawn point
	end

	-- Returns the full cart registry
	ProcessFunctions.GetCarts = function()
		return Carts
	end

	--[[
		CART DESTRUCTION
		Called when a player is defeated - removes their cart from the game
	--]]
	ProcessFunctions.DestroyCart = function(Player)
		local cart = CartCache[Player]

		Carts[cart] = nil
		cart:BreakJoints() -- Destroy all welds/joints
		task.wait(3) -- Wait before full cleanup

		cart:Destroy()
	end

	--[[
		RACE START SEQUENCE
		Teleports all players into their carts and initializes physics
	--]]
	GameplayFunctions.PutPlayersInCarts = function()
		PlayArea.Parent = nil -- Remove play area barrier
		game.ReplicatedStorage.RemoteEvent:FireAllClients("NewRacePlayerList")

		for Cart, Data in Carts do
			task.spawn(function()
				-- Show visual effects to driver
				game.ReplicatedStorage.UnreliableRemoteEvent:FireClient(Data.Driver, "Visual")
				task.wait(1)

				local Seat = Cart:WaitForChild("Seat", 5)
				if not Seat then return end	

				-- Unanchor all cart parts to enable physics
				for _, v in Cart:GetDescendants() do if v:IsA"BasePart" then v.Anchored = false end end
				task.wait(.5)

				if Data.Driver and Data.Driver.Character then
					-- Handle players sitting on swings - eject them first
					if Data.Driver.Character.Humanoid.SeatPart and Data.Driver.Character.Humanoid.SeatPart.Name == "Swing" then
						local Swing = Data.Driver.Character.Humanoid.SeatPart
						Swing.Disabled = true
						Data.Driver.Character.Humanoid.JumpHeight = 8
						Data.Driver.Character.Humanoid.Jump = true
						task.wait(.25)
						task.delay(.5, function()
							Swing.Disabled = false
						end)
					end

					-- Teleport player to cart and sit them down
					Data.Driver.Character:PivotTo(Cart.Seat.CFrame) task.wait()
					Seat.Disabled = false
					Cart.Seat:Sit(Data.Driver.Character.Humanoid)

					-- Disable jumping while in cart
					Data.Driver.Character.Humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, false)
					Data.Driver.Character.Humanoid.JumpHeight = 0

					-- Set collision group for player head (prevents pushing)
					Data.Driver.Character.Head.CollisionGroup = "Carts"
				end
			end)
		end
		task.wait(3)
	end

	-- Expose cart cache to game data
	GameData.CartDataCache = CartDataCache

	--[[
		PERK ATTACHMENT SYSTEM
		Attaches powerup items to carts (boosters for speed, bumpers for combat)
	--]]
	GameplayFunctions.AttachPerksToCart = function(Player, PerkData)
		local CartData = CartDataCache[Player]
		local Cart = CartCache[Player]

		-- BOOSTER ATTACHMENT (speed boost powerup)
		if PerkData.BoosterFuel and PerkData.BoosterFuel > 0 then
			local modelOrig = game.ReplicatedStorage.cartmods[Cart.Name].Booster
			-- Calculate relative position from template cart
			local relCF = game.ReplicatedStorage.carts[Cart.Name]:GetPivot():ToObjectSpace(modelOrig:GetPivot())
			local booster = modelOrig:Clone()
			booster:PivotTo(Cart:GetPivot() * relCF)
			booster.Parent = Cart

			-- Weld booster to cart
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = booster.PrimaryPart
			weld.Part1 = Cart.PrimaryPart
			weld.Parent = booster.PrimaryPart

			-- Store booster data
			CartData.Booster = booster
			CartData.BoosterFuel = PerkData.BoosterFuel
			CartData.BoosterStrength = PerkData.BoosterStrength
		end

		-- BUMPER ATTACHMENT (combat shield that damages others)
		if PerkData.ShieldAmount and PerkData.ShieldAmount > 0 then
			local modelOrig = game.ReplicatedStorage.cartmods[Cart.Name].Bumper
			local relCF = game.ReplicatedStorage.carts[Cart.Name]:GetPivot():ToObjectSpace(modelOrig:GetPivot())
			local bumper = modelOrig:Clone()
			bumper:PivotTo(Cart:GetPivot() * relCF)
			bumper.Parent = Cart

			-- Weld bumper to cart
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = bumper.PrimaryPart
			weld.Part1 = Cart.PrimaryPart
			weld.Parent = bumper.PrimaryPart

			-- Store bumper data
			CartData.Bumper = bumper
			CartData.ShieldUses = PerkData.ShieldAmount
			CartData.ShieldDamage = PerkData.ShieldHarm

			-- BUMPER COMBAT SYSTEM
			-- Uses raycasting to detect collisions in front of cart
			local RaycastParams = RaycastParams.new() 
			RaycastParams.FilterType = Enum.RaycastFilterType.Include

			-- Wait 5 seconds before enabling combat (gives players time to spread out)
			task.delay(5, function()
				RaycastParams.FilterDescendantsInstances = game:GetService("CollectionService"):GetTagged("CollisionBox")

				-- Function runs every frame to check for bumper hits
				CartData.BumperFunc = function()
					-- Cast 5 rays in a cone shape to detect carts in front
					for i = -2, 2 do
						-- Calculate angle for this ray (-17.5  to +17.5 )
						local angle = i * -.0875

						-- Cast ray 20 studs forward
						local rayOrigin = Cart:GetPivot().Position
						local raycastResult = workspace:Raycast(rayOrigin, (Cart:GetPivot() * CFrame.Angles(0, angle, 0)).LookVector * 20, RaycastParams)

						if raycastResult then
							local hitCart = raycastResult.Instance.Parent
							local LocalCartDict = Carts[hitCart]

							-- Check if hit a valid opponent cart (not already hit by this player)
							if LocalCartDict and not LocalCartDict.LostControl and LocalCartDict.Tagged ~= Player then 
								LocalCartDict.LostControl = true -- Prevent control during hit
								local Driver = LocalCartDict.Driver

								-- Tag this cart so same player can't hit it again immediately
								LocalCartDict.Tagged = Player
								task.delay(3, function()
									if LocalCartDict and LocalCartDict.Tagged == Player then
										LocalCartDict.Tagged = nil
									end
								end)

								if not Cart.PrimaryPart then return end

								-- Send hit effect to victim (knockback + damage)
								game.ReplicatedStorage.UnreliableRemoteEvent:FireClient(Driver, "BumperHit", 
									Cart.PrimaryPart.AssemblyLinearVelocity, -- Attacker's velocity
									hitCart:GetPivot():PointToObjectSpace(raycastResult.Position) * Vector3.new(1.2, .5, .3), -- Hit direction
									CartData.ShieldDamage or 0  -- Bonus damage from bumper
								)

								-- Apply damage to victim
								if not Driver.Character then return end
								local Humanoid = Driver.Character.Humanoid
								Humanoid:TakeDamage(4 + (CartData.ShieldDamage or 0) * 3)

								-- Check if victim was defeated
								if Humanoid.Health <= 0 then
									ProcessFunctions.HandleDefeat(Driver, "was rammed by " .. Player.DisplayName)
								end

								-- Restore control after 2 seconds
								task.delay(2, function()
									if LocalCartDict then
										LocalCartDict.LostControl = nil
									end
								end)
							end
						end
					end
				end
				-- Register function to run every frame
				ProcessFunctions.ToHeartbeat(CartData.BumperFunc)
			end)
		end
	end

	-- Called by client when booster runs out of fuel
	RemoteEventFunctions.OutOfFuel = function(Player)
		local CartData = CartDataCache[Player]
		local Cart = CartCache[Player]

		local Booster = Cart:FindFirstChild("Booster")
		if Booster then
			Booster:BreakJoints() -- Detach from cart
			for _, v in Booster:GetChildren() do 
				v.CanTouch = false -- Disable collision
			end
			game.Debris:AddItem(Booster, 3) -- Delete after 3 seconds
		end
	end

	-- Quick lookup for other systems to get cart data
	ProcessFunctions.GetCartData = function(Player)
		return CartCache[Player], CartDataCache[Player]
	end

	--[[
		BUMPER DESTRUCTION
		Removes bumper from cart and stops collision checking
	--]]
	ProcessFunctions.DestroyBumper = function(CartData)
		if CartData.Bumper then
			-- Disable collision on all bumper parts
			for i,v in CartData.Bumper:GetDescendants() do
				if v:IsA"BasePart" then
					v.CanTouch = false
				end
			end
			CartData.Bumper:BreakJoints()
			CartData.Bumper = nil

			-- Stop the frame-by-frame collision checking
			ProcessFunctions.RemoveFromHeartbeat(CartData.BumperFunc)
			CartData.BumperFunc = nil
		end
	end

	-- Reset spawn counter at end of each round
	ProcessFunctions.ResetCartCounter = function()
		Counter = 1
	end

	-- CHARACTER SETUP
	-- When player respawns, set their collision group
	ProcessFunctions.RegisterCharacterAddedFunction(function(Char)
		for i,v in Char:GetDescendants() do 
			if v:IsA("BasePart") then
				v.CollisionGroup = "Carts"
			end
		end
	end)

	-- CLEANUP ON PLAYER LEAVE
	ProcessFunctions.RegisterPlayerRemovingFunction(function(Player)
		task.delay(10, function()
			CartCache[Player] = nil
			CartDataCache[Player] = nil
		end)
	end)

	-- LOAD CART AND COSMETIC DATA
	-- Sets up available carts, skins, and customization options
	local CartData, CosmeticInfo = require(script.Setup)({
		NewDevProductFunction = ProcessFunctions.NewDevProductFunction,
		GetPlayerData = ProcessFunctions.GetPlayerData,
		SetPlayerData = ProcessFunctions.SetPlayerData
	})	
	print(CartData, CosmeticInfo," cart and cosmetic data setup!")
	ProcessFunctions.NewData("CartData", CartData)
	ProcessFunctions.NewData("CosmeticInfo", CosmeticInfo)
	GameData.CosmeticInfo = CosmeticInfo
	GameData.CartData = CartData

	-- Load physical parameters (speed, handling, etc.)
	GameData.CartConfig = require(script.CartConfiguration)
	ProcessFunctions.NewData("PhysicalParameters", GameData.CartConfig)

	-- AUDIOVISUAL REPLICATION
	-- Replicate sounds and particles to other players (not the player who triggered them)
	local UnreliableRemoteEvent = game.ReplicatedStorage.UnreliableRemoteEvent

	UnreliableRemoteFunctions.PlaySound = function(Player, ...)
		for _,a in ProcessFunctions.GetClientsInRound() do 
			if a == Player then continue end
			UnreliableRemoteEvent:FireClient(a, "PlaySound", ...)
		end
	end

	UnreliableRemoteFunctions.Particles = function(Player, ...)
		for _,a in ProcessFunctions.GetClientsInRound() do 
			if a == Player then continue end
			UnreliableRemoteEvent:FireClient(a, "Particles", ...)
		end
	end

	UnreliableRemoteFunctions.GameplayParticles = function(Player, ...)
		for _,a in ProcessFunctions.GetClientsInRound() do 
			if a == Player then continue end
			UnreliableRemoteEvent:FireClient(a, "GameplayParticles", ...)
		end
	end

	-- Load additional cart-related features from separate module
	require(script.Additional)(RemoteEventFunctions, UnreliableRemoteFunctions, GameplayFunctions, ProcessFunctions, GameState, GameData)

	-- Export spawn points to game data
	GameData.Spawns = Spawns
end
