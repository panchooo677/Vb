--[[
    Mobile Freecam — LocalScript
    Place inside StarterPlayerScripts.
]]

local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local UserInputService   = game:GetService("UserInputService")

local player    = Players.LocalPlayer
local camera    = workspace.CurrentCamera
local playerGui = player:WaitForChild("PlayerGui")

local playerScripts   = player:WaitForChild("PlayerScripts")
local playerModule    = nil
local controlsModule  = nil

task.spawn(function()
	local ok, mod = pcall(function()
		return require(playerScripts:WaitForChild("PlayerModule"))
	end)
	if ok and mod then
		playerModule   = mod
		controlsModule = mod:GetControls()
	end
end)

local DEFAULT_SPEED    = 50
local MIN_SPEED        = 5
local MAX_SPEED        = 1000
local SPEED_STEP       = 10
local LOOK_SENSITIVITY = 0.25
local TAP_THRESHOLD    = 10

local freecamOn    = false
local camSpeed     = DEFAULT_SPEED
local yaw          = 0
local pitch        = 0
local camPos       = Vector3.zero

local origCamType, origCamSubject
local origWalkSpeed, origJumpPower, origJumpHeight
local frozenPosition = nil

local lookTouch     = nil
local lastLookPos   = nil

local upHeld   = false
local downHeld = false

local dragModeOn = false

local enableFreecam
local disableFreecam

local function addCorner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius or 8)
	c.Parent = parent
end

local function addStroke(parent, color, thickness)
	local s = Instance.new("UIStroke")
	s.Color           = color or Color3.fromRGB(100, 100, 100)
	s.Thickness       = thickness or 1.5
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = parent
end

local function makeButton(props)
	local b = Instance.new("TextButton")
	b.BorderSizePixel = 0
	b.Active = true
	for k, v in pairs(props) do b[k] = v end
	return b
end

local gui = Instance.new("ScreenGui")
gui.Name           = "FreecamMobileGui"
gui.ResetOnSpawn   = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.DisplayOrder   = 10
gui.Parent         = playerGui

-- TOP BAR
local topBar = Instance.new("Frame")
topBar.Name                   = "TopBar"
topBar.Size                   = UDim2.new(0, 230, 0, 50)
topBar.Position               = UDim2.new(0.5, -115, 0, 14)
topBar.AnchorPoint            = Vector2.new(0, 0)
topBar.BackgroundTransparency = 1
topBar.Parent                 = gui
topBar.ZIndex                 = 50

-- TOGGLE BUTTON
local toggleBtn = makeButton({
	Name                   = "ToggleBtn",
	Size                   = UDim2.new(0, 150, 0, 50),
	Position               = UDim2.new(0, 0, 0, 0),
	AnchorPoint            = Vector2.new(0, 0),
	BackgroundColor3       = Color3.fromRGB(30, 30, 30),
	BackgroundTransparency = 0.15,
	Text                   = "📷 Freecam",
	TextColor3             = Color3.new(1, 1, 1),
	TextSize               = 16,
	Font                   = Enum.Font.GothamBold,
	AutoButtonColor        = false,
	Parent                 = topBar,
	ZIndex                 = 50,
})
addCorner(toggleBtn, 12)
addStroke(toggleBtn, Color3.fromRGB(90, 90, 90), 1.5)

-- LOCK BUTTON
local lockBtn = makeButton({
	Name                   = "LockBtn",
	Size                   = UDim2.new(0, 70, 0, 50),
	Position               = UDim2.new(0, 160, 0, 0),
	AnchorPoint            = Vector2.new(0, 0),
	BackgroundColor3       = Color3.fromRGB(50, 50, 50),
	BackgroundTransparency = 0.15,
	Text                   = "🔒",
	TextColor3             = Color3.fromRGB(180, 180, 180),
	TextSize               = 22,
	Font                   = Enum.Font.GothamBold,
	AutoButtonColor        = false,
	Visible                = false,
	Parent                 = topBar,
	ZIndex                 = 50,
})
addCorner(lockBtn, 12)
addStroke(lockBtn, Color3.fromRGB(90, 90, 90), 1.5)

