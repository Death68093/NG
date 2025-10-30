# NG


<details>
  <summary>ServerSide Code</summary>
  
  # Put in ServerScriptStorage
  
```
local RepStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local TeleportService = game:GetService("TeleportService")

local config = require(RepStorage:WaitForChild("NG_Config"))
local key = tostring(config.SecretSalt or "")

local NG_Events = RepStorage:FindFirstChild("NG_Events") or Instance.new("Folder")
NG_Events.Name = "NG_Events"
NG_Events.Parent = RepStorage

local NG_GetClient = NG_Events:FindFirstChild("NG_GetClient") or Instance.new("RemoteFunction")
NG_GetClient.Name = "NG_GetClient"
NG_GetClient.Parent = NG_Events

local NG_Fail = NG_Events:FindFirstChild("NG_Fail") or Instance.new("BindableEvent")
NG_Fail.Name = "NG_Fail"
NG_Fail.Parent = NG_Events

local NG_Click = NG_Events:FindFirstChild("NG_Click") or Instance.new("RemoteEvent")
NG_Click.Name = "NG_Click"
NG_Click.Parent = NG_Events


local function punishPlayer(player, reason)
	if not player or not player:IsA("Player") then return end
	if config.Punishment == "kick" then
		player:Kick("Anticheat kicked: "..tostring(reason))
	elseif config.Punishment == "respawn" then
		if player.Character then
			player.Character:BreakJoints()
		end
	elseif config.Punishment == "ban" then
		player:Kick("Anticheat banned: "..tostring(reason))

	elseif config.Punishment == "banWorld" and config.BanWorldId then
        TeleportService:TeleportAsync(config.BanWorldId, player)
	end
end

local function createHitbox(player, char)
    if not char then return end
    local hum = char:FindFirstChild("Humanoid")
    if not hum then return end
    if char:FindFirstChild("NG_Hitbox") then return end

    local basePart
    if hum.RigType == Enum.HumanoidRigType.R6 then
        basePart = char:FindFirstChild("Torso")
    else -- R15
        basePart = char:FindFirstChild("HumanoidRootPart")
    end
    if not basePart then return end

    local shrinkFactor = 0.9
    local hitboxSize = basePart.Size * shrinkFactor

    local hitbox = Instance.new("Part")
    hitbox.Name = "NG_Hitbox"
    hitbox.Size = hitboxSize
    hitbox.CFrame = basePart.CFrame
    hitbox.Transparency = 1
    hitbox.Anchored = false
    hitbox.CanCollide = false
    hitbox.CanTouch = true
    hitbox.Massless = true
    hitbox.Parent = char

    local weld = Instance.new("WeldConstraint")
    weld.Part0 = hitbox
    weld.Part1 = basePart
    weld.Parent = hitbox

    local debounce = false
    hitbox.Touched:Connect(function(hit)
        if debounce then return end
        local otherChar = hit and hit.Parent
        local otherPlayer = otherChar and Players:GetPlayerFromCharacter(otherChar)
        if not otherPlayer or otherPlayer == player then return end

        debounce = true
        NG_Fail:Fire(player, "No-Clip")
        task.delay(config.NoClipTimer or 1, function()
            debounce = false
        end)
    end)

    player.CharacterRemoving:Connect(function()
        if hitbox and hitbox.Parent then
            hitbox:Destroy()
        end
    end)
end





local function setupPlayer(player)
	player.CharacterAdded:Connect(function(char)
        char:SetAttribute("NG_Falling", false)
		if config.CheckNoClip then
			createHitbox(player, char)
		end
	end)
	if player.Character then
		if config.CheckNoClip then
			createHitbox(player, player.Character)
		end
	end
end

Players.PlayerAdded:Connect(setupPlayer)
for _, p in ipairs(Players:GetPlayers()) do
	setupPlayer(p)
end


RunService.Heartbeat:Connect(function()
	for _, player in ipairs(Players:GetPlayers()) do
		local success, data = pcall(function()
			return NG_GetClient:InvokeClient(player, key)
		end)
		if not success or not data then
			warn("Client did not respond: "..player.Name)
			continue
		end

		local char = player.Character
		if not char then continue end

		local hum = char:FindFirstChild("Humanoid")
		if not hum then continue end

		
		if config.CheckSpeed then
			if (config.UseServerSpeed or config.UseServer) then
				if data.speed ~= hum.WalkSpeed then
					punishPlayer(player, "Speed")
				end
			else
				if data.speed > config.MaxSpeed then
					punishPlayer(player, "Speed")
				end
			end
		end

		
		if config.CheckJump then
			if (config.UseServerJump or config.UseServer) then
				if data.jumpPower ~= hum.JumpPower or data.jumpHeight ~= (hum.JumpHeight or 0) then
					punishPlayer(player, "Jump")
				end
			else
				if data.jumpPower > config.MaxJumpPower or data.jumpHeight > config.MaxJumpHeight then
					punishPlayer(player, "Jump")
				end
			end
		end

		
		if config.CheckGravity then
			if (config.UseServerGravity or config.UseServer) then
				if data.gravity ~= Workspace.Gravity then
					punishPlayer(player, "Gravity")
				end
			else
				if data.gravity < config.MinGravity or data.gravity > config.MaxGravity then
					punishPlayer(player, "Gravity")
				end
			end
		end

        local state = hum:GetState()
        if state == Enum.HumanoidStateType.Landed then
            player.Character:SetAttribute("NG_Falling", false)
        elseif state == Enum.HumanoidStateType.Freefall then
            player.Character:SetAttribute("NG_Falling", true)
        elseif state == Enum.HumanoidStateType.Jumping then
            if player.Character:GetAttribute("NG_Falling") then
                NG_Fail:Fire(player, "Infinite Jump")
            end
        end


	end
end)


-- ======== EXTEIOR CHECKS ======== --

if config.CheckAutoClick then
    NG_Click.OnServerEvent:Connect(function(plr, clicks)
        if clicks >= config.MaxCps then
            NG_Fail:Fire(plr, "AutoClicking")
        end
    end)
end


NG_Fail.Event:Connect(function(plr, reason)
	punishPlayer(plr, reason)
end)

print("[NG] Loaded")
```
</details>



