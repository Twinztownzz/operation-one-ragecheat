local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Camera = Workspace.CurrentCamera

-- Cleanup old menu safely
local oldMenu = CoreGui:FindFirstChild("MyMenu")
if oldMenu then oldMenu:Destroy() end
pcall(function() RunService:UnbindFromRenderStep("DynamicZeroRecoil") end)

-- === SETTINGS ===
local PURPLE_COLOR = Color3.fromRGB(138, 43, 226)
local TRANSPARENCY = 0.35
local ESP_COLOR = Color3.new(1, 1, 1)
local BASE_PATH = "Viewmodels.LocalViewmodel.M4.Gun"
local FIRERATE_PATH = BASE_PATH .. ".firerate"
local FIRERATE_NEW_VALUE = 9999
local ACCURACY_PATH = BASE_PATH .. ".accuracy"
local ACCURACY_NEW_VALUE = 2.3
local RELOAD_SPEED_PATH = BASE_PATH .. ".reload_speed"
local RELOAD_SPEED_NEW_VALUE = 0
local MAG_PATH = BASE_PATH .. ".mag"
local MAG_NEW_VALUE = 999
local SPREAD_PATH = BASE_PATH .. ".spread"
local SPREAD_NEW_VALUE = 0
-- ================

-- State
local MenuOpen = true
local Tabs = {"Legit", "Ragebot", "Visuals", "Settings"}
local CurrentTab = "Ragebot"
local ragebotVisualsEnabled = false
local noRecoilEnabled = false
local noSpreadEnabled = false
local ESP_ENABLED = false
local TRACERS_ENABLED = false
local BOXES_ENABLED = false
local TEAM_CHECK = true
local MAX_STUCK_TIME = 1.5
local firerateEnabled = false
local accuracyEnabled = false
local instantReloadEnabled = false
local maxMagEnabled = false
local originalFirerateValue = nil
local originalAccuracyValue = nil
local originalReloadSpeedValue = nil
local originalMagValue = nil
local originalSpreadValue = nil

-- === ESP STATE ===
local ESP_Table = {}
local TeammateHighlights = {}
local viewmodels = Workspace:FindFirstChild("Viewmodels")

-- === DRAWING HELPER ===
local function newDrawing(type, props)
    local obj = Drawing.new(type)
    for k, v in pairs(props) do obj[k] = v end
    return obj
end

-- === TEAMMATE DETECTION ===
Workspace.ChildAdded:Connect(function(child)
    if child:IsA("Highlight") then TeammateHighlights[child] = true end
end)
Workspace.ChildRemoved:Connect(function(child)
    if child:IsA("Highlight") then TeammateHighlights[child] = nil end
end)

local function is_teammate(model)
    for highlight in pairs(TeammateHighlights) do
        if highlight.Adornee == model then return true end
    end
    return false
end

-- === ESP SETUP ===
local function hideESP(ui)
    if ui.Box then ui.Box.Visible = false end
    if ui.Tracer then ui.Tracer.Visible = false end
    if ui.Name then ui.Name.Visible = false end
end

local function createPlayerESP(player)
    if player == Players.LocalPlayer or ESP_Table[player] then return end
    ESP_Table[player] = {
        Player = player,
        Box = newDrawing("Square", {Thickness = 1.5, Filled = false, Color = ESP_COLOR, Visible = false}),
        Tracer = newDrawing("Line", {Thickness = 1.5, Color = Color3.fromRGB(255, 0, 0), Visible = false}),
        Name = newDrawing("Text", {Size = 12, Center = true, Outline = true, Color = Color3.fromRGB(255,255,255), Visible = false}),
        LastPosition = nil,
        StuckTime = 0
    }
end

for _, p in ipairs(Players:GetPlayers()) do createPlayerESP(p) end
Players.PlayerAdded:Connect(createPlayerESP)

