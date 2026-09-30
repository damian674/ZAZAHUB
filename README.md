-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Camera = workspace.CurrentCamera
local localPlayer = Players.LocalPlayer
local playerGui = localPlayer:WaitForChild("PlayerGui")
local mouse = localPlayer:GetMouse()

--------------------------------------------------------------------------------
-- CONFIGURATION & SETTINGS
--------------------------------------------------------------------------------
local centerOffsetX = 0 
local centerOffsetY = 0 

local aimbotEnabled = false
local aimAssistEnabled = false
local teamCheckEnabled = false
local wallCheckEnabled = true
local noclipEnabled = false
local fullbrightEnabled = false

local freecamEnabled = false

local deleteWidgetEnabled = false
local deleteToolActive = false
local returnToolActive = false

local aimAssistSmoothness = 5
local fovRadius = 150
local aimAssistFovRadius = 150
local targetPartName = "Head"
local baseWindowWidth = 480
local customCameraFov = Camera.FieldOfView

local screenStretchFactor = 1.0
local fovColorIndex = 1
local fovColors = {
	{Name = "White", Color = Color3.fromRGB(255, 255, 255)},
	{Name = "Indigo", Color = Color3.fromRGB(99, 102, 241)},
	{Name = "Red", Color = Color3.fromRGB(239, 68, 68)},
	{Name = "Green", Color = Color3.fromRGB(34, 197, 94)},
	{Name = "Cyan", Color = Color3.fromRGB(6, 182, 212)}
}
local toggleKey = Enum.KeyCode.Insert

local activeConnections = {}
local deletedPartsHistory = {}

-- Config System (Save/Load)
local CONFIG_FILE = "ZazaHubConfig.json"
local function saveConfig()
	local data = {
		Aimbot = aimbotEnabled, AimAssist = aimAssistEnabled, TeamCheck = teamCheckEnabled,
		WallCheck = wallCheckEnabled, Stretch = screenStretchFactor, FOVRadius = fovRadius,
		AAFOV = aimAssistFovRadius, AASpeed = aimAssistSmoothness, TargetPart = targetPartName,
		CameraFOV = customCameraFov
	}
	pcall(function() writefile(CONFIG_FILE, HttpService:JSONEncode(data)) end)
end

local function loadConfig()
	if pcall(function() readfile(CONFIG_FILE) end) then
		local success, decoded = pcall(function() return HttpService:JSONDecode(readfile(CONFIG_FILE)) end)
		if success and decoded then
			aimbotEnabled = decoded.Aimbot or false
			aimAssistEnabled = decoded.AimAssist or false
			teamCheckEnabled = decoded.TeamCheck or false
			wallCheckEnabled = decoded.WallCheck ~= nil and decoded.WallCheck or true
			screenStretchFactor = decoded.Stretch or 1.0
			fovRadius = decoded.FOVRadius or 150
			aimAssistFovRadius = decoded.AAFOV or 150
			aimAssistSmoothness = decoded.AASpeed or 5
			targetPartName = decoded.TargetPart or "Head"
			customCameraFov = decoded.CameraFOV or Camera.FieldOfView
		end
	end
end
loadConfig()

-- UI Creation
local zazaHubGui = Instance.new("ScreenGui")
zazaHubGui.Name = "ZazaHubGui"
zazaHubGui.ResetOnSpawn = false
zazaHubGui.Parent = playerGui

local toggleOpenBtn = Instance.new("TextButton")
toggleOpenBtn.Name = "ToggleOpenButton"
toggleOpenBtn.Size = UDim2.new(0, 110, 0, 38)
toggleOpenBtn.Position = UDim2.new(0, 20, 0, 20)
toggleOpenBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
toggleOpenBtn.TextColor3 = Color3.fromRGB(240, 240, 245)
toggleOpenBtn.TextSize = 12
toggleOpenBtn.Font = Enum.Font.GothamBold
toggleOpenBtn.Text = "⚡ ZAZA HUB"
toggleOpenBtn.Parent = zazaHubGui

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(1, 0)
toggleCorner.Parent = toggleOpenBtn

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = Color3.fromRGB(50, 50, 65)
toggleStroke.Thickness = 1.5
toggleStroke.Parent = toggleOpenBtn

local watermark = Instance.new("TextLabel")
watermark.Name = "StatusWatermark"
watermark.Size = UDim2.new(0, 200, 0, 28)
watermark.Position = UDim2.new(1, -215, 0, 15)
watermark.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
watermark.TextColor3 = Color3.fromRGB(200, 200, 220)
watermark.TextSize = 11
watermark.Font = Enum.Font.GothamMedium
watermark.Text = "Zaza Hub | FPS: 60 | Ping: 0ms"
watermark.Parent = zazaHubGui

local wmCorner = Instance.new("UICorner")
wmCorner.CornerRadius = UDim.new(0, 6)
wmCorner.Parent = watermark

