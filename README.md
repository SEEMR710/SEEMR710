--||Services||--
local RS = game:GetService("ReplicatedStorage")
local SS = game:GetService("ServerStorage")
local SoundService = game:GetService("SoundService")
local Debris = game:GetService("Debris")

--||Folder||--
local remotes = RS:WaitForChild("Events")
local AnimationsFolder = RS:WaitForChild("Animations")
local ServerModules = SS:WaitForChild("Modules")

local WeaponAnimationsFolder = AnimationsFolder:WaitForChild("Weapons")

local SoundFolder = SoundService:WaitForChild("SFX")
local WeaponSoundFolder = SoundFolder:WaitForChild("Weapons")

--||Modules||--
local ServerCombatModule = require(ServerModules.CombatModule)
local HitboxModule = require(ServerModules.MuchachoHitbox)
local HitServiceModule = require(ServerModules.HitService)
local WeaponInfoModule = require(ServerModules.WeaponInfo)

--||Events||--
local swingEvent = remotes.Swing
local VFX_Event = remotes.VFX

--||Settings||--
local Max_Combo = 4

local params = OverlapParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude

------------------------------------------------------------------------------------------------------------------



--||Hitbox Starting||--
local function Hitbox_Start_Stop(char,newHitbox)
	task.spawn(function()
		if not char:GetAttribute("Stunned") then
			newHitbox:Start()
			task.wait(.1)				
			newHitbox:Stop()
		else
			newHitbox:Stop()
		end
	end)
end

--||Feint||--
local function Feinting(char,hum,action,toolName)
	if action == "Feint" then --the feint stuff is really bad lol
		ServerCombatModule.stopAnims(hum)
		hum.Animator:LoadAnimation(WeaponAnimationsFolder[toolName].Attacking.Feint):Play()
		task.spawn(function()
			if char:GetAttribute("Combo") ~= 4 then 
				task.wait(.3)
				char:SetAttribute("Attacking", false)
				char:SetAttribute("Swing", false)
			else
				task.wait(1.6)
				char:SetAttribute("Attacking", false)
				char:SetAttribute("Swing", false)
			end
		end)
		local FeintSound = WeaponSoundFolder:WaitForChild(toolName):WaitForChild("Feint"):Clone()
		FeintSound.Parent = char:WaitForChild("HumanoidRootPart")
		FeintSound:Play()
		Debris:AddItem(FeintSound,1)
	end
end

--||Heavy attack||--
local function Heavy_Attack(char,hum,toolName,action,trail,newHitbox,HeavyHitboxSize)
	if action == "Heavy" then --the heavy attack stuff
		ServerCombatModule.stopAnims(hum)
		local HeavyAnim = hum.Animator:LoadAnimation(WeaponAnimationsFolder[toolName].Attacking.Heavy) --stopping the other animations and playing our heavy animation
		HeavyAnim:Play()  

		newHitbox.Size = HeavyHitboxSize --setting our hitbox size to the heavy hitbox size

		hum.WalkSpeed = 3 -- making the player even slower

		local HeavySound = WeaponSoundFolder:WaitForChild(toolName):WaitForChild("Heavy"):Clone() -- the sound is very important
		HeavySound.Parent = char.HumanoidRootPart
		HeavySound:Play()
		Debris:AddItem(HeavySound,4)

		VFX_Event:FireAllClients("HeavyAttack", char) -- for that Kanji character above the players head

		HeavyAnim.KeyframeReached:Connect(function(kf) -- same stuff as for the normal attacks
			if kf == "Hit" then
				char:SetAttribute("Attacking", false)

				Hitbox_Start_Stop(char,newHitbox)
			end
		end)

		HeavyAnim.Stopped:Connect(function()  --when the animation stops
			task.delay(.1,function() newHitbox:Destroy() end)
			if HeavySound then task.delay(.3,function() HeavySound:Destroy() end) end --destroy the heavy sound at the end of the animation to be able to continue attacking normaly
			char:SetAttribute("Swing", false)
			if char:FindFirstChild(toolName) and trail then  trail.Enabled = false end
			if not char:GetAttribute("IsBlocking") then hum.WalkSpeed = 14 hum.JumpHeight = 6.5	 end			
		end)		
	end
