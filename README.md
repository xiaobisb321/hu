-- 仅限你自己拥有或有权修改的 Roblox Studio 项目
-- 放置位置：StarterPlayer > StarterPlayerScripts 中的 LocalScript

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local FLY_SPEED = 60
local MIN_SPEED = 10
local MAX_SPEED = 500
local flying = false

local isTouch = UIS.TouchEnabled

local character, root, humanoid, bv, bg

-- 手机备用升降
local upPressed = false
local downPressed = false

-- 摇杆
local joystickVector = Vector2.new(0, 0)
local joystickActive = false
local JOYSTICK_RADIUS = 45

-- 相机俯仰死区
local PITCH_DEADZONE = 0.15

-- ===== 根 GUI =====
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlightControlGui"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = playerGui

-- ===== 悬浮球 =====
local BTN_SIZE = 60
local floatingBtn = Instance.new("TextButton")
floatingBtn.Name = "FloatingButton"
floatingBtn.Size = UDim2.new(0, BTN_SIZE, 0, BTN_SIZE)
-- 顶部中央
floatingBtn.Position = UDim2.new(0.5, -BTN_SIZE / 2, 0, 20)
floatingBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
floatingBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
floatingBtn.Font = Enum.Font.GothamBold
floatingBtn.TextSize = 22
floatingBtn.Text = "飞"
floatingBtn.AutoButtonColor = false
floatingBtn.Active = true
floatingBtn.Parent = screenGui

local floatCorner = Instance.new("UICorner")
floatCorner.CornerRadius = UDim.new(1, 0)
floatCorner.Parent = floatingBtn

local floatStroke = Instance.new("UIStroke")
floatStroke.Color = Color3.fromRGB(255, 255, 255)
floatStroke.Thickness = 2
floatStroke.Transparency = 0.4
floatStroke.Parent = floatingBtn

-- ===== 控制面板 =====
local PANEL_W = 270
local PANEL_H = 235

local frame = Instance.new("Frame")
frame.Name = "Panel"
frame.Size = UDim2.new(0, PANEL_W, 0, PANEL_H)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.BackgroundTransparency = 0.1
frame.BorderSizePixel = 0
frame.Active = true
frame.Visible = false
frame.Parent = screenGui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 10)
frameCorner.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 36)
title.BackgroundTransparency = 1
title.Text = "飞行控制"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 20
title.Parent = frame

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(1, -20, 0, 40)
toggleBtn.Position = UDim2.new(0, 10, 0, 42)
toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 18
toggleBtn.Text = "开启飞行"
toggleBtn.Parent = frame

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 8)
toggleCorner.Parent = toggleBtn

local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(1, -20, 0, 22)
speedLabel.Position = UDim2.new(0, 10, 0, 92)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "速度 (10 - 500)"
speedLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
speedLabel.TextXAlignment = Enum.TextXAlignment.Left
speedLabel.Font = Enum.Font.Gotham
speedLabel.TextSize = 14
speedLabel.Parent = frame

local speedBox = Instance.new("TextBox")
speedBox.Size = UDim2.new(1, -100, 0, 36)
speedBox.Position = UDim2.new(0, 10, 0, 118)
speedBox.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
speedBox.TextColor3 = Color3.fromRGB(255, 255, 255)
speedBox.PlaceholderText = "60"
speedBox.Text = tostring(FLY_SPEED)
speedBox.Font = Enum.Font.Gotham
speedBox.TextSize = 18
speedBox.ClearTextOnFocus = false
speedBox.Parent = frame

local speedCorner = Instance.new("UICorner")
speedCorner.CornerRadius = UDim.new(0, 8)
speedCorner.Parent = speedBox

local applyBtn = Instance.new("TextButton")
applyBtn.Size = UDim2.new(0, 80, 0, 36)
applyBtn.Position = UDim2.new(1, -90, 0, 118)
applyBtn.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
applyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
applyBtn.Font = Enum.Font.GothamBold
applyBtn.TextSize = 16
applyBtn.Text = "应用"
applyBtn.Parent = frame