-- CONTROLS CONTAINER
local controls = Instance.new("Frame")
controls.Name                   = "Controls"
controls.Size                   = UDim2.new(1, 0, 1, 0)
controls.BackgroundTransparency = 1
controls.Visible                = false
controls.Parent                 = gui

-- UP / DOWN BUTTONS
local function vertButton(name, label, yOffset)
	local b = makeButton({
		Name                   = name,
		Size                   = UDim2.new(0, 72, 0, 72),
		AnchorPoint            = Vector2.new(0.5, 0.5),
		Position               = UDim2.new(1, -80, 1, yOffset),
		BackgroundColor3       = Color3.fromRGB(50, 50, 50),
		BackgroundTransparency = 0.25,
		Text                   = label,
		TextColor3             = Color3.new(1, 1, 1),
		TextSize               = 28,
		Font                   = Enum.Font.GothamBold,
		Parent                 = controls,
	})
	addCorner(b, 14)
	addStroke(b, Color3.fromRGB(110, 110, 110))
	return b
end

local upBtn   = vertButton("UpBtn",   "▲", -260)
local downBtn = vertButton("DownBtn", "▼", -170)

-- SPEED PANEL
local speedPanel = Instance.new("Frame")
speedPanel.Name                   = "SpeedPanel"
speedPanel.Size                   = UDim2.new(0, 220, 0, 52)
speedPanel.AnchorPoint            = Vector2.new(0.5, 1)
speedPanel.Position               = UDim2.new(0.5, 0, 1, -16)
speedPanel.BackgroundColor3       = Color3.fromRGB(28, 28, 28)
speedPanel.BackgroundTransparency = 0.2
speedPanel.Active                 = true
speedPanel.Parent                 = controls
addCorner(speedPanel, 14)
addStroke(speedPanel, Color3.fromRGB(80, 80, 80))

local speedDown = makeButton({
	Name                   = "SpeedDown",
	Size                   = UDim2.new(0, 52, 1, 0),
	Position               = UDim2.new(0, 0, 0, 0),
	BackgroundTransparency = 1,
	Text                   = "−",
	TextColor3             = Color3.fromRGB(255, 110, 110),
	TextSize               = 30,
	Font                   = Enum.Font.GothamBold,
	Parent                 = speedPanel,
})

local speedLabel = Instance.new("TextLabel")
speedLabel.Name                   = "SpeedLabel"
speedLabel.Size                   = UDim2.new(1, -104, 1, 0)
speedLabel.Position               = UDim2.new(0, 52, 0, 0)
speedLabel.BackgroundTransparency = 1
speedLabel.Text                   = "Speed: " .. camSpeed
speedLabel.TextColor3             = Color3.new(1, 1, 1)
speedLabel.TextSize               = 15
speedLabel.Font                   = Enum.Font.GothamBold
speedLabel.Parent                 = speedPanel

local speedUp = makeButton({
	Name                   = "SpeedUp",
	Size                   = UDim2.new(0, 52, 1, 0),
	Position               = UDim2.new(1, -52, 0, 0),
	BackgroundTransparency = 1,
	Text                   = "+",
	TextColor3             = Color3.fromRGB(110, 255, 110),
	TextSize               = 30,
	Font                   = Enum.Font.GothamBold,
	Parent                 = speedPanel,
})