end



swingEvent.OnServerEvent:Connect(function(plr, action,toolName)
	local char = plr.Character
	local hum = char:WaitForChild("Humanoid")
	local hrp = char:WaitForChild("HumanoidRootPart")

	local attacking = char:GetAttribute("Attacking")
	local stunned = char:GetAttribute("Stunned")
	local isRagdoll = char:GetAttribute("IsRagdoll")
	local swing = char:GetAttribute("Swing")
	local blocking = char:GetAttribute("IsBlocking")	
	local Heavy = char.HumanoidRootPart:FindFirstChild(WeaponSoundFolder[toolName]:FindFirstChild("Heavy").Name) 

	if attacking or stunned or swing or isRagdoll or blocking or Heavy then return end

	local WeaponStats = WeaponInfoModule:getweapon(toolName) --getting all the weapon stats, Located in -> ServerStorage -> Modules -> WeaponInfo
	local Damage = WeaponStats.Damage	
	local SwingReset = WeaponStats.SwingReset
	local StunTime = WeaponStats.StunTime
	local HitboxSize = WeaponStats.HitboxSize
	local HitboxOffset = WeaponStats.HitboxOffset
	local HeavyDamage = WeaponStats.HeavyDamage
	local HeavyHitboxSize = WeaponStats.HeavyHitboxSize

	Feinting(char,hum,action,toolName) --feinting function

	char:SetAttribute("Attacking", true)
	char:SetAttribute("Swing", true)

	ServerCombatModule.ChangeCombo(char, WeaponStats.MaxCombo)
	ServerCombatModule.stopAnims(hum)

	hum.WalkSpeed = 7
	hum.JumpHeight = 0

	local ragdoll = false


	local newHitbox = HitboxModule.CreateHitbox() --creating the hitbox
	newHitbox.Size = HitboxSize
	newHitbox.CFrame = hrp
	newHitbox.Offset = HitboxOffset
	newHitbox.OverlapParams = params
	newHitbox.AutoDestroy = false

	--// Hoss modification
	local Trail
	local Body = char:FindFirstChild(toolName):FindFirstChild("BodyAttach")
	if Body then
		Trail = Body:FindFirstChild("Trail")
		if Trail then Trail.Enabled = true end
	end
	--//

	Heavy_Attack(char,hum,toolName,action,Trail,newHitbox,HeavyHitboxSize) --heavy attack function

	newHitbox.Touched:Connect(function(hit,enemyHum)
		if enemyHum ~= hum then			
			local isHeavyHit = char.HumanoidRootPart:FindFirstChild(WeaponSoundFolder[toolName]:FindFirstChild("Heavy").Name) --checking if the player has the heavy sound, if yes then we know that he is currently using his heavy attack 
			if enemyHum.Parent:GetAttribute("Parrying") and not isHeavyHit and ServerCombatModule.CheckInfront(char,enemyHum.Parent) then ServerCombatModule.Parrying(char,enemyHum.Parent) return end
			if enemyHum.Parent:GetAttribute("IsBlocking") and not isHeavyHit and ServerCombatModule.CheckInfront(char,enemyHum.Parent) then ServerCombatModule.Blocking(enemyHum.Parent, Damage) return end
			if enemyHum.Parent:GetAttribute("IsRagdoll") or enemyHum.Parent:GetAttribute("Iframes") or enemyHum.Health <= 0 then return end

			if char.HumanoidRootPart:FindFirstChild(WeaponSoundFolder[toolName]:FindFirstChild("Heavy").Name) then char:SetAttribute("Combo",4) end

			local enemyChar = enemyHum.Parent
			local enemyHrp = enemyChar:WaitForChild("HumanoidRootPart")

			local center = (enemyHrp.Position - hrp.Position).Unit
			local strength = 10

			--enemyHrp.CFrame = CFrame.lookAt(enemyHrp.Position, hrp.Position)--makes the enemy look at you, disabled it cuz it doesnt look good lol
			ServerCombatModule.stopAnims(enemyHum)
			local currCombo = char:GetAttribute("Combo")
			if currCombo <= 4 and not isHeavyHit then
				enemyHum.Animator:LoadAnimation(WeaponAnimationsFolder[toolName].Hit["Hit"..currCombo]):Play()
			elseif isHeavyHit then
				enemyHum.Animator:LoadAnimation(WeaponAnimationsFolder[toolName].Hit.HeavyHit):Play()
				local toothdrop = math.random(1, 7)
				if toothdrop == 1 then
					local tooth = game.ReplicatedStorage.Tooth:Clone()
					tooth.Parent = game.Workspace
					tooth.Position = enemyHum.Parent.Head.Position
				end
			end

			if currCombo == Max_Combo and not isHeavyHit then
				ragdoll = false
				strength = 43
			end 

			local kockback = center * strength

			if char.HumanoidRootPart:FindFirstChild(WeaponSoundFolder[toolName]:FindFirstChild("Heavy").Name) then Damage = HeavyDamage end
			HitServiceModule.Hit(enemyHum,Damage,StunTime,kockback,ragdoll,StunTime)

			VFX_Event:FireAllClients("SwordHit", enemyChar, currCombo)
			VFX_Event:FireClient(plr,"SwordHitShake")

			local HitSound = WeaponSoundFolder:WaitForChild(toolName):WaitForChild("Hit"):Clone()
			HitSound.Parent = char:WaitForChild("HumanoidRootPart")
			HitSound:Play()
			Debris:AddItem(HitSound,2)
			

			if currCombo ~= Max_Combo then
				local bv = Instance.new("BodyVelocity")
				bv.MaxForce = Vector3.new(math.huge,0,math.huge)
				bv.P = 100000
				bv.Velocity = kockback
				bv.Parent = hrp

				Debris:AddItem(bv,.2)
			end

		end
	end)

	local swingAnim = ServerCombatModule.getSwingAnimation(char,toolName, WeaponStats.MaxComboChangeCombo)
	local playSwingAnimation = hum.Animator:LoadAnimation(swingAnim)

	task.delay(WeaponStats.StartTime or 0.4, function()
		if newHitbox then
			Hitbox_Start_Stop(char,newHitbox)
		end

		if char:GetAttribute("Combo") == Max_Combo then --if the combo is the last hit then let him not attack for longer
			task.wait(1) 
		else
			task.wait(SwingReset)
		end
	end)

	if WeaponStats.StartTime then
		task.delay(WeaponStats.StartTime + 0.1, function()
			char:SetAttribute("Attacking", false)
			char:SetAttribute("Swing", false)
			if char:FindFirstChild(toolName) and Trail then  Trail.Enabled = false end
			if not char:GetAttribute("IsBlocking") then hum.WalkSpeed = 14 hum.JumpHeight = 6.5	 end			
		end)
	else
		task.delay( 0.6, function()
			char:SetAttribute("Attacking", false)
			char:SetAttribute("Swing", false)
			if char:FindFirstChild(toolName) and Trail then  Trail.Enabled = false end
			if not char:GetAttribute("IsBlocking") then hum.WalkSpeed = 14 hum.JumpHeight = 6.5	 end			
		end)
	end

	if not char.HumanoidRootPart:FindFirstChild(WeaponSoundFolder[toolName]:FindFirstChild("Heavy").Name) then playSwingAnimation:Play() end
	local SwordSwingSound = WeaponSoundFolder:WaitForChild(toolName):WaitForChild("Swing"):Clone()
	SwordSwingSound.Parent = char:WaitForChild("HumanoidRootPart")
	SwordSwingSound:Play()
	Debris:AddItem(SwordSwingSound,1)
end)