local wmStroke = Instance.new("UIStroke")
wmStroke.Color = Color3.fromRGB(45, 45, 60)
wmStroke.Thickness = 1
wmStroke.Parent = watermark

local mainWindow = Instance.new("Frame")
mainWindow.Name = "ZazaMainWindow"
mainWindow.Size = UDim2.new(0, baseWindowWidth, 0, 420)
mainWindow.Position = UDim2.new(0.5, -baseWindowWidth/2, 0.5, -210)
mainWindow.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
mainWindow.BorderSizePixel = 0
mainWindow.Visible = true
mainWindow.Parent = zazaHubGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 12)
mainCorner.Parent = mainWindow

local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(45, 45, 60)
mainStroke.Thickness = 1.5
mainStroke.Parent = mainWindow

-- Dragging Logic
local toggleDragging = false
local toggleDragStart, toggleStartPos
local hasMoved = false

toggleOpenBtn.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		toggleDragging = true
		toggleDragStart = input.Position
		toggleStartPos = toggleOpenBtn.Position
		hasMoved = false
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then toggleDragging = false end
		end)
	end
end)

toggleOpenBtn.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		if toggleDragging then
			local delta = input.Position - toggleDragStart
			if delta.Magnitude > 5 then hasMoved = true end
			toggleOpenBtn.Position = UDim2.new(toggleStartPos.X.Scale, toggleStartPos.X.Offset + delta.X, toggleStartPos.Y.Scale, toggleStartPos.Y.Offset + delta.Y)
		end
	end
end)

toggleOpenBtn.MouseButton1Click:Connect(function()
	if not hasMoved then mainWindow.Visible = not mainWindow.Visible end
end)

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, 50)
header.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
header.BorderSizePixel = 0
header.Parent = mainWindow

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 12)
headerCorner.Parent = header

local fixHeader = Instance.new("Frame")
fixHeader.Size = UDim2.new(1, 0, 0, 12)
fixHeader.Position = UDim2.new(0, 0, 1, -12)
fixHeader.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
fixHeader.BorderSizePixel = 0
fixHeader.Parent = header

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, -30, 1, 0)
titleLabel.Position = UDim2.new(0, 18, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
titleLabel.TextSize = 15
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Text = "ZAZA HUB // MOBILE & PC"
titleLabel.Parent = header

local dragging = false
local dragInput, dragStart, startPos

header.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = mainWindow.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then dragging = false end
		end)
	end
end)

header.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

table.insert(activeConnections, UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		local delta = input.Position - dragStart
		mainWindow.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
	end
end))

-- Tabs Setup
local tabBar = Instance.new("Frame")
tabBar.Name = "TabBar"
tabBar.Size = UDim2.new(1, -24, 0, 36)
tabBar.Position = UDim2.new(0, 12, 0, 60)
tabBar.BackgroundTransparency = 1
tabBar.Parent = mainWindow

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.SortOrder = Enum.SortOrder.LayoutOrder
tabLayout.Padding = UDim.new(0, 6)
tabLayout.Parent = tabBar

local contentContainer = Instance.new("Frame")
contentContainer.Name = "ContentContainer"
contentContainer.Size = UDim2.new(1, -24, 1, -114)
contentContainer.Position = UDim2.new(0, 12, 0, 104)
contentContainer.BackgroundTransparency = 1
contentContainer.Parent = mainWindow

local tabs = {}
local activeTabName = nil

local function createTab(name, layoutOrder)
	local tabButton = Instance.new("TextButton")
	tabButton.Name = name .. "TabButton"
	tabButton.Size = UDim2.new(0, 72, 1, 0)
	tabButton.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
	tabButton.TextColor3 = Color3.fromRGB(140, 140, 160)
	tabButton.TextSize = 11
	tabButton.Font = Enum.Font.GothamMedium
	tabButton.Text = name
	tabButton.LayoutOrder = layoutOrder
	tabButton.Parent = tabBar

	local btnCorner = Instance.new("UICorner")
	btnCorner.CornerRadius = UDim.new(0, 6)
	btnCorner.Parent = tabButton

	local page = Instance.new("ScrollingFrame")
	page.Name = name .. "Page"
	page.Size = UDim2.new(1, 0, 1, 0)
	page.BackgroundTransparency = 1
	page.BorderSizePixel = 0
	page.Visible = false
	page.ScrollBarThickness = 3
	page.ScrollBarImageColor3 = Color3.fromRGB(70, 70, 90)
	page.Parent = contentContainer

	local pageLayout = Instance.new("UIListLayout")
	pageLayout.SortOrder = Enum.SortOrder.LayoutOrder
	pageLayout.Padding = UDim.new(0, 8)
	pageLayout.Parent = page

	tabButton.MouseButton1Click:Connect(function()
		for _, t in pairs(tabs) do
			t.Page.Visible = false
			t.Button.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
			t.Button.TextColor3 = Color3.fromRGB(140, 140, 160)
		end
		page.Visible = true
		tabButton.BackgroundColor3 = Color3.fromRGB(99, 102, 241)
		tabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
		activeTabName = name
	end)

	tabs[name] = {Button = tabButton, Page = page}

	if not activeTabName then
		page.Visible = true
		tabButton.BackgroundColor3 = Color3.fromRGB(99, 102, 241)
		tabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
		activeTabName = name
	end

	return page