-- LOCK BUTTON VISUALS
local function updateLockVisuals()
	if dragModeOn then
		lockBtn.Text             = "🔓"
		lockBtn.BackgroundColor3 = Color3.fromRGB(30, 100, 50)
		lockBtn.TextColor3       = Color3.fromRGB(110, 255, 130)
		for _, child in ipairs(lockBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color = Color3.fromRGB(60, 150, 80)
			end
		end
		for _, child in ipairs(upBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color     = Color3.fromRGB(80, 200, 100)
				child.Thickness = 2.5
			end
		end
		for _, child in ipairs(downBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color     = Color3.fromRGB(80, 200, 100)
				child.Thickness = 2.5
			end
		end
	else
		lockBtn.Text             = "🔒"
		lockBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
		lockBtn.TextColor3       = Color3.fromRGB(180, 180, 180)
		for _, child in ipairs(lockBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color = Color3.fromRGB(90, 90, 90)
			end
		end
		for _, child in ipairs(upBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color     = Color3.fromRGB(110, 110, 110)
				child.Thickness = 1.5
			end
		end
		for _, child in ipairs(downBtn:GetChildren()) do
			if child:IsA("UIStroke") then
				child.Color     = Color3.fromRGB(110, 110, 110)
				child.Thickness = 1.5
			end
		end
	end
end

-- DRAGGABLE: TOP BAR
topBar.Active = true

do
	local state = {
		touch = nil, startPos = nil, startAnchor = nil,
		totalDist = 0, dragging = false,
	}

	topBar.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local pos = Vector2.new(input.Position.X, input.Position.Y)
		local function onRect(p, obj)
			local ap = obj.AbsolutePosition
			local as = obj.AbsoluteSize
			return p.X >= ap.X and p.X <= ap.X + as.X
				and p.Y >= ap.Y and p.Y <= ap.Y + as.Y
		end
		if onRect(pos, toggleBtn) or (lockBtn.Visible and onRect(pos, lockBtn)) then
			return
		end
		state.touch       = input
		state.startPos    = pos
		state.startAnchor = topBar.Position
		state.totalDist   = 0
		state.dragging    = false
	end)

	topBar.InputChanged:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local current = Vector2.new(input.Position.X, input.Position.Y)
		state.totalDist = (current - state.startPos).Magnitude
		if state.totalDist > TAP_THRESHOLD then
			state.dragging = true
			local deltaX = input.Position.X - state.startPos.X
			local deltaY = input.Position.Y - state.startPos.Y
			topBar.Position = UDim2.new(
				state.startAnchor.X.Scale,
				state.startAnchor.X.Offset + deltaX,
				state.startAnchor.Y.Scale,
				state.startAnchor.Y.Offset + deltaY
			)
		end
	end)

	topBar.InputEnded:Connect(function(input)
		if input ~= state.touch then return end
		state.touch    = nil
		state.dragging = false
	end)
end

-- DRAGGABLE: TOGGLE BUTTON
do
	local state = {
		touch = nil, startPos = nil, startAnchor = nil,
		totalDist = 0, dragging = false,
	}

	toggleBtn.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		state.touch       = input
		state.startPos    = Vector2.new(input.Position.X, input.Position.Y)
		state.startAnchor = topBar.Position
		state.totalDist   = 0
		state.dragging    = false
	end)

	toggleBtn.InputChanged:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local current = Vector2.new(input.Position.X, input.Position.Y)
		state.totalDist = (current - state.startPos).Magnitude
		if state.totalDist > TAP_THRESHOLD then
			state.dragging = true
			local deltaX = input.Position.X - state.startPos.X
			local deltaY = input.Position.Y - state.startPos.Y
			topBar.Position = UDim2.new(
				state.startAnchor.X.Scale,
				state.startAnchor.X.Offset + deltaX,
				state.startAnchor.Y.Scale,
				state.startAnchor.Y.Offset + deltaY
			)
		end
	end)

	toggleBtn.InputEnded:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		if state.totalDist <= TAP_THRESHOLD then
			if freecamOn then
				disableFreecam()
			else
				enableFreecam()
			end
		end
		state.touch    = nil
		state.dragging = false
	end)
end