<details>
  <summary>ClientSide Code</summary>

 # Put in StarterPlayer > StarterPlayerScripts

 ```
 local Players = game:GetService("Players")
local RepStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local NG_GetClient = RepStorage:WaitForChild("NG_GetClient")
local NG_Click = RepStorage:WaitForChild("NG_Click")

local player = Players.LocalPlayer


local function getHumanoid()
    local char = player.Character
    if char then
        return char:FindFirstChild("Humanoid")
    end
    return nil
end

local clicks = 0

while task.wait(1) do
    NG_Click:FireServer(clicks)
    clicks = 0
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        clicks = clicks + 1
    end
end)

NG_GetClient.OnClientInvoke = function(key)
    local hum = getHumanoid()
    local walkSpeed, jumpPower, jumpHeight = 0, 0, 0

    if hum then
        walkSpeed = hum.WalkSpeed
        jumpPower = hum.JumpPower
        jumpHeight = hum.JumpHeight or 0
    end

    return {
        speed = walkSpeed,
        jumpHeight = jumpHeight,
        jumpPower = jumpPower,
        gravity = Workspace.Gravity,
        key = key
    }
end
```
</details>

<details>
  <summary>Config File</summary>

  # Put this in ReplicatedStorage
  # Name it EXACTLY  NG_Config

  ```
local Config = {}

Config.SecretSalt = math.random(10, 10000)

Config.CheckInt = 5 -- How often to verify files (in seconds), and most checks
Config.FailTime = 3 -- How long until check fails

-- Check Toggles
Config.CheckSpeed = true
Config.CheckJump = true
Config.CheckInfJump = true
Config.CheckWorkspace = true -- Not implemented
Config.CheckPlayerGui = true -- Not Implemented
Config.CheckAutoClick = true
Config.CheckFly = true
Config.CheckNoClip = true
Config.LagSwitch = true -- Not implemented
Config.CheckRootParts = true -- Not Implemented
Config.CheckGravity = true


-- Check Settings
Config.FlyCheckTimer = 1 -- How often to check for flying (in seconds) High values will cause lag
Config.FlyHeight = 20 -- How high to flag flying (studs)

Config.UseServer = true -- If true > Use Server vals for all below true vals.  If false > All UseServerVals will be disregarded (false)
Config.UseServerGravity = true 
Config.UseServerSpeed = true
Config.UseServerJump = true
Config.UseServerJump = true

Config.MaxSpeed = 16
Config.MaxJumpPower = 70
Config.MaxJumpHeight = 7.2
Config.MinGravity = 196.2
Config.MaxGravity = 196.2

Config.MaxCps = 20 -- Max clicks per second to trigger AutoClicker (some legit methods like drag clicking can get 25 cps so don't set too low, default = 20)

Config.UseFlyRaycast = true -- Check Fly using raycast (can be performance intensive)

-- Punishments
Config.Punishment = "respawn" -- (respawn, kick, ban, banWorld) Punishment for failed check..   banWorld will send player's to the placeId set with Config.BanWorldId

Config.BanWorldId = nil

-- Admins  (Use Player ID's)
Config.Admins = {5820167686}


-- ==== END OF SETTINGS ==== --

-- Functions
Config.toggleState = function(check, plr)
    local found = table.find(Config.Admins, plr)
    if found then
        if Config[check] ~= nil and type(Config[check]) == "boolean" then
		Config[check] = not Config[check]
        return Config[check]
	    end
    else
        return "No Permission"
    end
end


return Config
```