local applyCorner = Instance.new("UICorner")
applyCorner.CornerRadius = UDim.new(0, 8)
applyCorner.Parent = applyBtn

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -20, 0, 55)
statusLabel.Position = UDim2.new(0, 10, 0, 164)
statusLabel.BackgroundTransparency = 1
statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
statusLabel.Font = Enum.Font.Gotham
statusLabel.TextSize = 13
statusLabel.TextWrapped = true
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Text = ""
statusLabel.Parent = frame

-- ===== 面板跟随悬浮球 =====
local function updatePanelPosition()
	local px = floatingBtn.Position.X.Scale
	local py = floatingBtn.Position.Y.Scale
	local ox = floatingBtn.Position.X.Offset
	local oy = floatingBtn.Position.Y.Offset

	-- 面板水平居中于悬浮球，垂直在球下面 8px
	local panelOffsetX = ox + BTN_SIZE / 2 - PANEL_W / 2
	local panelOffsetY = oy + BTN_SIZE + 8

	frame.Position = UDim2.new(px, panelOffsetX, py, panelOffsetY)
end

updatePanelPosition()

-- ===== 悬浮球拖动 + 点击 =====
local dragging = false
local dragStart = Vector2.new(0, 0)
local startPos = UDim2.new(0, 0, 0, 0)
local movedDistance = 0

local DRAG_THRESHOLD = 8

floatingBtn.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = Vector2.new(input.Position.X, input.Position.Y)
		startPos = floatingBtn.Position
		movedDistance = 0
		floatingBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
	end
end)

UIS.InputChanged:Connect(function(input)
	if not dragging then return end
	if input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch then
		local cur = Vector2.new(input.Position.X, input.Position.Y)
		local delta = cur - dragStart
		movedDistance = delta.Magnitude

		floatingBtn.Position = UDim2.new(
			startPos.X.Scale, startPos.X.Offset + delta.X,
			startPos.Y.Scale, startPos.Y.Offset + delta.Y
		)
		updatePanelPosition()
	end
end)

UIS.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
		if dragging then
			dragging = false
			floatingBtn.BackgroundColor3 = flying
				and Color3.fromRGB(200, 50, 50)
				or Color3.fromRGB(0, 170, 0)

			-- 移动距离很小视为点击
			if movedDistance < DRAG_THRESHOLD then
				frame.Visible = not frame.Visible
				if frame.Visible then
					updatePanelPosition()
				end
			end
		end
	end
end)

-- ===== 手机摇杆 =====
local joystickBase = Instance.new("Frame")
joystickBase.Name = "JoystickBase"
joystickBase.Size = UDim2.new(0, 160, 0, 160)
joystickBase.Position = UDim2.new(1, -180, 1, -180)
joystickBase.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
joystickBase.BackgroundTransparency = 0.4
joystickBase.BorderSizePixel = 0
joystickBase.Active = true
joystickBase.Visible = isTouch
joystickBase.Parent = screenGui

local baseCorner = Instance.new("UICorner")
baseCorner.CornerRadius = UDim.new(1, 0)
baseCorner.Parent = joystickBase

local joystickKnob = Instance.new("Frame")
joystickKnob.Name = "JoystickKnob"
joystickKnob.Size = UDim2.new(0, 70, 0, 70)
joystickKnob.AnchorPoint = Vector2.new(0.5, 0.5)
joystickKnob.Position = UDim2.new(0.5, 0, 0.5, 0)
joystickKnob.BackgroundColor3 = Color3.fromRGB(80, 140, 220)
joystickKnob.BackgroundTransparency = 0.1
joystickKnob.BorderSizePixel = 0
joystickKnob.Parent = joystickBase

local knobCorner = Instance.new("UICorner")
knobCorner.CornerRadius = UDim.new(1, 0)
knobCorner.Parent = joystickKnob

local function updateJoystick(touchPos)
	local baseCenter = joystickBase.AbsolutePosition + joystickBase.AbsoluteSize / 2
	local delta = touchPos - baseCenter
	local dist = delta.Magnitude

	if dist > JOYSTICK_RADIUS then
		delta = delta.Unit * JOYSTICK_RADIUS
	end

	joystickKnob.Position = UDim2.new(0.5, delta.X, 0.5, delta.Y)
	joystickVector = delta / JOYSTICK_RADIUS