-- DRAGGABLE: LOCK BUTTON
do
	local state = {
		touch = nil, startPos = nil, startAnchor = nil,
		totalDist = 0, dragging = false,
	}

	lockBtn.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		state.touch       = input
		state.startPos    = Vector2.new(input.Position.X, input.Position.Y)
		state.startAnchor = topBar.Position
		state.totalDist   = 0
		state.dragging    = false
	end)

	lockBtn.InputChanged:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local current = Vector2.new(input.Position.X, input.Position.Y)
		state.totalDist = (current - state.startPos).Magnitude
		if state.totalDist > TAP_THRESHOLD then
			state.dragging = true
			local deltaX = input.Position.X - state.startPos.X
			local deltaY = input.Position.Y - state.startPos.Y
			topBar.Position = UDim2.new(
				state.startAnchor.X.Scale,
				state.startAnchor.X.Offset + deltaX,
				state.startAnchor.Y.Scale,
				state.startAnchor.Y.Offset + deltaY
			)
		end
	end)

	lockBtn.InputEnded:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		if state.totalDist <= TAP_THRESHOLD then
			dragModeOn = not dragModeOn
			updateLockVisuals()
		end
		state.touch    = nil
		state.dragging = false
	end)
end

-- DRAGGABLE: UP BUTTON
do
	local state = {
		touch = nil, startPos = nil, startAnchor = nil,
		totalDist = 0, dragging = false,
	}

	upBtn.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		state.touch       = input
		state.startPos    = Vector2.new(input.Position.X, input.Position.Y)
		state.startAnchor = upBtn.Position
		state.totalDist   = 0
		state.dragging    = false
		if not dragModeOn then
			upHeld = true
		end
	end)

	upBtn.InputChanged:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local current = Vector2.new(input.Position.X, input.Position.Y)
		state.totalDist = (current - state.startPos).Magnitude
		if dragModeOn and state.totalDist > TAP_THRESHOLD then
			state.dragging = true
			upHeld = false
			local deltaX = input.Position.X - state.startPos.X
			local deltaY = input.Position.Y - state.startPos.Y
			upBtn.Position = UDim2.new(
				state.startAnchor.X.Scale,
				state.startAnchor.X.Offset + deltaX,
				state.startAnchor.Y.Scale,
				state.startAnchor.Y.Offset + deltaY
			)
		end
	end)

	upBtn.InputEnded:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		upHeld         = false
		state.touch    = nil
		state.dragging = false
	end)
end

-- DRAGGABLE: DOWN BUTTON
do
	local state = {
		touch = nil, startPos = nil, startAnchor = nil,
		totalDist = 0, dragging = false,
	}

	downBtn.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		state.touch       = input
		state.startPos    = Vector2.new(input.Position.X, input.Position.Y)
		state.startAnchor = downBtn.Position
		state.totalDist   = 0
		state.dragging    = false
		if not dragModeOn then
			downHeld = true
		end
	end)

	downBtn.InputChanged:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		local current = Vector2.new(input.Position.X, input.Position.Y)
		state.totalDist = (current - state.startPos).Magnitude
		if dragModeOn and state.totalDist > TAP_THRESHOLD then
			state.dragging = true
			downHeld = false
			local deltaX = input.Position.X - state.startPos.X
			local deltaY = input.Position.Y - state.startPos.Y
			downBtn.Position = UDim2.new(
				state.startAnchor.X.Scale,
				state.startAnchor.X.Offset + deltaX,
				state.startAnchor.Y.Scale,
				state.startAnchor.Y.Offset + deltaY
			)
		end
	end)

	downBtn.InputEnded:Connect(function(input)
		if input ~= state.touch then return end
		if input.UserInputType ~= Enum.UserInputType.Touch then return end
		downHeld       = false
		state.touch    = nil
		state.dragging = false
	end)
end

-- SPEED BUTTONS
speedUp.Activated:Connect(function()
	camSpeed = math.min(camSpeed + SPEED_STEP, MAX_SPEED)
	speedLabel.Text = "Speed: " .. camSpeed
end)

