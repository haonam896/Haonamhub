-- Knife > Client (LocalScript) — Non-gore takedown, cooldown 5s (text), cinematic blur+zoom
local tool = script.Parent
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer
local REMOTE = tool:WaitForChild("KnifeRE")

-- ==== THAM SỐ CHỈNH ====
local TAKEDOWN_RANGE = 4.5
local BACK_DOT_MIN = 0.5
local COOLDOWN_SECS = 5

-- Animation IDs (điền nếu có; để "" nếu chưa có)
local KNIFE_EQUIP_ANIM_ID = "" -- ví dụ: "rbxassetid://1234567890"
local KNIFE_TAKEDOWN_ANIM_ID = ""

-- Camera/Cinematic
local DO_BLUR = true
local BLUR_SIZE = 16
local FOV_IN = 55
local FOV_OUT = 70

-- ==== TRẠNG THÁI ====
local char, humanoid, animator
local equipTrack, takedownTrack
local countdownGui, countdownLabel
local cooldownUntil = 0

-- ==== UI Cooldown ====
local function ensureCountdownUI()
	if countdownGui and countdownLabel then return end
	countdownGui = Instance.new("ScreenGui")
	countdownGui.Name = "KnifeCooldownUI"
	countdownGui.ResetOnSpawn = false
	countdownGui.Parent = player:WaitForChild("PlayerGui")

	countdownLabel = Instance.new("TextLabel")
	countdownLabel.Name = "CooldownLabel"
	countdownLabel.Size = UDim2.fromOffset(180, 40)
	countdownLabel.AnchorPoint = Vector2.new(0.5, 0.5)
	countdownLabel.Position = UDim2.new(0.5, 0, 0.85, 0)
	countdownLabel.BackgroundTransparency = 0.25
	countdownLabel.BackgroundColor3 = Color3.new(0, 0, 0)
	countdownLabel.TextColor3 = Color3.new(1, 1, 1)
	countdownLabel.Font = Enum.Font.GothamBold
	countdownLabel.TextScaled = true
	countdownLabel.Visible = false
	countdownLabel.Text = "Ready"
	countdownLabel.Parent = countdownGui
end

local function setCountdownVisible(v)
	ensureCountdownUI()
	countdownLabel.Visible = v
end

local function updateCountdownText()
	if not countdownLabel then return end
	local remain = math.max(0, math.ceil(cooldownUntil - os.clock()))
	if remain > 0 then
		countdownLabel.Text = "Cooldown: " .. remain .. "s"
	else
		countdownLabel.Text = "Ready"
	end
end

-- ==== Hiệu ứng camera ngắn ====
local function playCinematicShort()
	local cam = workspace.CurrentCamera
	if not cam then return end

	local blur
	if DO_BLUR then
		blur = Instance.new("BlurEffect")
		blur.Size = 0
		blur.Parent = game:GetService("Lighting")
	end

	local tweenIn = TweenService:Create(
		cam, TweenInfo.new(0.15, Enum.EasingStyle.Sine, Enum.EasingDirection.Out),
		{ FieldOfView = FOV_IN }
	)
	local tweenOut = TweenService:Create(
		cam, TweenInfo.new(0.25, Enum.EasingStyle.Sine, Enum.EasingDirection.Out),
		{ FieldOfView = FOV_OUT }
	)

	local blurIn, blurOut
	if blur then
		blurIn = TweenService:Create(blur, TweenInfo.new(0.15), { Size = BLUR_SIZE })
		blurOut = TweenService:Create(blur, TweenInfo.new(0.25), { Size = 0 })
	end

	tweenIn:Play()
	if blurIn then blurIn:Play() end
	task.wait(0.18)
	tweenOut:Play()
	if blurOut then blurOut:Play() end
	if blur then
		blurOut.Completed:Wait()
		blur:Destroy()
	end
end

-- ==== Helpers ====
local function getHumanoid(model)
	return model and model:FindFirstChildOfClass("Humanoid")
end

local function validBackstab(myHRP, targetModel)
	if not myHRP or not targetModel then return false end
	local tHRP = targetModel:FindFirstChild("HumanoidRootPart") or targetModel:FindFirstChild("UpperTorso") or targetModel:FindFirstChild("Torso")
	local tHum = getHumanoid(targetModel)
	if not (tHRP and tHum) or tHum.Health <= 0 then return false end

	if (tHRP.Position - myHRP.Position).Magnitude > TAKEDOWN_RANGE then return false end
	local targetLook = tHRP.CFrame.LookVector
	local toAttacker = (myHRP.Position - tHRP.Position).Unit
	if targetLook:Dot(toAttacker) <= BACK_DOT_MIN then return false end
	return true
end

local function findNearestValidTarget()
	char = player.Character or player.CharacterAdded:Wait()
	local myHRP = char and char:FindFirstChild("HumanoidRootPart")
	if not myHRP then return end

	local nearest, nearestDist
	for _, m in ipairs(workspace:GetChildren()) do
		local hum = getHumanoid(m)
		if hum and m ~= char and hum.Health > 0 then
			local tHRP = m:FindFirstChild("HumanoidRootPart") or m:FindFirstChild("UpperTorso") or m:FindFirstChild("Torso")
			if tHRP then
				local d = (tHRP.Position - myHRP.Position).Magnitude
				if d <= TAKEDOWN_RANGE and validBackstab(myHRP, m) then
					if not nearest or d < nearestDist then
						nearest, nearestDist = m, d
					end
				end
			end
		end
	end
	return nearest
end