end

local function resetJoystick()
	joystickActive = false
	joystickVector = Vector2.new(0, 0)
	joystickKnob.Position = UDim2.new(0.5, 0, 0.5, 0)
end

joystickBase.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch
		or input.UserInputType == Enum.UserInputType.MouseButton1 then
		joystickActive = true
		updateJoystick(Vector2.new(input.Position.X, input.Position.Y))
	end
end)

joystickBase.InputChanged:Connect(function(input)
	if not joystickActive then return end
	if input.UserInputType == Enum.UserInputType.Touch
		or input.UserInputType == Enum.UserInputType.MouseMovement then
		updateJoystick(Vector2.new(input.Position.X, input.Position.Y))
	end
end)

joystickBase.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch
		or input.UserInputType == Enum.UserInputType.MouseButton1 then
		resetJoystick()
	end
end)

-- ===== 手机备用升降按钮 =====
local function createHoldButton(text, pos)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0, 60, 0, 60)
	btn.Position = pos
	btn.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
	btn.BackgroundTransparency = 0.15
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 18
	btn.Text = text
	btn.AutoButtonColor = false
	btn.Visible = isTouch
	btn.Parent = screenGui

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 12)
	corner.Parent = btn

	local normalColor = Color3.fromRGB(45, 45, 45)
	local pressedColor = Color3.fromRGB(80, 140, 220)

	btn.MouseButton1Down:Connect(function()
		btn.BackgroundColor3 = pressedColor
	end)

	local function release()
		btn.BackgroundColor3 = normalColor
	end

	btn.MouseButton1Up:Connect(release)
	btn.MouseLeave:Connect(release)
	btn.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch
			or input.UserInputType == Enum.UserInputType.MouseButton1 then
			release()
		end
	end)

	return btn
end

local upBtn = createHoldButton("升", UDim2.new(1, -260, 1, -180))
local downBtn = createHoldButton("降", UDim2.new(1, -260, 1, -110))

upBtn.MouseButton1Down:Connect(function() upPressed = true end)
upBtn.MouseButton1Up:Connect(function() upPressed = false end)
upBtn.MouseLeave:Connect(function() upPressed = false end)
upBtn.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch then
		upPressed = false
	end
end)

downBtn.MouseButton1Down:Connect(function() downPressed = true end)
downBtn.MouseButton1Up:Connect(function() downPressed = false end)
downBtn.MouseLeave:Connect(function() downPressed = false end)
downBtn.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch then
		downPressed = false
	end
end)

-- ===== 更新 UI =====
local function updateUI()
	if flying then
		toggleBtn.Text = "关闭飞行"
		toggleBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
		floatingBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
	else
		toggleBtn.Text = "开启飞行"
		toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
		floatingBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
	end

	local tip
	if isTouch then
		tip = "手机：摇杆水平移动，镜头抬高上升、压低下降"
	else
		tip = "电脑：F 开关，WASD 水平移动，镜头抬高上升、压低下降"
	end

	statusLabel.Text = string.format(
		"状态：%s | 当前速度：%d\n%s",
		flying and "开启" or "关闭",
		FLY_SPEED,
		tip
	)
end

-- ===== 飞行开关 =====
local function setFlight(enabled)
	if enabled and (not root or not humanoid or not bv or not bg) then
		return
	end

	flying = enabled

	if flying then
		bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
		bg.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
		humanoid.PlatformStand = true
	else
		if bv then
			bv.MaxForce = Vector3.zero
			bv.Velocity = Vector3.zero
		end
		if bg then
			bg.MaxTorque = Vector3.zero
		end
		if humanoid then
			humanoid.PlatformStand = false
		end

		upPressed = false
		downPressed = false
		resetJoystick()
	end

	updateUI()
end

-- ===== 应用速度 =====
local function applySpeed()
	local n = tonumber(speedBox.Text)
	if not n then
		speedBox.Text = tostring(FLY_SPEED)
		return
	end

	n = math.clamp(math.floor(n), MIN_SPEED, MAX_SPEED)
	FLY_SPEED = n
	speedBox.Text = tostring(FLY_SPEED)
	updateUI()