-- === FIND CHARACTER ===
local function findCharacter(player)
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        local root = char:FindFirstChild("HumanoidRootPart")
        if hum and root then return char, hum, root end
    end
    return nil
end

-- === GET BOX ===
local function getBoundingBox(character)
    local hrp = character:FindFirstChild("HumanoidRootPart")
    local hum = character:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return nil end
    
    local pos = hrp.Position
    local top = pos + Vector3.new(0, hum.Height/2 + 0.5, 0)
    local bottom = pos - Vector3.new(0, hum.Height/2, 0)
    
    local topPos, vis1 = Camera:WorldToViewportPoint(top)
    local bottomPos, vis2 = Camera:WorldToViewportPoint(bottom)
    
    if not vis1 or not vis2 then return nil end
    
    local height = math.abs(topPos.Y - bottomPos.Y)
    local width = height * 0.6
    
    return Vector2.new(topPos.X - width/2, topPos.Y), width, height
end

-- === TRACER START POINT ===
local function getTracerOrigin()
    local vp = Camera.ViewportSize
    return Vector2.new(vp.X/2, vp.Y)
end

-- === MAIN ESP UPDATE LOOP ===
RunService.RenderStepped:Connect(function(dt)
    Camera = Workspace.CurrentCamera
    for player, ui in pairs(ESP_Table) do
        if not BOXES_ENABLED and not TRACERS_ENABLED then
            hideESP(ui)
            continue
        end
        
        if TEAM_CHECK and player.Team == Players.LocalPlayer.Team then
            hideESP(ui)
            continue
        end
        
        local char, hum, root = findCharacter(player)
        if not char or not hum or hum.Health <= 0 then
            hideESP(ui)
            continue
        end
        
        local pos, w, h = getBoundingBox(char)
        if not pos then
            hideESP(ui)
            continue
        end
        
        ui.Box.Visible = BOXES_ENABLED
        if BOXES_ENABLED then
            ui.Box.Size = Vector2.new(w, h)
            ui.Box.Position = pos
        end
        
        ui.Tracer.Visible = TRACERS_ENABLED
        if TRACERS_ENABLED then
            ui.Tracer.From = getTracerOrigin()
            ui.Tracer.To = Vector2.new(pos.X + w/2, pos.Y + h)
        end
        
        ui.Name.Visible = BOXES_ENABLED
        if BOXES_ENABLED then
            ui.Name.Text = player.Name
            ui.Name.Position = Vector2.new(pos.X + w/2, pos.Y - 16)
        end
    end
end)

-- === HELPER: Get Value Instance ===
local function GetValueInstance(pathStr)
    local current = game
    local parts = string.split(pathStr, ".")
    for _, part in ipairs(parts) do
        current = current:FindFirstChild(part)
        if not current then return nil end
    end
    return current
end

-- === FIRERATE ===
local function SetFirerate(enabled)
    local firerateObj = GetValueInstance(FIRERATE_PATH)
    if not firerateObj then return end
    if enabled then
        if originalFirerateValue == nil then originalFirerateValue = firerateObj.Value end
        firerateObj.Value = FIRERATE_NEW_VALUE
    else
        if originalFirerateValue ~= nil then firerateObj.Value = originalFirerateValue end
    end
end

-- === ACCURACY ===
local function SetAccuracy(enabled)
    local accuracyObj = GetValueInstance(ACCURACY_PATH)
    if not accuracyObj then return end
    if enabled then
        if originalAccuracyValue == nil then originalAccuracyValue = accuracyObj.Value end
        accuracyObj.Value = ACCURACY_NEW_VALUE
    else
        if originalAccuracyValue ~= nil then accuracyObj.Value = originalAccuracyValue end
    end
end