end

local combatPage = createTab("Combat", 1)
local visualsPage = createTab("Visuals", 2)
local farmPage = createTab("Farm", 3)
local settingsPage = createTab("Settings", 4)
local otherPage = createTab("Other", 5)

-- Components Helpers
local function createToggle(parent, defaultText, initialState, callback)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, 0, 0, 38)
	btn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
	btn.Parent = parent

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 6)
	corner.Parent = btn

	local state = initialState or false
	local function updateStyle()
		if state then
			btn.BackgroundColor3 = Color3.fromRGB(24, 40, 30)
			btn.Text = "  " .. defaultText .. " [ON]"
			btn.TextColor3 = Color3.fromRGB(34, 197, 94)
		else
			btn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
			btn.Text = "  " .. defaultText .. " [OFF]"
			btn.TextColor3 = Color3.fromRGB(160, 160, 180)
		end
	end
	
	updateStyle()
	btn.TextSize = 13
	btn.Font = Enum.Font.GothamBold
	btn.TextXAlignment = Enum.TextXAlignment.Left

	btn.MouseButton1Click:Connect(function()
		state = not state
		updateStyle()
		callback(state)
		saveConfig()
	end)
	return btn
end

local function createSlider(parent, title, minVal, maxVal, defaultVal, callback)
	local container = Instance.new("Frame")
	container.Size = UDim2.new(1, 0, 0, 52)
	container.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
	container.BorderSizePixel = 0
	container.Parent = parent

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 6)
	corner.Parent = container

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(1, -24, 0, 22)
	label.Position = UDim2.new(0, 12, 0, 6)
	label.BackgroundTransparency = 1
	label.TextColor3 = Color3.fromRGB(200, 200, 220)
	label.TextSize = 12
	label.Font = Enum.Font.GothamBold
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Text = title .. ": " .. tostring(defaultVal)
	label.Parent = container

	local sliderBar = Instance.new("Frame")
	sliderBar.Size = UDim2.new(1, -24, 0, 6)
	sliderBar.Position = UDim2.new(0, 12, 0, 34)
	sliderBar.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
	sliderBar.BorderSizePixel = 0
	sliderBar.Parent = container

	local barCorner = Instance.new("UICorner")
	barCorner.CornerRadius = UDim.new(1, 0)
	barCorner.Parent = sliderBar

	local fill = Instance.new("Frame")
	fill.Size = UDim2.new((defaultVal - minVal) / (maxVal - minVal), 0, 1, 0)
	fill.BackgroundColor3 = Color3.fromRGB(99, 102, 241)
	fill.BorderSizePixel = 0
	fill.Parent = sliderBar

	local fillCorner = Instance.new("UICorner")
	fillCorner.CornerRadius = UDim.new(1, 0)
	fillCorner.Parent = fill

	local sliding = false
	local function updateValue(input)
		local pos = math.clamp((input.Position.X - sliderBar.AbsolutePosition.X) / sliderBar.AbsoluteSize.X, 0, 1)
		local val = math.floor(minVal + ((maxVal - minVal) * pos))
		fill.Size = UDim2.new(pos, 0, 1, 0)
		label.Text = title .. ": " .. tostring(val)
		callback(val)
	end

	sliderBar.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = true
			updateValue(input)
		end
	end)

	table.insert(activeConnections, UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = false
			saveConfig()
		end
	end))

	table.insert(activeConnections, UserInputService.InputChanged:Connect(function(input)
		if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			updateValue(input)
		end
	end))

	return container
end

-- FOV Circle
local fovCircle = Instance.new("Frame")
fovCircle.Name = "FOVCircle"
fovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
fovCircle.BackgroundTransparency = 1
fovCircle.Visible = false
fovCircle.Parent = zazaHubGui

local fovCorner = Instance.new("UICorner")
fovCorner.CornerRadius = UDim.new(1, 0)
fovCorner.Parent = fovCircle

local fovStroke = Instance.new("UIStroke")
fovStroke.Color = fovColors[1].Color
fovStroke.Thickness = 1.5
fovStroke.Transparency = 0.4
fovStroke.Parent = fovCircle

-- Combat Calculations
local function isTeamMatch(player)
	if not teamCheckEnabled then return false end
	if player.Team and localPlayer.Team then return player.Team == localPlayer.Team end
	return false
end