</details>
<details>
  <summary>Test File</summary>

  # Place in StarterPlayer > StarterPlayerScripts
  **Use this code to test the anticheat**

```

local RepStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local player = Players.LocalPlayer

local NG_GetClient = RepStorage:WaitForChild("NG_Events"):WaitForChild("NG_GetClient")
local NG_Click = RepStorage:WaitForChild("NG_Events"):WaitForChild("NG_Click")

local char = player.Character or player.CharacterAdded:Wait()
local hum = char:WaitForChild("Humanoid")
local hrp = char:WaitForChild("HumanoidRootPart")


-- TEST FUNCTIONS --

-- 1. Test Speed violation
local function testSpeed()
    hum.WalkSpeed = hum.WalkSpeed + 50
    print("Speed increased for test!")
end

-- 2. Test Jump violation
local function testJump()
    hum.JumpPower = hum.JumpPower + 50
    print("JumpPower increased for test!")
end

-- 3. Test Gravity violation
local function testGravity()
    Workspace.Gravity = Workspace.Gravity + 50
    print("Gravity changed for test!")
end

-- 4. Test Infinite Jump
local function testInfJump()
    hum:ChangeState(Enum.HumanoidStateType.Jumping)
    print("Simulated infinite jump!")
end

-- 5. Test AutoClick
local function testAutoClick()
    NG_Click:FireServer(999)
    print("Sent high click count to server!")
end

-- Keybinds for testing
local UserInputService = game:GetService("UserInputService")

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.One then
        testSpeed()
    elseif input.KeyCode == Enum.KeyCode.Two then
        testJump()
    elseif input.KeyCode == Enum.KeyCode.Three then
        testGravity()
    elseif input.KeyCode == Enum.KeyCode.Four then
        testInfJump()
    elseif input.KeyCode == Enum.KeyCode.Five then
        testAutoClick()
    end
end)

print("Anticheat test script loaded! Press 1-5 to trigger tests:")
print("1 = Speed, 2 = Jump, 3 = Gravity, 4 = InfJump, 5 = AutoClick")
```