speedDown.Activated:Connect(function()
	camSpeed = math.max(camSpeed - SPEED_STEP, MIN_SPEED)
	speedLabel.Text = "Speed: " .. camSpeed
end)

-- HIT-TEST HELPERS
local function isOnRect(pos, obj)
	local ap = obj.AbsolutePosition
	local as = obj.AbsoluteSize
	return pos.X >= ap.X and pos.X <= ap.X + as.X
		and pos.Y >= ap.Y and pos.Y <= ap.Y + as.Y
end

local function isOnUI(pos)
	local elements = {
		upBtn, downBtn, lockBtn, speedDown, speedUp, speedPanel,
		toggleBtn, topBar,
	}
	for _, obj in ipairs(elements) do
		if isOnRect(pos, obj) then
			return true
		end
	end
	local touchGui = playerGui:FindFirstChild("TouchGui")
	if touchGui then
		local touchFrame = touchGui:FindFirstChild("TouchControlFrame", true)
		if touchFrame then
			local thumbstick = touchFrame:FindFirstChild("DynamicThumbstickFrame", true)
				or touchFrame:FindFirstChild("ThumbstickFrame", true)
			if thumbstick and thumbstick:IsA("GuiObject") and isOnRect(pos, thumbstick) then
				return true
			end
		end
	end
	return false
end

local function isOnToggleBtn(pos)
	return isOnRect(pos, toggleBtn) or isOnRect(pos, topBar)
end

local function isLeftSide(pos)
	local viewSize = camera.ViewportSize
	return pos.X < viewSize.X * 0.4
end

-- GLOBAL TOUCH HANDLING
UserInputService.TouchStarted:Connect(function(input, _gameProcessed)
	if not freecamOn then return end
	local pos = input.Position
	if isOnToggleBtn(pos) then return end
	if isOnUI(pos) then return end
	if isLeftSide(pos) then return end
	if not lookTouch then
		lookTouch   = input
		lastLookPos = pos
	end
end)

UserInputService.TouchMoved:Connect(function(input)
	if not freecamOn then return end
	if input == lookTouch and lastLookPos then
		local delta = input.Position - lastLookPos
		yaw   = yaw   - delta.X * LOOK_SENSITIVITY
		pitch = math.clamp(pitch - delta.Y * LOOK_SENSITIVITY, -89, 89)
		lastLookPos = input.Position
	end
end)

UserInputService.TouchEnded:Connect(function(input)
	if input == lookTouch then
		lookTouch   = nil
		lastLookPos = nil
	end
end)

-- READ MOVE VECTOR
local function getMoveVector()
	if controlsModule then
		local ok, vec = pcall(function()
			return controlsModule:GetMoveVector()
		end)
		if ok and vec then
			return vec
		end
	end
	return Vector3.zero
end

-- FREEZE / UNFREEZE CHARACTER
local function freezeCharacter()
	local char = player.Character
	if not char then return end
	local humanoid = char:FindFirstChildWhichIsA("Humanoid")
	local rootPart = char:FindFirstChild("HumanoidRootPart")
	if humanoid then
		origWalkSpeed  = humanoid.WalkSpeed
		origJumpPower  = humanoid.JumpPower
		origJumpHeight = humanoid.JumpHeight
		humanoid.WalkSpeed  = 0
		humanoid.JumpPower  = 0
		humanoid.JumpHeight = 0
		humanoid:Move(Vector3.zero, false)
	end
	if rootPart then
		frozenPosition = rootPart.CFrame
		rootPart.AssemblyLinearVelocity  = Vector3.zero
		rootPart.AssemblyAngularVelocity = Vector3.zero
		rootPart.Anchored = true
		rootPart.CFrame = frozenPosition
	end
	if controlsModule then
		pcall(function() controlsModule:Disable() end)
		task.defer(function()
			if controlsModule then
				pcall(function() controlsModule:Enable() end)
			end
		end)
	end
end