-- ==== Equip / Anim / SFX ====
tool.Equipped:Connect(function()
	char = player.Character or player.CharacterAdded:Wait()
	humanoid = char and getHumanoid(char)
	if humanoid then
		animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)
		if KNIFE_EQUIP_ANIM_ID ~= "" then
			local anim = Instance.new("Animation")
			anim.AnimationId = KNIFE_EQUIP_ANIM_ID
			equipTrack = animator:LoadAnimation(anim)
			equipTrack.Priority = Enum.AnimationPriority.Action
			equipTrack:Play()
		end
	end

	local equipSfx = tool:FindFirstChild("EquipSfx")
	if equipSfx and equipSfx:IsA("Sound") then equipSfx:Play() end

	ensureCountdownUI()
	updateCountdownText()
	setCountdownVisible(true)

	if not tool:GetAttribute("UIConn") then
		local conn = RunService.RenderStepped:Connect(updateCountdownText)
		tool:SetAttribute("UIConn", true)
		tool.AncestryChanged:Connect(function(_, parent)
			if not parent and conn.Connected then
				conn:Disconnect()
			end
		end)
	end
end)

tool.Unequipped:Connect(function()
	if equipTrack then pcall(function() equipTrack:Stop(0.1) end) end
	setCountdownVisible(false)
end)

-- ==== Kích hoạt ====
tool.Activated:Connect(function()
	if os.clock() < cooldownUntil then return end
	local target = findNearestValidTarget()
	if target then
		REMOTE:FireServer("takedown", target)
	else
		-- (tuỳ chọn) hiển thị thông báo: "Hãy đứng sau lưng và lại gần!"
	end
end)

-- ==== Nhận phản hồi từ Server ====
REMOTE.OnClientEvent:Connect(function(evt, data)
	if evt == "cooldown_start" then
		cooldownUntil = os.clock() + (data and data.duration or COOLDOWN_SECS)
		setCountdownVisible(true)
	elseif evt == "play_cinematic" then
		if animator and KNIFE_TAKEDOWN_ANIM_ID ~= "" then
			local anim = Instance.new("Animation")
			anim.AnimationId = KNIFE_TAKEDOWN_ANIM_ID
			takedownTrack = animator:LoadAnimation(anim)
			takedownTrack.Priority = Enum.AnimationPriority.Action
			takedownTrack:Play()
		end
		local sfx = tool:FindFirstChild("TakedownSfx")
		if sfx and sfx:IsA("Sound") then sfx:Play() end
		playCinematicShort()
	end
end)-- Knife > Server (Script) — Team Check bật, non-gore knockout, cooldown 5s
local tool = script.Parent
local REMOTE = tool:WaitForChild("KnifeRE")

local COOLDOWN_SECS = 5
local TAKEDOWN_RANGE = 4.5
local BACK_DOT_MIN = 0.5
local TEAM_CHECK = true -- đã bật

local lastUse = {} -- player -> timestamp

local function humOf(model)
	return model and model:FindFirstChildOfClass("Humanoid")
end

local function validTakedown(attackerChar, targetModel)
	if not attackerChar or not targetModel or attackerChar == targetModel then return false end
	local aHRP = attackerChar:FindFirstChild("HumanoidRootPart")
	local tHRP = targetModel:FindFirstChild("HumanoidRootPart") or targetModel:FindFirstChild("UpperTorso") or targetModel:FindFirstChild("Torso")
	local tHum = humOf(targetModel)
	if not (aHRP and tHRP and tHum) then return false end
	if tHum.Health <= 0 then return false end

	if (tHRP.Position - aHRP.Position).Magnitude > TAKEDOWN_RANGE then return false end

	local targetLook = tHRP.CFrame.LookVector
	local toAttacker = (aHRP.Position - tHRP.Position).Unit
	if targetLook:Dot(toAttacker) <= BACK_DOT_MIN then return false end

	-- Team check (không cho hạ gục đồng đội)
	local Players = game:GetService("Players")
	local attackerPlr = Players:GetPlayerFromCharacter(attackerChar)
	local targetPlr = Players:GetPlayerFromCharacter(targetModel)
	if TEAM_CHECK and attackerPlr and targetPlr and attackerPlr.Team ~= nil and attackerPlr.Team == targetPlr.Team then
		return false
	end

	return true
end

REMOTE.OnServerEvent:Connect(function(player, action, targetModel)
	if action ~= "takedown" then return end

	-- Cooldown
	local now = os.clock()
	if (lastUse[player] or 0) + COOLDOWN_SECS > now then return end

	local attackerChar = player.Character
	if not attackerChar then return end
	if typeof(targetModel) ~= "Instance" or not targetModel:IsDescendantOf(workspace) then return end

	if not validTakedown(attackerChar, targetModel) then return end
	lastUse[player] = now

	-- Canh vị trí attacker đứng sau lưng cho mượt hơn
	local aHRP = attackerChar:FindFirstChild("HumanoidRootPart")
	local tHRP = targetModel:FindFirstChild("HumanoidRootPart") or targetModel:FindFirstChild("UpperTorso") or targetModel:FindFirstChild("Torso")
	if aHRP and tHRP then
		local behind = tHRP.CFrame * CFrame.new(0, 0, -1.2)
		aHRP.CFrame = CFrame.new(behind.Position, tHRP.Position)
	end

	-- Gửi yêu cầu cinematic/animation xuống Client
	REMOTE:FireClient(player, "play_cinematic", {})

	-- Hạ gục: giảm HP về 0 (non-gore)
	local tHum = humOf(targetModel)
	if tHum then
		tHum:TakeDamage(tHum.Health)
	end

	-- Bắt đầu cooldown 5s cho UI chữ đếm ngược
	REMOTE:FireClient(player, "cooldown_start", { duration = COOLDOWN_SECS })
end)