-- === INSTANT RELOAD ===
local function SetInstantReload(enabled)
    local reloadSpeedObj = GetValueInstance(RELOAD_SPEED_PATH)
    if not reloadSpeedObj then return end
    if enabled then
        if originalReloadSpeedValue == nil then originalReloadSpeedValue = reloadSpeedObj.Value end
        reloadSpeedObj.Value = RELOAD_SPEED_NEW_VALUE
    else
        if originalReloadSpeedValue ~= nil then reloadSpeedObj.Value = originalReloadSpeedValue end
    end
end

-- === MAX MAG ===
local function SetMaxMag(enabled)
    local magObj = GetValueInstance(MAG_PATH)
    if not magObj then return end
    if enabled then
        if originalMagValue == nil then originalMagValue = magObj.Value end
        magObj.Value = MAG_NEW_VALUE
    else
        if originalMagValue ~= nil then magObj.Value = originalMagValue end
    end
end

-- === NO SPREAD ===
local function SetNoSpread(enabled)
    local spreadObj = GetValueInstance(SPREAD_PATH)
    if not spreadObj then return end
    if enabled then
        if originalSpreadValue == nil then originalSpreadValue = spreadObj.Value end
        spreadObj.Value = SPREAD_NEW_VALUE
    else
        if originalSpreadValue ~= nil then spreadObj.Value = originalSpreadValue end
    end
end

-- === NO RECOIL ===
local lp = Players.LocalPlayer
local NoRecoilSettings = {
    Enabled = false,
    ActiveKey = Enum.UserInputType.MouseButton1,
    Sens = 1.0,
    Smoothness = 0.12
}
local targetX, targetY, targetZ = 0, 0, 0
local isFiring = false
local recoveryTime = 0

local function syncRecoil()
    local x, y, z = Camera.CFrame:ToEulerAnglesYXZ()
    targetX, targetY, targetZ = x, y, z
end
syncRecoil()

RunService:BindToRenderStep("DynamicZeroRecoil", Enum.RenderPriority.Camera.Value + 1, function(dt)
    if not NoRecoilSettings.Enabled then return end
    if UIS.MouseBehavior == Enum.MouseBehavior.Default then syncRecoil() return end
    local char = lp.Character
    if not char or not char:FindFirstChildOfClass("Humanoid") or char.Humanoid.Health <= 0 then
        syncRecoil()
        return
    end
    local delta = UIS:GetMouseDelta()
    local _, _, realZ = Camera.CFrame:ToEulerAnglesYXZ()
    local flip = (Camera.CFrame.UpVector.Y < 0) and -1 or 1
    if UIS:IsMouseButtonPressed(NoRecoilSettings.ActiveKey) then
        isFiring = true
        recoveryTime = 0.35
        targetY = targetY - math.rad(delta.X * NoRecoilSettings.Sens * 0.48 * flip)
        targetX = targetX - math.rad(delta.Y * NoRecoilSettings.Sens * 0.48)
        targetX = math.clamp(targetX, math.rad(-85), math.rad(85))
        Camera.CFrame = CFrame.new(Camera.CFrame.Position) * CFrame.fromEulerAnglesYXZ(targetX, targetY, realZ)
    else
        if isFiring then
            if recoveryTime > 0 then
                recoveryTime = recoveryTime - dt
                if delta.Magnitude > 0 then
                    targetY = targetY - math.rad(delta.X * NoRecoilSettings.Sens * 0.48 * flip)
                    targetX = targetX - math.rad(delta.Y * NoRecoilSettings.Sens * 0.48)
                    targetX = math.clamp(targetX, math.rad(-85), math.rad(85))
                end
                Camera.CFrame = CFrame.new(Camera.CFrame.Position) * CFrame.fromEulerAnglesYXZ(targetX, targetY, realZ)
            else
                syncRecoil()
                isFiring = false
            end
        else
            syncRecoil()
        end
    end
end)

local function SetNoRecoil(enabled)
    noRecoilEnabled = enabled
    NoRecoilSettings.Enabled = enabled
    if enabled then syncRecoil() end
end