end

-- ===== 角色初始化 =====
local function setupCharacter(newCharacter)
	character = newCharacter
	root = character:WaitForChild("HumanoidRootPart")
	humanoid = character:WaitForChild("Humanoid")

	if bv then bv:Destroy() end
	if bg then bg:Destroy() end

	bv = Instance.new("BodyVelocity")
	bv.Name = "FlightVelocity"
	bv.MaxForce = Vector3.zero
	bv.Velocity = Vector3.zero
	bv.Parent = root

	bg = Instance.new("BodyGyro")
	bg.Name = "FlightGyro"
	bg.MaxTorque = Vector3.zero
	bg.P = 10000
	bg.D = 500
	bg.CFrame = root.CFrame
	bg.Parent = root

	flying = false
	upPressed = false
	downPressed = false
	resetJoystick()
	updateUI()
end

if player.Character then
	setupCharacter(player.Character)
end

player.CharacterAdded:Connect(setupCharacter)

-- ===== 面板按钮事件 =====
toggleBtn.MouseButton1Click:Connect(function()
	setFlight(not flying)
end)

applyBtn.MouseButton1Click:Connect(applySpeed)

speedBox.FocusLost:Connect(function()
	applySpeed()
end)

-- ===== 键盘快捷键 =====
UIS.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end

	if input.KeyCode == Enum.KeyCode.F then
		setFlight(not flying)
	end
end)

-- ===== 飞行主循环 =====
RunService.RenderStepped:Connect(function()
	if not flying or not root or not humanoid or not bv or not bg then
		return
	end

	local camera = workspace.CurrentCamera
	if not camera then return end

	local look = camera.CFrame.LookVector

	local horizontalForward = Vector3.new(look.X, 0, look.Z)
	if horizontalForward.Magnitude > 0 then
		horizontalForward = horizontalForward.Unit
	end

	local horizontalRight = Vector3.new(
		camera.CFrame.RightVector.X, 0, camera.CFrame.RightVector.Z
	)
	if horizontalRight.Magnitude > 0 then
		horizontalRight = horizontalRight.Unit
	end

	local moveDir = Vector3.zero

	-- 电脑 WASD 水平
	if UIS:IsKeyDown(Enum.KeyCode.W) then
		moveDir = moveDir + horizontalForward
	end
	if UIS:IsKeyDown(Enum.KeyCode.S) then
		moveDir = moveDir - horizontalForward
	end
	if UIS:IsKeyDown(Enum.KeyCode.D) then
		moveDir = moveDir + horizontalRight
	end
	if UIS:IsKeyDown(Enum.KeyCode.A) then
		moveDir = moveDir - horizontalRight
	end

	-- 手机摇杆水平
	if joystickActive and joystickVector.Magnitude > 0.15 then
		local jx = joystickVector.X
		local jy = joystickVector.Y
		moveDir = moveDir + horizontalForward * (-jy)
		moveDir = moveDir + horizontalRight * jx
	end

	-- 相机俯仰升降
	local lookY = look.Y
	local verticalFromCamera = 0

	if lookY > PITCH_DEADZONE then
		verticalFromCamera = (lookY - PITCH_DEADZONE) / (1 - PITCH_DEADZONE)
	elseif lookY < -PITCH_DEADZONE then
		verticalFromCamera = (lookY + PITCH_DEADZONE) / (1 - PITCH_DEADZONE)
	end

	if verticalFromCamera ~= 0 then
		moveDir = moveDir + Vector3.new(0, verticalFromCamera, 0)
	end

	-- 手机备用升降按钮
	if upPressed then
		moveDir = moveDir + Vector3.new(0, 1, 0)
	end
	if downPressed then
		moveDir = moveDir - Vector3.new(0, 1, 0)
	end

	if moveDir.Magnitude > 0 then
		moveDir = moveDir.Unit
	end

	bv.Velocity = moveDir * FLY_SPEED
	bg.CFrame = CFrame.new(root.Position, root.Position + look)
end)

updateUI()