local function hasLineOfSight(targetPart, targetCharacter)
	if not wallCheckEnabled then return true end
	local origin = Camera.CFrame.Position
	local direction = targetPart.Position - origin
	local raycastParams = RaycastParams.new()
	raycastParams.FilterType = Enum.RaycastFilterType.Exclude
	raycastParams.FilterDescendantsInstances = {localPlayer.Character}
	raycastParams.IgnoreWater = true
	local result = workspace:Raycast(origin, direction, raycastParams)
	if result then return result.Instance:IsDescendantOf(targetCharacter) end
	return true
end

local function getClosestPlayerInFOV(radius)
	local closestPlayer = nil
	local shortestDistance = radius
	local screenCenter = Vector2.new((Camera.ViewportSize.X / 2) + centerOffsetX, (Camera.ViewportSize.Y / 2) + centerOffsetY)
	
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and not isTeamMatch(player) and player.Character then
			local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
			local targetPart = player.Character:FindFirstChild(targetPartName) or player.Character:FindFirstChild("Head")
			
			if humanoid and humanoid.Health > 0 and targetPart then
				local calcPos = targetPart.Position
				local screenPoint, onScreen = Camera:WorldToViewportPoint(calcPos)
				if onScreen then
					local pos2D = Vector2.new(screenPoint.X, screenPoint.Y)
					local distanceFromCenter = (pos2D - screenCenter).Magnitude
					if distanceFromCenter <= shortestDistance then
						if hasLineOfSight(targetPart, player.Character) then
							shortestDistance = distanceFromCenter
							closestPlayer = player
						end
					end
				end
			end
		end
	end
	return closestPlayer
end

-- UNIFIED SINGLE RENDER STEP LOOP (Optimized to prevent frame drops and crashes)
local freecamCF = Camera.CFrame
local freecamSpeed = 1.0
local activeVisuals = {}
local espTracesEnabled, boxEspEnabled, healthEspEnabled, distanceEspEnabled, espInventoryEnabled, gunEspEnabled = false, false, false, false, false, false