-- === RAGEBOT VISUALS ===
local originalParts = {}
local function GetViewmodel() return Workspace:FindFirstChild("Viewmodels") and Workspace.Viewmodels:FindFirstChild("LocalViewmodel") end

local function CacheOriginalColors()
    table.clear(originalParts)
    local c = lp.Character
    if not c then return end
    for _, d in ipairs(c:GetDescendants()) do
        if d:IsA("BasePart") then
            originalParts[d] = {Transparency = d.Transparency, Color = d.Color}
        elseif d:IsA("Clothing") or d:IsA("Shirt") or d:IsA("Pants") then
            originalParts[d] = {Transparency = d.Transparency}
        end
    end
end

local function ApplyPurple(on)
    local c = lp.Character
    if not c then return end
    if not on then
        for obj, data in pairs(originalParts) do
            if obj then
                if obj:IsA("BasePart") then
                    obj.Transparency = data.Transparency
                    obj.Color = data.Color
                else
                    obj.Transparency = data.Transparency
                end
            end
        end
        table.clear(originalParts)
        return
    end
    CacheOriginalColors()
    for _, d in ipairs(c:GetDescendants()) do
        if d:IsA("BasePart") then
            d.Color = PURPLE_COLOR
            d.Transparency = TRANSPARENCY
        elseif d:IsA("Clothing") or d:IsA("Shirt") or d:IsA("Pants") then
            d.Transparency = TRANSPARENCY
        end
    end
    local hands = lp:FindFirstChild("Items") and lp.Items:FindFirstChild("Hands")
    if hands then
        for _, d in ipairs(hands:GetDescendants()) do
            if d:IsA("BasePart") then
                d.Color = PURPLE_COLOR
                d.Transparency = TRANSPARENCY
            end
        end
    end
    local vm = GetViewmodel()
    if vm then
        for _, d in ipairs(vm:GetDescendants()) do
            if d:IsA("BasePart") then
                d.Color = PURPLE_COLOR
                d.Transparency = TRANSPARENCY
            end
        end
    end
end

lp.CharacterAdded:Connect(function()
    task.wait(0.2)
    if ragebotVisualsEnabled then ApplyPurple(true) end
end)

-- === MENU TOGGLE (RIGHT SHIFT) ===
UIS.InputBegan:ConnectFunction(function(input)
    if input.KeyCode == Enum.KeyCode.RightShift then
        MenuOpen = not MenuOpen
        MainFrame.Visible = MenuOpen
    end
end)

-- === MENU BUILD ===
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MyMenu"
ScreenGui.Parent = CoreGui
ScreenGui.DisplayOrder = 100000
ScreenGui.ResetOnSpawn = false
pcall(function() ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling end)

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 460, 0, 620)
MainFrame.Position = UDim2.new(0.02, 0, 0.05, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 30)
MainFrame.BorderSizePixel = 1
MainFrame.BorderColor3 = Color3.fromRGB(70, 60, 120)
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Visible = MenuOpen
MainFrame.Parent = ScreenGui
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 10)

local TabBar = Instance.new("Frame")
TabBar.Size = UDim2.new(1, 0, 0, 44)
TabBar.BackgroundColor3 = Color3.fromRGB(32, 32, 45)
TabBar.Parent = MainFrame
Instance.new("UICorner", TabBar).CornerRadius = UDim.new(0, 10)

local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -20, 1, -80)
Content.Position = UDim2.new(0, 10, 0, 60)
Content.BackgroundTransparency = 1
Content.Parent = MainFrame