local function unfreezeCharacter()
	local char = player.Character
	if not char then return end
	local humanoid = char:FindFirstChildWhichIsA("Humanoid")
	local rootPart = char:FindFirstChild("HumanoidRootPart")
	if rootPart then
		rootPart.Anchored = false
		rootPart.AssemblyLinearVelocity  = Vector3.zero
		rootPart.AssemblyAngularVelocity = Vector3.zero
	end
	if humanoid then
		humanoid.WalkSpeed  = origWalkSpeed  or 16
		humanoid.JumpPower  = origJumpPower  or 50
		humanoid.JumpHeight = origJumpHeight or 7.2
		humanoid:Move(Vector3.zero, false)
	end
	frozenPosition = nil
end

-- ENABLE / DISABLE FREECAM
enableFreecam = function()
	freecamOn = true
	origCamType    = camera.CameraType
	origCamSubject = camera.CameraSubject
	camera.CameraType = Enum.CameraType.Scriptable
	freezeCharacter()
	local cf = camera.CFrame
	camPos   = cf.Position
	local lv = cf.LookVector
	yaw      = math.deg(math.atan2(-lv.X, -lv.Z))
	pitch    = math.deg(math.asin(math.clamp(lv.Y, -1, 1)))
	dragModeOn = false
	updateLockVisuals()
	controls.Visible = true
	lockBtn.Visible  = true
	toggleBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
	toggleBtn.Text             = "✖  Exit Cam"
	task.spawn(function()
		local touchGui = playerGui:FindFirstChild("TouchGui")
		if touchGui then touchGui.Enabled = true end
	end)
end

disableFreecam = function()
	freecamOn = false
	camera.CameraType    = origCamType    or Enum.CameraType.Custom
	camera.CameraSubject = origCamSubject
		or (player.Character and player.Character:FindFirstChildWhichIsA("Humanoid"))
	unfreezeCharacter()
	controls.Visible = false
	lockBtn.Visible  = false
	toggleBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	toggleBtn.Text             = "📷 Freecam"
	lookTouch   = nil
	lastLookPos = nil
	upHeld      = false
	downHeld    = false
	dragModeOn  = false
	task.spawn(function()
		local touchGui = playerGui:FindFirstChild("TouchGui")
		if touchGui then touchGui.Enabled = true end
	end)
end

-- CAMERA UPDATE
RunService.RenderStepped:Connect(function(dt)
	if not freecamOn then return end
	local char = player.Character
	if char then
		local humanoid = char:FindFirstChildWhichIsA("Humanoid")
		local rootPart = char:FindFirstChild("HumanoidRootPart")
		if humanoid then
			humanoid.WalkSpeed  = 0
			humanoid.JumpPower  = 0
			humanoid.JumpHeight = 0
		end
		if rootPart then
			if not rootPart.Anchored then
				rootPart.Anchored = true
			end
			if frozenPosition then
				rootPart.AssemblyLinearVelocity  = Vector3.zero
				rootPart.AssemblyAngularVelocity = Vector3.zero
				rootPart.CFrame = frozenPosition
			end
		end
	end
	local vertInput = 0
	if upHeld   then vertInput = vertInput + 1 end
	if downHeld then vertInput = vertInput - 1 end
	local moveVector = getMoveVector()
	local rotation = CFrame.Angles(0, math.rad(yaw), 0)
		* CFrame.Angles(math.rad(pitch), 0, 0)
	local right = rotation.RightVector
	local fwd   = rotation.LookVector
	local move = (right * moveVector.X)
		+ (fwd * -moveVector.Z)
		+ (Vector3.yAxis * vertInput)
	camPos = camPos + move * camSpeed * dt
	camera.CameraType = Enum.CameraType.Scriptable
	camera.CFrame     = CFrame.new(camPos) * rotation
end)

-- HANDLE RESPAWN
player.CharacterAdded:Connect(function()
	if freecamOn then
		task.wait(0.5)
		camera.CameraType = Enum.CameraType.Scriptable
		freezeCharacter()
	end
end)