table.insert(activeConnections, RunService.RenderStepped:Connect(function(dt)
	local centerX = (Camera.ViewportSize.X / 2) + centerOffsetX
	local centerY = (Camera.ViewportSize.Y / 2) + centerOffsetY
	
	-- Watermark
	local fps = math.floor(1 / dt)
	local ping = 0
	pcall(function() ping = math.floor(localPlayer:GetNetworkPing() * 1000) end)
	watermark.Text = string.format("Zaza Hub | FPS: %d | Ping: %dms", fps, ping)
	
	-- Screen Stretch
	if screenStretchFactor ~= 1.0 then
		Camera.CFrame = Camera.CFrame * CFrame.new(0, 0, 0, 1, 0, 0, 0, screenStretchFactor, 0, 0, 0, 1)
	end
	
	-- Freecam
	if freecamEnabled then
		Camera.CameraType = Enum.CameraType.Scriptable
		local moveDir = Vector3.new()
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + Vector3.new(0, 0, -1) end
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir + Vector3.new(0, 0, 1) end
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir + Vector3.new(-1, 0, 0) end
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + Vector3.new(1, 0, 0) end
		freecamCF = freecamCF + (Camera.CFrame:VectorToWorldSpace(moveDir) * (freecamSpeed * 50 * dt))
		Camera.CFrame = freecamCF
	else
		if Camera.CameraType == Enum.CameraType.Scriptable then
			Camera.CameraType = Enum.CameraType.Custom
		end
		freecamCF = Camera.CFrame
	end

	-- FOV Circle Drawing & Aimbot
	if aimbotEnabled then
		fovCircle.Visible = true
		fovCircle.Position = UDim2.new(0, centerX, 0, centerY)
		fovCircle.Size = UDim2.new(0, fovRadius * 2, 0, fovRadius * 2)
		local target = getClosestPlayerInFOV(fovRadius)
		if target and target.Character then
			local aimPart = target.Character:FindFirstChild(targetPartName) or target.Character:FindFirstChild("Head")
			if aimPart then
				Camera.CFrame = CFrame.new(Camera.CFrame.Position, aimPart.Position)
			end
		end
	else
		fovCircle.Visible = false
	end
	
	-- Aim Assist
	if aimAssistEnabled then
		local target = getClosestPlayerInFOV(aimAssistFovRadius)
		if target and target.Character then
			local aimPart = target.Character:FindFirstChild(targetPartName) or target.Character:FindFirstChild("Head")
			if aimPart then
				local goalCFrame = CFrame.new(Camera.CFrame.Position, aimPart.Position)
				Camera.CFrame = Camera.CFrame:Lerp(goalCFrame, math.clamp(dt * aimAssistSmoothness * 3, 0, 1))
			end
		end
	end
	
	-- Noclip
	if noclipEnabled and localPlayer.Character then
		for _, part in ipairs(localPlayer.Character:GetDescendants()) do
			if part:IsA("BasePart") then part.CanCollide = false end
		end
	end
	
	-- Fullbright
	if fullbrightEnabled then
		Lighting.Brightness = 2
		Lighting.ClockTime = 14
		Lighting.GlobalShadows = false
		Lighting.OutdoorAmbient = Color3.fromRGB(128, 128, 128)
	end

	-- Unified Optimized ESP Render Loop
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and player.Character then
			local char = player.Character
			local hrp = char:FindFirstChild("HumanoidRootPart")
			local humanoid = char:FindFirstChildOfClass("Humanoid")
			local shouldShow = hrp and humanoid and humanoid.Health > 0 and not isTeamMatch(player)
			
			if shouldShow then
				local visData = activeVisuals[player]
				if not visData then
					visData = {
						Tracer = Drawing and Drawing.new("Line") or nil,
						Box = Instance.new("Frame"),
						BoxStroke = Instance.new("UIStroke"),
						InfoLabel = Instance.new("TextLabel"),
						InvLabel = Instance.new("TextLabel"),
						GunLabel = Instance.new("TextLabel")
					}
					
					if visData.Tracer then
						visData.Tracer.Visible = false
						visData.Tracer.Color = Color3.fromRGB(255, 255, 255)
						visData.Tracer.Thickness = 1
						visData.Tracer.Transparency = 0.7
					end
					
					visData.Box.BackgroundTransparency = 1
					visData.Box.Visible = false
					visData.Box.Parent = zazaHubGui
					visData.BoxStroke.Color = Color3.fromRGB(255, 255, 255)
					visData.BoxStroke.Thickness = 1.5
					visData.BoxStroke.Parent = visData.Box
					
					visData.InfoLabel.BackgroundTransparency = 1
					visData.InfoLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
					visData.InfoLabel.TextSize = 11
					visData.InfoLabel.Font = Enum.Font.GothamBold
					visData.InfoLabel.TextStrokeTransparency = 0.5
					visData.InfoLabel.Visible = false
					visData.InfoLabel.Parent = zazaHubGui
					
					visData.InvLabel.BackgroundTransparency = 1
					visData.InvLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
					visData.InvLabel.TextSize = 11
					visData.InvLabel.Font = Enum.Font.GothamBold
					visData.InvLabel.TextStrokeTransparency = 0.5
					visData.InvLabel.Visible = false
					visData.InvLabel.Parent = zazaHubGui

					visData.GunLabel.BackgroundTransparency = 1
					visData.GunLabel.TextColor3 = Color3.fromRGB(239, 68, 68)
					visData.GunLabel.TextSize = 11
					visData.GunLabel.Font = Enum.Font.GothamBold
					visData.GunLabel.TextStrokeTransparency = 0.5
					visData.GunLabel.Visible = false
					visData.GunLabel.Parent = zazaHubGui
					
					activeVisuals[player] = visData
				end
				
				local head = char:FindFirstChild("Head")
				local rootPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
				
				if onScreen then
					-- Tracers
					if espTracesEnabled and visData.Tracer then
						visData.Tracer.Visible = true
						visData.Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
						visData.Tracer.To = Vector2.new(rootPos.X, rootPos.Y)
					elseif visData.Tracer then
						visData.Tracer.Visible = false
					end
					
					-- Box
					if boxEspEnabled and head then
						local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
						local legPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))
						local height = math.abs(headPos.Y - legPos.Y)
						local width = height / 2
						visData.Box.Visible = true
						visData.Box.Position = UDim2.new(0, rootPos.X - width / 2, 0, headPos.Y)
						visData.Box.Size = UDim2.new(0, width, 0, height)
					else
						visData.Box.Visible = false
					end
					
					-- Info (Health & Distance)
					local infoTexts = {}
					if healthEspEnabled then
						table.insert(infoTexts, string.format("HP: %d/%d", math.floor(humanoid.Health), math.floor(humanoid.MaxHealth)))
					end
					if distanceEspEnabled and localPlayer.Character and localPlayer.Character:FindFirstChild("HumanoidRootPart") then
						local dist = (localPlayer.Character.HumanoidRootPart.Position - hrp.Position).Magnitude
						table.insert(infoTexts, string.format("[%dft]", math.floor(dist)))
					end
					
					if #infoTexts > 0 then
						visData.InfoLabel.Visible = true
						visData.InfoLabel.Position = UDim2.new(0, rootPos.X - 100, 0, rootPos.Y - 35)
						visData.InfoLabel.Size = UDim2.new(0, 200, 0, 20)
						visData.InfoLabel.Text = table.concat(infoTexts, " ")
					else
						visData.InfoLabel.Visible = false
					end
					
					-- Inventory
					if espInventoryEnabled then
						local items = {}
						if player:FindFirstChild("Backpack") then
							for _, item in ipairs(player.Backpack:GetChildren()) do table.insert(items, item.Name) end
						end
						for _, item in ipairs(char:GetChildren()) do
							if item:IsA("Tool") then table.insert(items, item.Name) end
						end
						visData.InvLabel.Visible = true
						visData.InvLabel.Position = UDim2.new(0, rootPos.X - 100, 0, rootPos.Y + 25)
						visData.InvLabel.Size = UDim2.new(0, 200, 0, 40)
						visData.InvLabel.Text = (#items > 0) and ("Inv: " .. table.concat(items, ", ")) or "Inv: Empty"
					else
						visData.InvLabel.Visible = false
					end

					-- Gun ESP
					if gunEspEnabled then
						local activeTool = nil
						for _, item in ipairs(char:GetChildren()) do
							if item:IsA("Tool") then activeTool = item.Name break end
						end
						if activeTool then
							visData.GunLabel.Visible = true
							visData.GunLabel.Position = UDim2.new(0, rootPos.X - 100, 0, rootPos.Y + 65)
							visData.GunLabel.Size = UDim2.new(0, 200, 0, 20)
							visData.GunLabel.Text = "Gun: " .. activeTool
						else
							visData.GunLabel.Visible = false
						end
					else
						visData.GunLabel.Visible = false
					end
				else
					if visData.Tracer then visData.Tracer.Visible = false end
					visData.Box.Visible = false
					visData.InfoLabel.Visible = false
					visData.InvLabel.Visible = false
					visData.GunLabel.Visible = false
				end
			else
				if activeVisuals[player] then
					if activeVisuals[player].Tracer then activeVisuals[player].Tracer:Remove() end
					activeVisuals[player].Box:Destroy()
					activeVisuals[player].InfoLabel:Destroy()
					activeVisuals[player].InvLabel:Destroy()
					activeVisuals[player].GunLabel:Destroy()
					activeVisuals[player] = nil
				end
			end
		end
	end
end))