local TabBtns = {}
for i, name in ipairs(Tabs) do
    local btn = Instance.new("TextButton")
    btn.Text = name
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 13
    btn.TextColor3 = name == CurrentTab and Color3.fromRGB(160, 100, 255) or Color3.fromRGB(140, 140, 155)
    btn.BackgroundColor3 = name == CurrentTab and Color3.fromRGB(55, 45, 85) or Color3.fromRGB(40, 40, 55)
    btn.Size = UDim2.new(0.235, 0, 0.78, 0)
    btn.Position = UDim2.new(0.012 + ((i-1) * 0.252), 0, 0.11, 0)
    btn.Parent = TabBar
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    TabBtns[name] = btn
    btn.MouseButton1Click:Connect(function()
        CurrentTab = name
        for tabName, b in pairs(TabBtns) do
            b.TextColor3 = tabName == CurrentTab and Color3.fromRGB(160, 100, 255) or Color3.fromRGB(140, 140, 155)
            b.BackgroundColor3 = tabName == CurrentTab and Color3.fromRGB(55, 45, 85) or Color3.fromRGB(40, 40, 55)
        end
        UpdateContent()
    end)
end

-- === TOGGLE HELPER ===
local function MakeToggle(parent, labelText, enabled, callback)
    local Container = Instance.new("Frame")
    Container.Size = UDim2.new(1, 0, 0, 52)
    Container.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
    Container.Parent = parent
    Instance.new("UICorner", Container).CornerRadius = UDim.new(0, 8)
    
    local Label = Instance.new("TextLabel")
    Label.Text = labelText
    Label.Font = Enum.Font.Gotham
    Label.TextColor3 = Color3.fromRGB(230, 230, 230)
    Label.TextSize = 14
    Label.Size = UDim2.new(1, -75, 1, 0)
    Label.Position = UDim2.new(0, 15, 0, 0)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.BackgroundTransparency = 1
    Label.Parent = Container
    
    local ToggleBtn = Instance.new("TextButton")
    ToggleBtn.Size = UDim2.new(0, 48, 0, 24)
    ToggleBtn.Position = UDim2.new(1, -63, 0.5, -12)
    ToggleBtn.BackgroundColor3 = enabled and Color3.fromRGB(100, 60, 200) or Color3.fromRGB(70, 70, 90)
    ToggleBtn.Text = ""
    ToggleBtn.Parent = Container
    Instance.new("UICorner", ToggleBtn).CornerRadius = UDim.new(1, 0)
    
    local Knob = Instance.new("Frame")
    Knob.Size = UDim2.new(0, 18, 0, 18)
    Knob.Position = enabled and UDim2.new(1, -21, 0.5, -9) or UDim2.new(0, 3, 0.5, -9)
    Knob.BackgroundColor3 = Color3.new(1, 1, 1)
    Knob.Parent = ToggleBtn
    Instance.new("UICorner", Knob).CornerRadius = UDim.new(1, 0)
    
    local isOn = enabled
    ToggleBtn.MouseButton1Click:Connect(function()
        isOn = not isOn
        ToggleBtn.BackgroundColor3 = isOn and Color3.fromRGB(100, 60, 200) or Color3.fromRGB(70, 70, 90)
        TweenService:Create(Knob, TweenInfo.new(0.12), {Position = isOn and UDim2.new(1, -21, 0.5, -9) or UDim2.new(0, 3, 0.5, -9)}):Play()
        callback(isOn)
    end)
    
    return Container
end

