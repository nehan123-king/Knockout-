

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local skinsFolder = ReplicatedStorage:WaitForChild("Models"):WaitForChild("Skins")

local currentSkin = nil
local currentSkinName = nil

------------------------------------------------
-- CLEANUP
------------------------------------------------
local function cleanup()
	if currentSkin then
		currentSkin:Destroy()
		currentSkin = nil
	end
end

------------------------------------------------
-- CHECK IF IN SKIN
------------------------------------------------
local function isInSkin(obj)
	local parent = obj.Parent
	while parent do
		if parent == currentSkin then
			return true
		end
		parent = parent.Parent
	end
	return false
end

------------------------------------------------
-- HIDE CHARACTER
------------------------------------------------
local function hideCharacter(char)
	for _, obj in ipairs(char:GetDescendants()) do
		if isInSkin(obj) then continue end

		if obj:IsA("BasePart") then
			obj.Transparency = 1
			obj.LocalTransparencyModifier = 1
		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			obj.Transparency = 1
		end
	end
end

------------------------------------------------
-- RESTORE
------------------------------------------------
local function restoreCharacter(char)
	for _, obj in ipairs(char:GetDescendants()) do
		if obj:IsA("BasePart") then
			obj.Transparency = 0
			obj.LocalTransparencyModifier = 0
		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			obj.Transparency = 0
		end
	end
end

------------------------------------------------
-- PREPARE SKIN
------------------------------------------------
local function prepareSkin(model)
	for _, obj in ipairs(model:GetDescendants()) do
		if obj:IsA("BasePart") then
			obj.Anchored = false
			obj.CanCollide = false
			obj.Massless = true

			if obj:IsA("MeshPart") or obj:IsA("UnionOperation") then
				obj.Transparency = 0
				obj.LocalTransparencyModifier = 0
			else
				obj.Transparency = 1
				obj.LocalTransparencyModifier = 1
			end

		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			obj.Transparency = 0

		elseif obj:IsA("Script") or obj:IsA("LocalScript") or obj:IsA("Humanoid") then
			obj:Destroy()
		end
	end
end

------------------------------------------------
-- APPLY SKIN
------------------------------------------------
local function applySkin(model)
	local char = player.Character
	if not char then return end

	local root = char:FindFirstChild("HumanoidRootPart")
	if not root then return end

	cleanup()

	local clone = model:Clone()
	currentSkin = clone
	currentSkinName = model.Name
	clone.Parent = char

	prepareSkin(clone)

	if not clone.PrimaryPart then
		clone.PrimaryPart = clone:FindFirstChildWhichIsA("BasePart")
	end

	if clone.PrimaryPart then
		clone:SetPrimaryPartCFrame(root.CFrame)
	end

	for _, part in ipairs(clone:GetDescendants()) do
		if part:IsA("BasePart") then
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = root
			weld.Part1 = part
			weld.Parent = part
		end
	end

	hideCharacter(char)
end

------------------------------------------------
-- REMOVE SKIN
------------------------------------------------
local function removeSkin()
	cleanup()
	local char = player.Character
	if char then
		restoreCharacter(char)
	end
	currentSkinName = nil
end

------------------------------------------------
-- GET SKINS
------------------------------------------------
local function getSkins()
	local list = {}
	for _, obj in ipairs(skinsFolder:GetDescendants()) do
		if obj:IsA("Model") then
			table.insert(list, obj)
		end
	end
	return list
end

------------------------------------------------
-- SEARCH SYSTEM (INSANE)
------------------------------------------------
local function normalize(text)
	return string.gsub(string.lower(text), "%s+", "")
end

-- simple fuzzy scoring
local function getScore(name, query)
	name = normalize(name)
	query = normalize(query)

	if query == "" then return 1 end

	if name == query then
		return 100 -- exact match
	end

	if string.find(name, query, 1, true) then
		return 80 -- direct contains
	end

	-- fuzzy match (character order match)
	local score = 0
	local j = 1

	for i = 1, #name do
		if name:sub(i,i) == query:sub(j,j) then
			score += 1
			j += 1
			if j > #query then break end
		end
	end

	return score
end

------------------------------------------------
-- PREVIEW
------------------------------------------------
local function createPreview(parent, model)
	local viewport = Instance.new("ViewportFrame")
	viewport.Size = UDim2.new(1, -20, 0, 160)
	viewport.Position = UDim2.new(0,10,0,10)
	viewport.BackgroundColor3 = Color3.fromRGB(20,20,25)
	viewport.Parent = parent
	Instance.new("UICorner", viewport).CornerRadius = UDim.new(0,12)

	local cam = Instance.new("Camera")
	viewport.CurrentCamera = cam
	cam.Parent = viewport

	local clone = model:Clone()
	prepareSkin(clone)
	clone.Parent = viewport

	local primary = clone:FindFirstChildWhichIsA("BasePart")
	if primary then
		clone.PrimaryPart = primary
		clone:SetPrimaryPartCFrame(
			CFrame.new(0,0,0) * CFrame.Angles(0, math.rad(180), 0)
		)

		cam.CFrame = CFrame.new(Vector3.new(0,1.5,6), Vector3.new(0,1,0))
	end