--------------------------------------------------------------------------------
-- POPULATE TABS
--------------------------------------------------------------------------------
createToggle(combatPage, "Aimbot", aimbotEnabled, function(state)
	aimbotEnabled = state
	if fovSliderContainer then fovSliderContainer.Visible = state end
end)

local fovSliderContainer = createSlider(combatPage, "FOV Scale", 1, 300, fovRadius, function(value) fovRadius = value end)
fovSliderContainer.Visible = aimbotEnabled

createToggle(combatPage, "Aim Assist", aimAssistEnabled, function(state) aimAssistEnabled = state end)
createSlider(combatPage, "Aim Assist Speed", 1, 20, aimAssistSmoothness, function(value) aimAssistSmoothness = value end)
createToggle(combatPage, "Team Check", teamCheckEnabled, function(state) teamCheckEnabled = state end)
createToggle(combatPage, "Wall Check (LoS)", wallCheckEnabled, function(state) wallCheckEnabled = state end)

local targetPartBtn = Instance.new("TextButton")
targetPartBtn.Size = UDim2.new(1, 0, 0, 38)
targetPartBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
targetPartBtn.TextColor3 = Color3.fromRGB(200, 200, 220)
targetPartBtn.TextSize = 13
targetPartBtn.Font = Enum.Font.GothamBold
targetPartBtn.Text = "  Target Part: " .. targetPartName
targetPartBtn.TextXAlignment = Enum.TextXAlignment.Left
targetPartBtn.Parent = combatPage

local tpCorner = Instance.new("UICorner")
tpCorner.CornerRadius = UDim.new(0, 6)
tpCorner.Parent = targetPartBtn

targetPartBtn.MouseButton1Click:Connect(function()
	if targetPartName == "Head" then
		targetPartName = "HumanoidRootPart"
		targetPartBtn.Text = "  Target Part: Torso"
	else
		targetPartName = "Head"
		targetPartBtn.Text = "  Target Part: Head"
	end
	saveConfig()
end)

-- Visuals Page Toggles
createToggle(visualsPage, "ESP Traces", false, function(state) espTracesEnabled = state end)
createToggle(visualsPage, "Box ESP", false, function(state) boxEspEnabled = state end)
createToggle(visualsPage, "Health ESP", false, function(state) healthEspEnabled = state end)
createToggle(visualsPage, "Distance ESP", false, function(state) distanceEspEnabled = state end)
createToggle(visualsPage, "ESP Inventory", false, function(state) espInventoryEnabled = state end)
createToggle(visualsPage, "Gun ESP", false, function(state) gunEspEnabled = state end)
createToggle(visualsPage, "Fullbright", false, function(state)
	fullbrightEnabled = state
	if not state then
		Lighting.Brightness = 1
		Lighting.GlobalShadows = true
	end
end)