-- === UPDATE CONTENT ===
function UpdateContent()
    Content:ClearAllChildren()
    local yPos = 10
    
    if CurrentTab == "Legit" then
        local Title = Instance.new("TextLabel")
        Title.Text = "Legit"
        Title.Font = Enum.Font.GothamBold
        Title.TextColor3 = Color3.fromRGB(255, 255, 255)
        Title.TextSize = 17
        Title.Size = UDim2.new(1, 0, 0, 28)
        Title.TextXAlignment = Enum.TextXAlignment.Left
        Title.BackgroundTransparency = 1
        Title.Parent = Content
        
        local Info = Instance.new("TextLabel")
        Info.Text = "No features here yet.\nRightShift to toggle menu."
        Info.Font = Enum.Font.Gotham
        Info.TextColor3 = Color3.fromRGB(160, 160, 170)
        Info.TextSize = 14
        Info.Size = UDim2.new(1, 0, 0, 60)
        Info.Position = UDim2.new(0, 0, 0, 40)
        Info.TextXAlignment = Enum.TextXAlignment.Left
        Info.BackgroundTransparency = 1
        Info.TextWrapped = true
        Info.Parent = Content
        
    elseif CurrentTab == "Ragebot" then
        local Title = Instance.new("TextLabel")
        Title.Text = "Ragebot"
        Title.Font = Enum.Font.GothamBold
        Title.TextColor3 = Color3.fromRGB(255, 255, 255)
        Title.TextSize = 17
        Title.Size = UDim2.new(1, 0, 0, 28)
        Title.TextXAlignment = Enum.TextXAlignment.Left
        Title.BackgroundTransparency = 1
        Title.Parent = Content
        
        MakeToggle(Content, "Firerate (M4 → 9999)", firerateEnabled, function(val)
            firerateEnabled = val
            SetFirerate(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Accuracy (M4 → 2.3)", accuracyEnabled, function(val)
            accuracyEnabled = val
            SetAccuracy(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Instant Reload", instantReloadEnabled, function(val)
            instantReloadEnabled = val
            SetInstantReload(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Max Magazine (999)", maxMagEnabled, function(val)
            maxMagEnabled = val
            SetMaxMag(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "No Spread", noSpreadEnabled, function(val)
            noSpreadEnabled = val
            SetNoSpread(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "No Recoil (hold LMB)", noRecoilEnabled, function(val)
            noRecoilEnabled = val
            SetNoRecoil(val)
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Ragebot Visuals", ragebotVisualsEnabled, function(val)
            ragebotVisualsEnabled = val
            ApplyPurple(val)
        end).Position = UDim2.new(0, 0, 0, yPos)
        
    elseif CurrentTab == "Visuals" then
        local Title = Instance.new("TextLabel")
        Title.Text = "Visuals — Universal ESP"
        Title.Font = Enum.Font.GothamBold
        Title.TextColor3 = Color3.fromRGB(255, 255, 255)
        Title.TextSize = 17
        Title.Size = UDim2.new(1, 0, 0, 28)
        Title.TextXAlignment = Enum.TextXAlignment.Left
        Title.BackgroundTransparency = 1
        Title.Parent = Content
        
        MakeToggle(Content, "Player Boxes", BOXES_ENABLED, function(val)
            BOXES_ENABLED = val
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Tracers", TRACERS_ENABLED, function(val)
            TRACERS_ENABLED = val
        end).Position = UDim2.new(0, 0, 0, yPos); yPos += 62
        
        MakeToggle(Content, "Team Check", TEAM_CHECK, function(val)
            TEAM_CHECK = val
        end).Position = UDim2.new(0, 0, 0, yPos)
        
    elseif CurrentTab == "Settings" then
        local Title = Instance.new("TextLabel")
        Title.Text = "Settings"
        Title.Font = Enum.Font.GothamBold
        Title.TextColor3 = Color3.fromRGB(255, 255, 255)
        Title.TextSize = 17
        Title.Size = UDim2.new(1, 0, 0, 28)
        Title.TextXAlignment = Enum.TextXAlignment.Left
        Title.BackgroundTransparency = 1
        Title.Parent = Content
        
        local Hint = Instance.new("TextLabel")
        Hint.Text = "Press RightShift to open/close menu"
        Hint.Font = Enum.Font.Gotham
        Hint.TextColor3 = Color3.fromRGB(150, 150, 165)
        Hint.TextSize = 13
        Hint.Size = UDim2.new(1, 0, 0, 40)
        Hint.Position = UDim2.new(0, 0, 0, 45)
        Hint.TextXAlignment = Enum.TextXAlignment.Left
        Hint.BackgroundTransparency = 1
        Hint.Parent = Content
    end
end

UpdateContent()

print("✅ Menu Loaded! — RightShift = Toggle")
