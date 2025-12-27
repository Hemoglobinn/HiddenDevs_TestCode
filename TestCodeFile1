--[[
Written by Dot_Cor/Dudeguy90539
This is a script inside my sample game which handles the behavior of a weapon from the player's weapon. 
It handles things like third person camera toggles and manipulation, firing signals to the local projectile script, interpreting player input for the weapon, running checks for ammo, etc. 
]]

-- Global References
local UIS = game:GetService("UserInputService")
local plr = game:GetService("Players").LocalPlayer
local InputModule = require(plr.PlayerScripts.Inputs)
local CommonFunctions = require(game:GetService("ReplicatedStorage"):WaitForChild("CommonFunctions"))
local FirstPersonModule = require(plr.PlayerScripts:WaitForChild("FirstPersonModelHandler"))

local clientCasterEvent = game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("ReplicateProjectileClient")
local ServerCasterEvent = nil

-- Single Fire Projectile
local WeaponModule = {}
WeaponModule.__index = WeaponModule

-- Instantiator
function WeaponModule.new(Tool)
	--Object Definition
	local selfData = {
		ConnectedEvents = {}
	}
	
	--Tool object setup
	selfData.Tool = Tool
	Tool.Enabled = true
	local muzzle = Tool.Muzzle
	local animations = Tool.Animations
	local Crosshair = Tool:FindFirstChild("Crosshair") or script:WaitForChild("Crosshair")
	ServerCasterEvent = Tool:FindFirstChild("Fire")

	--Character References
	local hum = nil
	local char = nil
	local Cam = workspace.CurrentCamera

	--Runtime Values
	local Equipped = false
	local lastFire = tick()
	local update = nil
	local m1Down = false
	local activeCrosshair = nil
	local ammo = Tool:GetAttribute("Ammo") or 0
	local stepClick = false
	local CurrentRecoil = 0
	local SpreadFactor = 0
	local ActiveCrosshairModule = nil
	
	--tool stats
	local toolStats = require(game:GetService("ReplicatedStorage"):WaitForChild("ToolStats"))[Tool.Name] or {}

	--animation tracks
	local firetrack = nil
	local fireAnim = Tool.Animations:FindFirstChild("Fire")
	
	--camera data
	local PlayerModule = game.Players.LocalPlayer.PlayerScripts:WaitForChild("PlayerModule")
	local cameras = require(PlayerModule):GetCameras()
	local BoundKeys = PlayerModule.CameraModule.MouseLockController:WaitForChild("BoundKeys")
	local OldBoundKeys = BoundKeys.Value
	
	--aiming tween references
	local aimTweenOffset = nil
	local aimTweenFOV = nil
	local plrCamTween = nil
	
	--temporary offset values
	local SL_default = Vector3.new(2,.5,0)
	local SL_offset = Vector3.new(-.7,-.5,0)
	local defaultZmin = plr.CameraMinZoomDistance 
	local defaultZmax = plr.CameraMaxZoomDistance
	local defaultFOV = 70
	
	
	--============================mouse lock for PC/Console===============================
	local function PcConsoleLock(lock)
		local CameraController = cameras.activeCameraController
		local MouseLockController = cameras.activeMouseLockController

		--perform lock
		if lock then
			if MouseLockController:GetIsMouseLocked() then -- toggle shift lock off
				MouseLockController:OnMouseLockToggled()
			end
			CameraController:SetMouseLockOffset(Vector3.new())
			CameraController:SetIsMouseLocked(true)
			BoundKeys.Value = "" -- disables shift lock toggle
		else
			CameraController:SetIsMouseLocked(false)
			BoundKeys.Value = OldBoundKeys -- restores shift lock toggle
		end
	end

	--locks mouse to center in third person
	local function ToggleLock(lock)
		if UIS.TouchEnabled then
			game.ReplicatedStorage.Remotes.MobileLock:Fire(lock)
		else
			PcConsoleLock(lock)
		end
	end
	
	
	--===============================Aim Function=============================
	local aimed = false
	local function ads(aim)
		if not hum then return end
		if not Equipped then return end
		if Tool:GetAttribute("ReloadState") and aim then return end
		aimed = aim

		if aim then
			--aim in
			aimTweenOffset = game.TweenService:Create(hum, TweenInfo.new(.5), {CameraOffset = SL_offset + SL_default})
			aimTweenFOV = game.TweenService:Create(workspace.Camera, TweenInfo.new(.5), {FieldOfView = toolStats.AimFOV})
			plrCamTween = game.TweenService:Create(plr, TweenInfo.new(.5), {CameraMinZoomDistance = 2, CameraMaxZoomDistance = 2})

			aimTweenOffset:Play()  aimTweenFOV:Play() plrCamTween:Play()

		else
			--aim end
			aimTweenOffset = game.TweenService:Create(hum, TweenInfo.new(.5), {CameraOffset = Vector3.new(.4,-.2,0) + SL_default})
			aimTweenFOV = game.TweenService:Create(workspace.Camera, TweenInfo.new(.5), {FieldOfView = defaultFOV})
			plrCamTween = game.TweenService:Create(plr, TweenInfo.new(.5), {CameraMinZoomDistance = 7.5, CameraMaxZoomDistance = 7.5})

			aimTweenOffset:Play()  aimTweenFOV:Play() plrCamTween:Play()
		end
	end
	
	
	--===================First-Person Toggle Function=======================
	
	
	
	--==========================Fire function===============================
	local function fire()
		--equip check
		if not Equipped then return end
		
		--fire rate check
		if lastFire + toolStats.FireRate > tick() then 
			--allows gun to fire immediately when ready if player clicks beforehand
			if lastFire + toolStats.FireRate * .75 > tick() then
				stepClick = true
			end
			return 
		end
		
		--reload check
		if Tool:GetAttribute("ReloadState") then return end
		
		--ammo check
		if ammo <= 0 then Tool.PrimaryPart.Click:Play() m1Down = false return end
		
		--increment values
		lastFire = tick()
		ammo = ammo - 1
		stepClick = false

		--cast projectile
		local rayLast = tick()
		local mPosResult = workspace:Raycast(workspace.Camera.CFrame.Position, workspace.Camera.CFrame.LookVector*2000) 
		local mPos = mPosResult and mPosResult.Position
		
		local calcSpreadFactor = toolStats.MinSpread + (toolStats.MaxSpread-toolStats.MinSpread) * SpreadFactor
		local StartCF = CFrame.new(muzzle.Position, mPos or (muzzle.Position + workspace.CurrentCamera.CFrame.LookVector) )
		local spreadCF = CFrame.fromOrientation(math.random(-calcSpreadFactor*100, calcSpreadFactor*100)/10000, math.random(-calcSpreadFactor*100, calcSpreadFactor*100)/10000, 0 )
		StartCF = StartCF * spreadCF

		local projectileParams = CommonFunctions.deepCopy(toolStats.ProjectileParams)
		
		projectileParams.StartTime = tick()
		projectileParams.StartCF = StartCF
		projectileParams.Caster = muzzle
		projectileParams.ModelsBlacklist = {Tool.Parent, Tool}
		projectileParams.ToolTip = Tool.Name
		
		--Client projectile
		clientCasterEvent:Fire(plr, projectileParams)
		
		--Server projectile request
		if ServerCasterEvent then ServerCasterEvent:FireServer(projectileParams) end
		
		--Recoil
		CurrentRecoil = toolStats.CamRecoil
		SpreadFactor = math.min(1, SpreadFactor + toolStats.SpreadStep)

		--animation
		if firetrack and (not firetrack.IsPlaying) then firetrack:Play() end
		if toolStats.IKRecoil and char.PrimaryPart and char.PrimaryPart:FindFirstChild("IKPointer") then 
			char.PrimaryPart.IKPointer.CFrame = char.PrimaryPart.IKPointer.CFrame * CFrame.Angles(math.rad(toolStats.IKRecoil),0,0) 
		end
	end
	
	
	--=============================update loop=============================
	local ammoSyncInterval = 1
	local lastAmmoSync = tick()
	local function updateFunc(dt)
		--sync ammo data
		if lastAmmoSync + ammoSyncInterval < tick() then
			lastAmmoSync = tick()
			ammo = Tool:GetAttribute("Ammo") or 0
		end
		
		if stepClick then fire() return end
		--if inputs.M1 then fire() end
		
		--Recoil
		if CurrentRecoil > 0 then
			Cam.CFrame = Cam.CFrame * CFrame.Angles(math.rad(CurrentRecoil * dt),0,0)
			CurrentRecoil = math.max(0, CurrentRecoil - (toolStats.CamRecoilRecovery or 0.1) * dt)
		end
		
		--Recovery
		SpreadFactor = math.max(0, SpreadFactor - (toolStats.SpreadRecovery or .1) * dt)
		
		--Crosshair
		if ActiveCrosshairModule then
			ActiveCrosshairModule.Factor = SpreadFactor
			ActiveCrosshairModule.Update()
		end
		
	end
	
	
	--==============================Equip====================
	local EquipFunc = function()
		--Equip
		if Equipped then return end
		Equipped = true
		
		--Establish References
		char = plr.Character
		hum = char:WaitForChild("Humanoid")
		
		--Enable FPS module
		FirstPersonModule.Active = true
		
		--Instantiate Crosshair
		activeCrosshair = Crosshair:Clone()
		activeCrosshair.SourceScript.Value = Tool
		activeCrosshair.Parent = game.Players.LocalPlayer.PlayerGui
		if activeCrosshair and activeCrosshair:FindFirstChild("CrosshairModule") then
			ActiveCrosshairModule = require(activeCrosshair.CrosshairModule)
			ActiveCrosshairModule.MinScale = toolStats.MinSpread
			ActiveCrosshairModule.MaxScale = toolStats.MaxSpread
		end
		
		--Hide Mouse
		UIS.MouseIconEnabled = false
		
		--Break previous lock if one exists
		if cameras.activeMouseLockController and cameras.activeMouseLockController:GetIsMouseLocked() then
			cameras.activeMouseLockController:OnMouseLockToggled()
			cameras.activeCameraController:SetIsMouseLocked(false)
			wait()
		end
		
		--Lock Camera
		ToggleLock(true)
		
		--Offset Camera
		if hum then hum.CameraOffset = Vector3.new(.4,-.2,0) + SL_default end
		
		--Lock Camera Distance
		plr.CameraMaxZoomDistance = 7.5
		plr.CameraMinZoomDistance = 7.5
		
		--animations
		firetrack = hum:LoadAnimation(fireAnim)
		
		--Bind Update Loop
		update = game:GetService("RunService").RenderStepped:Connect(updateFunc)
	end
	
	
	--==================================Unequip=====================
	local UnequipFunc = function()
		--Unequip
		if not Equipped then return end
		Equipped = false
		
		--Disable FPS module
		FirstPersonModule.Active = false
		
		--Unlock Camera
		ToggleLock(false)
		UIS.MouseIconEnabled = true
		
		--Reset Offset
		if hum then hum.CameraOffset = Vector3.new() end
		
		--Unlock Camera Distance
		plr.CameraMaxZoomDistance = defaultZmax
		plr.CameraMinZoomDistance = defaultZmin
		
		--Show Mouse
		UIS.MouseIconEnabled = true
		
		--Cancel Tweens
		if aimTweenFOV then aimTweenFOV:Cancel() end
		if aimTweenOffset then aimTweenOffset:Cancel() end
		if plrCamTween then plrCamTween:Cancel() end
		workspace.Camera.FieldOfView = defaultFOV
		
		--Cancel recoil
		CurrentRecoil = 0
		SpreadFactor = 0
		
		--remove crosshair
		if activeCrosshair then activeCrosshair:Destroy() end
		
		--Break Update Loop
		if update then update:Disconnect() update = nil end
	end
	
	
	--======================================Connections==========================
	selfData.ConnectedEvents[#selfData.ConnectedEvents + 1] = Tool.Equipped:Connect(EquipFunc)
	selfData.ConnectedEvents[#selfData.ConnectedEvents + 1] = Tool.Unequipped:Connect(UnequipFunc)
	
	--fire equip if placed directly into character
	if game.Players:GetPlayerFromCharacter(Tool.Parent) then
		EquipFunc()
	end
	
	--connect input module
	selfData.ConnectedEvents[#selfData.ConnectedEvents + 1] = InputModule.InputBegan:Connect(function(action)
		if not Equipped then return end
		if action == "Reload" then
			--reload
			Tool.Action:FireServer("R")
		end
		if action == "Fire" then
			--fire
			m1Down = true
		end
		if action == "Aim" then
			ads(true)
		end
	end)
	
	selfData.ConnectedEvents[#selfData.ConnectedEvents + 1] = InputModule.InputEnded:Connect(function(action)
		if not Equipped then return end
		if action == "Aim" then
			ads(false)
		end
		if action == "Fire" then
			m1Down = false
		end
	end)
	
	UIS.InputBegan:Connect(function(input, processed)
		if not Equipped then return end
		if processed then return end
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			m1Down = true
			fire()
		end
		if input.UserInputType == Enum.UserInputType.MouseButton2 then
			ads(true)
		end
	end)
	
	UIS.InputEnded:Connect(function(input, processed)
		if not Equipped then return end
		if processed then return end
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			m1Down = false
		end
		if input.UserInputType == Enum.UserInputType.MouseButton2 then
			ads(false)
		end
	end)
	
	--temporary bindings
	InputModule.BindToInput(Enum.KeyCode.R, "Reload")
	
	--=========================Object Insertion============================
	return setmetatable(selfData, WeaponModule)
end

-- Removal
function WeaponModule:Destroy()
	if getmetatable(self) == nil then
		return
	end
	
	--disconnect events
	for _, i in pairs(self.ConnectedEvents) do
		i:Disconnect()
	end
	
	--delete object
	setmetatable(self, nil)
end

--close module
return WeaponModule