-- Farm Tab
local loadScriptBtn = Instance.new("TextButton")
loadScriptBtn.Size = UDim2.new(1, 0, 0, 42)
loadScriptBtn.BackgroundColor3 = Color3.fromRGB(99, 102, 241)
loadScriptBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
loadScriptBtn.TextSize = 13
loadScriptBtn.Font = Enum.Font.GothamBold
loadScriptBtn.Text = "Run Farm Script"
loadScriptBtn.Parent = farmPage

local loaderCorner = Instance.new("UICorner")
loaderCorner.CornerRadius = UDim.new(0, 6)
loaderCorner.Parent = loadScriptBtn

loadScriptBtn.MouseButton1Click:Connect(function()
	loadScriptBtn.Text = "Running..."
	local success = pcall(function()
		loadstring(game:HttpGet("https://raw.githubusercontent.com/rexxymayor-ai/SCRIPTtt/refs/heads/main/script%20automs", true))()
	end)
	if success then
		loadScriptBtn.Text = "Successfully Executed!"
		loadScriptBtn.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
	else
		loadScriptBtn.Text = "Execution Failed"
		loadScriptBtn.BackgroundColor3 = Color3.fromRGB(239, 68, 68)
	end
	task.delay(2, function()
		loadScriptBtn.Text = "Run Farm Script"
		loadScriptBtn.BackgroundColor3 = Color3.fromRGB(99, 102, 241)
	end)
end)

-- Settings Tab
createSlider(settingsPage, "Stretch Resolution", 50, 100, math.floor(screenStretchFactor * 100), function(value)
	screenStretchFactor = value / 100
end)

createSlider(settingsPage, "Camera FOV", 50, 120, math.floor(customCameraFov), function(value)
	customCameraFov = value
	Camera.FieldOfView = value
end)

local fovColorBtn = Instance.new("TextButton")
fovColorBtn.Size = UDim2.new(1, 0, 0, 38)
fovColorBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
fovColorBtn.TextColor3 = Color3.fromRGB(200, 200, 220)
fovColorBtn.TextSize = 13
fovColorBtn.Font = Enum.Font.GothamBold
fovColorBtn.Text = "  FOV Circle Color: " .. fovColors[fovColorIndex].Name
fovColorBtn.TextXAlignment = Enum.TextXAlignment.Left
fovColorBtn.Parent = settingsPage

local fcCorner = Instance.new("UICorner")
fcCorner.CornerRadius = UDim.new(0, 6)
fcCorner.Parent = fovColorBtn

fovColorBtn.MouseButton1Click:Connect(function()
	fovColorIndex = fovColorIndex % #fovColors + 1
	fovColorBtn.Text = "  FOV Circle Color: " .. fovColors[fovColorIndex].Name
	fovStroke.Color = fovColors[fovColorIndex].Color
end)

-- Other Tab
createToggle(otherPage, "Noclip", false, function(state) noclipEnabled = state end)
createToggle(otherPage, "Freecam", false, function(state) freecamEnabled = state end)

local serverHopBtn = Instance.new("TextButton")
serverHopBtn.Size = UDim2.new(1, 0, 0, 38)
serverHopBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
serverHopBtn.TextColor3 = Color3.fromRGB(200, 200, 220)
serverHopBtn.TextSize = 13
serverHopBtn.Font = Enum.Font.GothamBold
serverHopBtn.Text = "  Server Hop (Find New Lobby)"
serverHopBtn.TextXAlignment = Enum.TextXAlignment.Left
serverHopBtn.Parent = otherPage

local shCorner = Instance.new("UICorner")
shCorner.CornerRadius = UDim.new(0, 6)
shCorner.Parent = serverHopBtn