end

------------------------------------------------
-- GUI
------------------------------------------------
local function makeGui()
	local old = player.PlayerGui:FindFirstChild("SkinGui")
	if old then old:Destroy() end

	local gui = Instance.new("ScreenGui")
	gui.Name = "SkinGui"
	gui.ResetOnSpawn = false
	gui.Parent = player.PlayerGui

	local frame = Instance.new("Frame")
	frame.Size = UDim2.new(0, 380, 0, 600)
	frame.Position = UDim2.new(0.05, 0, 0.1, 0)
	frame.BackgroundColor3 = Color3.fromRGB(18,18,22)
	frame.Active = true
	frame.Draggable = true
	frame.Parent = gui
	Instance.new("UICorner", frame).CornerRadius = UDim.new(0,16)

	local title = Instance.new("TextLabel")
	title.Size = UDim2.new(1, -60, 0, 40)
	title.Position = UDim2.new(0,15,0,10)
	title.Text = "Skin Selector"
	title.Font = Enum.Font.FredokaOne
	title.TextSize = 26
	title.TextColor3 = Color3.fromRGB(255,255,255)
	title.TextStrokeTransparency = 0
	title.BackgroundTransparency = 1
	title.Parent = frame

	local search = Instance.new("TextBox")
	search.Size = UDim2.new(1,-20,0,40)
	search.Position = UDim2.new(0,10,0,60)
	search.PlaceholderText = "🔍 Search skins..."
	search.Font = Enum.Font.FredokaOne
	search.TextSize = 18
	search.TextColor3 = Color3.fromRGB(255,255,255)
	search.BackgroundColor3 = Color3.fromRGB(35,35,45)
	search.Parent = frame
	Instance.new("UICorner", search).CornerRadius = UDim.new(0,12)

	local scroll = Instance.new("ScrollingFrame")
	scroll.Size = UDim2.new(1,-20,1,-160)
	scroll.Position = UDim2.new(0,10,0,110)
	scroll.BackgroundTransparency = 1
	scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
	scroll.Parent = frame

	local layout = Instance.new("UIListLayout")
	layout.Padding = UDim.new(0,12)
	layout.Parent = scroll

	local skins = getSkins()
	local containers = {}

	local function populate(filter)
		for _, c in ipairs(containers) do c:Destroy() end
		containers = {}

		local scored = {}

		for _, skin in ipairs(skins) do
			local score = getScore(skin.Name, filter)
			if score > 0 then
				table.insert(scored, {skin = skin, score = score})
			end
		end

		table.sort(scored, function(a,b)
			return a.score > b.score
		end)

		for _, data in ipairs(scored) do
			local skin = data.skin

			local container = Instance.new("Frame")
			container.Size = UDim2.new(1,0,0,240)
			container.BackgroundColor3 = Color3.fromRGB(28,28,36)
			container.Parent = scroll
			Instance.new("UICorner", container).CornerRadius = UDim.new(0,14)

			createPreview(container, skin)

			local btn = Instance.new("TextButton")
			btn.Size = UDim2.new(1,-20,0,50)
			btn.Position = UDim2.new(0,10,1,-60)
			btn.Text = skin.Name
			btn.Font = Enum.Font.FredokaOne
			btn.TextSize = 20
			btn.TextColor3 = Color3.fromRGB(255,255,255)
			btn.TextStrokeTransparency = 0
			btn.BackgroundColor3 = Color3.fromRGB(90,90,140)
			btn.Parent = container
			Instance.new("UICorner", btn).CornerRadius = UDim.new(0,12)

			btn.MouseButton1Click:Connect(function()
				applySkin(skin)
			end)

			table.insert(containers, container)
		end
	end

	populate("")

	search:GetPropertyChangedSignal("Text"):Connect(function()
		populate(search.Text)
	end)

	local removeBtn = Instance.new("TextButton")
	removeBtn.Size = UDim2.new(1,-20,0,50)
	removeBtn.Position = UDim2.new(0,10,1,-60)
	removeBtn.Text = "Remove Skin"
	removeBtn.Font = Enum.Font.FredokaOne
	removeBtn.TextSize = 20
	removeBtn.TextColor3 = Color3.fromRGB(255,255,255)
	removeBtn.TextStrokeTransparency = 0
	removeBtn.BackgroundColor3 = Color3.fromRGB(200,70,70)
	removeBtn.Parent = frame
	Instance.new("UICorner", removeBtn).CornerRadius = UDim.new(0,14)

	removeBtn.MouseButton1Click:Connect(removeSkin)

	UserInputService.InputBegan:Connect(function(input, gpe)
		if gpe then return end
		if input.KeyCode == Enum.KeyCode.C then
			gui.Enabled = not gui.Enabled
		end
	end)
end

------------------------------------------------
-- START
------------------------------------------------
makeGui()

player.CharacterAdded:Connect(function()
	task.wait(0.5)
	if currentSkinName then
		local skin = skinsFolder:FindFirstChild(currentSkinName, true)
		if skin then
			applySkin(skin)
		end
	end
end)