serverHopBtn.MouseButton1Click:Connect(function()
	serverHopBtn.Text = "  Hopping Servers..."
	pcall(function()
		local servers = {}
		local req = game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100")
		local body = HttpService:JSONDecode(req)
		if body and body.data then
			for _, s in ipairs(body.data) do
				if type(s) == "table" and s.maxPlayers and s.playing and s.playing < s.maxPlayers and s.id ~= game.JobId then
					table.insert(servers, s.id)
				end
			end
		end
		if #servers > 0 then
			TeleportService:TeleportToPlaceInstance(game.PlaceId, servers[math.random(1, #servers)], localPlayer)
		else
			TeleportService:Teleport(game.PlaceId, localPlayer)
		end
	end)
end)

local externalWidget = Instance.new("Frame")
externalWidget.Name = "ExternalDeleteWidget"
externalWidget.Size = UDim2.new(0, 160, 0, 100)
externalWidget.Position = UDim2.new(0, 20, 0, 160)
externalWidget.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
externalWidget.BorderSizePixel = 0
externalWidget.Visible = false
externalWidget.Parent = zazaHubGui

local widgetCorner = Instance.new("UICorner")
widgetCorner.CornerRadius = UDim.new(0, 8)
widgetCorner.Parent = externalWidget

local widgetStroke = Instance.new("UIStroke")
widgetStroke.Color = Color3.fromRGB(45, 45, 60)
widgetStroke.Thickness = 1.5
widgetStroke.Parent = externalWidget

local widgetHeader = Instance.new("TextButton")
widgetHeader.Size = UDim2.new(1, 0, 0, 30)
widgetHeader.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
widgetHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
widgetHeader.TextSize = 12
widgetHeader.Font = Enum.Font.GothamBold
widgetHeader.Text = "Delete Tool Menu"
widgetHeader.Parent = externalWidget

local extDeleteBtn = Instance.new("TextButton")
extDeleteBtn.Size = UDim2.new(1, -12, 0, 28)
extDeleteBtn.Position = UDim2.new(0, 6, 0, 34)
extDeleteBtn.BackgroundColor3 = Color3.fromRGB(40, 24, 24)
extDeleteBtn.TextColor3 = Color3.fromRGB(239, 68, 68)
extDeleteBtn.TextSize = 11
extDeleteBtn.Font = Enum.Font.GothamBold
extDeleteBtn.Text = "Delete: OFF"
extDeleteBtn.Parent = externalWidget

local edCorner = Instance.new("UICorner")
edCorner.CornerRadius = UDim.new(0, 6)
edCorner.Parent = extDeleteBtn

extDeleteBtn.MouseButton1Click:Connect(function()
	deleteToolActive = not deleteToolActive
	if deleteToolActive then
		extDeleteBtn.BackgroundColor3 = Color3.fromRGB(24, 40, 30)
		extDeleteBtn.TextColor3 = Color3.fromRGB(34, 197, 94)
		extDeleteBtn.Text = "Delete: ON"
	else
		extDeleteBtn.BackgroundColor3 = Color3.fromRGB(40, 24, 24)
		extDeleteBtn.TextColor3 = Color3.fromRGB(239, 68, 68)
		extDeleteBtn.Text = "Delete: OFF"
	end
end)

local extReturnBtn = Instance.new("TextButton")
extReturnBtn.Size = UDim2.new(1, -12, 0, 28)
extReturnBtn.Position = UDim2.new(0, 6, 0, 66)
extReturnBtn.BackgroundColor3 = Color3.fromRGB(40, 24, 24)
extReturnBtn.TextColor3 = Color3.fromRGB(239, 68, 68)
extReturnBtn.TextSize = 11
extReturnBtn.Font = Enum.Font.GothamBold
extReturnBtn.Text = "Return: OFF"
extReturnBtn.Parent = externalWidget

local erCorner = Instance.new("UICorner")
erCorner.CornerRadius = UDim.new(0, 6)
erCorner.Parent = extReturnBtn

extReturnBtn.MouseButton1Click:Connect(function()
	returnToolActive = not returnToolActive
	if returnToolActive then
		extReturnBtn.BackgroundColor3 = Color3.fromRGB(24, 40, 30)
		extReturnBtn.TextColor3 = Color3.fromRGB(34, 197, 94)
		extReturnBtn.Text = "Return: ON"
	else
		extReturnBtn.BackgroundColor3 = Color3.fromRGB(40, 24, 24)
		extReturnBtn.TextColor3 = Color3.fromRGB(239, 68, 68)
		extReturnBtn.Text = "Return: OFF"
	end
end)

createToggle(otherPage, "Delete Tool Widget", false, function(state)
	deleteWidgetEnabled = state
	externalWidget.Visible = state
end)

table.insert(activeConnections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if input.KeyCode == toggleKey then
		mainWindow.Visible = not mainWindow.Visible
	end
end))

table.insert(activeConnections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not gameProcessed and (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) then
		if deleteWidgetEnabled and deleteToolActive then
			local targetPart = mouse.Target
			if targetPart and targetPart:IsA("BasePart") and not targetPart:IsDescendantOf(Players) then
				table.insert(deletedPartsHistory, {
					Name = targetPart.Name, Size = targetPart.Size, CFrame = targetPart.CFrame,
					Material = targetPart.Material, Color = targetPart.Color, Transparency = targetPart.Transparency,
					Parent = targetPart.Parent, Shape = targetPart.Shape
				})
				targetPart:Destroy()
			end
		end

		if deleteWidgetEnabled and returnToolActive then
			if #deletedPartsHistory > 0 then
				local lastData = table.remove(deletedPartsHistory)
				local restoredPart = Instance.new("Part")
				restoredPart.Name = lastData.Name
				restoredPart.Size = lastData.Size
				restoredPart.CFrame = lastData.CFrame
				restoredPart.Material = lastData.Material
				restoredPart.Color = lastData.Color
				restoredPart.Transparency = lastData.Transparency
				restoredPart.Shape = lastData.Shape
				restoredPart.Anchored = true
				restoredPart.Parent = lastData.Parent or workspace
			end
		end
	end
end))
