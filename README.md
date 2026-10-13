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
-- ================
-- State
local MenuOpen = true
local Tabs = {"Legit", "Ragebot", "Visuals", "Settings"}
local CurrentTab = "Ragebot"
local ragebotVisualsEnabled = false
local noRecoilEnabled = false
local ESP_ENABLED = false
local TRACERS_ENABLED = false
local HEALTHBARS_ENABLED = false
local SKELETONS_ENABLED = false
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
-- === ESP STATE ===
local ESP_Table = {}
local skeleton_list = {}
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
    ui.Box.Visible = false
    ui.Tracer.Visible = false
    ui.Health.Visible = false
    ui.Name.Visible = false
end
local function createPlayerESP(player)
    if player == Players.LocalPlayer or ESP_Table[player] then return end
    ESP_Table[player] = {
        Player = player,
        Box = newDrawing("Square", {Thickness = 1, Filled = false, Color = ESP_COLOR, Visible = false}),
        Tracer = newDrawing("Line", {Thickness = 1, Color = ESP_COLOR, Visible = false}),
        Health = newDrawing("Line", {Thickness = 5, Visible = false}),
        Name = newDrawing("Text", {Size = 13, Center = true, Outline = false, Color = Color3.fromRGB(255,255,255), Visible = false}),
        LastPosition = nil,
        StuckTime = 0
    }
end
for _, p in ipairs(Players:GetPlayers()) do createPlayerESP(p) end
Players.PlayerAdded:Connect(createPlayerESP)
-- === FIND CHARACTER ===
local function findCharacter(player)
    for _, model in ipairs(Workspace:GetChildren()) do
        if model:IsA("Model") and model.Name == player.Name then
            local hum = model:FindFirstChildOfClass("Humanoid")
            local root = model:FindFirstChild("HumanoidRootPart")
            if hum and root then return model, hum, root end
        end
    end
    return nil
end
-- === GET BOX ===
local function getBoundingBox(character)
    local cf, size = character:GetBoundingBox()
    local top = cf.Position + Vector3.new(0, size.Y/2, 0)
    local bottom = cf.Position - Vector3.new(0, size.Y/2, 0)
    local topPos, vis1 = Camera:WorldToViewportPoint(top)
    local bottomPos, vis2 = Camera:WorldToViewportPoint(bottom)
    if not vis1 or not vis2 then return nil end
    local height = math.abs(topPos.Y - bottomPos.Y)
    local width = height / 2
    return Vector2.new(topPos.X - width/2, topPos.Y), width, height
end
-- === TRACER START POINT ===
local function getTracerOrigin()
    local vp = Camera.ViewportSize
    return Vector2.new(vp.X/2, vp.Y)
end
-- === SKELETON SYSTEM ===
local bones = {
    {"torso", "head"}, {"torso", "shoulder1"}, {"torso", "shoulder2"},
    {"shoulder1", "arm1"}, {"shoulder2", "arm2"}, {"torso", "hip1"},
    {"torso", "hip2"}, {"hip1", "leg1"}, {"hip2", "leg2"}
}
local requiredBones = {"torso", "head", "shoulder1", "shoulder2", "arm1", "arm2", "hip1", "hip2", "leg1", "leg2"}
local function isValidSkeletonModel(model)
    if not model or not model.Parent or model.Name == "LocalViewmodel" or not viewmodels or model.Parent ~= viewmodels then return false end
    local torso = model:FindFirstChild("torso")
    return torso and torso:IsA("BasePart")
end
local function createSkeleton(character)
    if not character or skeleton_list[character] or not isValidSkeletonModel(character) then return end
    local charBones = {}
    for _, name in ipairs(requiredBones) do
        local b = character:FindFirstChild(name)
        if not b or not b:IsA("BasePart") then return end
        charBones[name] = b
    end
    local lines = {}
    for i = 1, #bones do
        lines[i] = newDrawing("Line", {Visible = false, Color = ESP_COLOR, Thickness = 1, Transparency = 1})
    end
    skeleton_list[character] = {Lines = lines, Bones = charBones}
end
local function removeSkeleton(character)
    local data = skeleton_list[character]
    if not data then return end
    for _, line in ipairs(data.Lines) do line:Remove() end
    skeleton_list[character] = nil
end
-- === MAIN ESP UPDATE LOOP ===
RunService.RenderStepped:Connect(function(dt)
    if not viewmodels then viewmodels = Workspace:FindFirstChild("Viewmodels") end
    Camera = Workspace.CurrentCamera
    -- Player ESP
    for player, ui in pairs(ESP_Table) do
        task.spawn(function()
            if TEAM_CHECK and player.Team == Players.LocalPlayer.Team then hideESP(ui) return end
            local char, hum, root = findCharacter(player)
            if not char or not hum or hum.Health <= 0 then hideESP(ui) return end
            local pos, w, h = getBoundingBox(char)
            if not pos then hideESP(ui) return end
            if ui.LastPosition and (ui.LastPosition - pos).Magnitude < 1 then
                ui.StuckTime = ui.StuckTime + dt
            else
                ui.StuckTime = 0
            end
            if ui.StuckTime >= MAX_STUCK_TIME then hideESP(ui) return end
            ui.LastPosition = pos
            ui.Box.Visible = ESP_ENABLED
            if ESP_ENABLED then
                ui.Box.Size = Vector2.new(w, h)
                ui.Box.Position = pos
                ui.Box.Color = ESP_COLOR
            end
            ui.Name.Visible = ESP_ENABLED
            if ESP_ENABLED then
                local myRoot = Players.LocalPlayer.Character and Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                local dist = myRoot and math.floor((myRoot.Position - root.Position).Magnitude) or "?"
                ui.Name.Text = player.Name .. " [" .. dist .. "m]"
                ui.Name.Position = Vector2.new(pos.X + w/2, pos.Y - 15)
            end
            ui.Tracer.Visible = TRACERS_ENABLED
            if TRACERS_ENABLED then
                ui.Tracer.From = getTracerOrigin()
                ui.Tracer.To = Vector2.new(pos.X + w/2, pos.Y)
                ui.Tracer.Color = ESP_COLOR
            end
            ui.Health.Visible = HEALTHBARS_ENABLED
            if HEALTHBARS_ENABLED then
                local hp = hum.Health / hum.MaxHealth
                local hpHeight = h * hp
                ui.Health.From = Vector2.new(pos.X - 5, pos.Y + h)
                ui.Health.To = Vector2.new(pos.X - 5, pos.Y + h - hpHeight)
                ui.Health.Color = Color3.fromRGB(255 - (255*hp), 255*hp, 0)
            end
        end)
    end
    -- Skeleton ESP
    if SKELETONS_ENABLED and viewmodels then
        for _, model in ipairs(viewmodels:GetChildren()) do
            if not model:IsA("Model") then continue end
            if not isValidSkeletonModel(model) then removeSkeleton(model) continue end
            if is_teammate(model) then removeSkeleton(model) continue end
            local torso = model:FindFirstChild("torso")
            if not torso or torso.Transparency >= 1 then removeSkeleton(model) continue end
            local pos, onScreen = Camera:WorldToScreenPoint(torso.Position)
            if not onScreen or (Camera.CFrame.Position - torso.Position).Magnitude > 3571 then
                removeSkeleton(model)
                continue
            end
            if not skeleton_list[model] then createSkeleton(model) end
            local skel = skeleton_list[model]
            if not skel then continue end
            for i, conn in ipairs(bones) do
                local b1, b2 = skel.Bones[conn[1]], skel.Bones[conn[2]]
                if b1 and b2 then
                    local p1 = Camera:WorldToViewportPoint(b1.Position)
                    local p2 = Camera:WorldToViewportPoint(b2.Position)
                    if p1.Z > 0 and p2.Z > 0 then
                        skel.Lines[i].From = Vector2.new(p1.X, p1.Y)
                        skel.Lines[i].To = Vector2.new(p2.X, p2.Y)
                        skel.Lines[i].Visible = true
                        skel.Lines[i].Color = ESP_COLOR
                    else
                        skel.Lines[i].Visible = false
                    end
                else
                    skel.Lines[i].Visible = false
                end
            end
        end
    else
        for _, data in pairs(skeleton_list) do
            for _, line in ipairs(data.Lines) do
                line.Visible = false
            end
        end
    end
end)
-- === HELPER: Get Value Object from Path ===
local function GetValueInstance(pathStr)
    local current = Workspace
    for _, part in ipairs(string.split(pathStr, ".")) do
        current = current:FindFirstChild(part)
        if not current then return nil end
    end
    return current
end
-- === FIRERATE ===
local function SetFirerate(enabled)
    local firerateObj = GetValueInstance(FIRERATE_PATH)
    if not firerateObj then
        warn("❌ Firerate path not found: Workspace." .. FIRERATE_PATH)
        return
    end
    if enabled then
        if originalFirerateValue == nil then
            originalFirerateValue = firerateObj.Value
        end
        firerateObj.Value = FIRERATE_NEW_VALUE
        print("✅ Firerate → " .. FIRERATE_NEW_VALUE)
    else
        if originalFirerateValue ~= nil then
            firerateObj.Value = originalFirerateValue
            print("✅ Firerate restored → " .. originalFirerateValue)
        end
    end
end
-- === ACCURACY ===
local function SetAccuracy(enabled)
    local accuracyObj = GetValueInstance(ACCURACY_PATH)
    if not accuracyObj then
        warn("❌ Accuracy path not found: Workspace." .. ACCURACY_PATH)
        return
    end
    if enabled then
        if originalAccuracyValue == nil then
            originalAccuracyValue = accuracyObj.Value
        end
        accuracyObj.Value = ACCURACY_NEW_VALUE
        print("✅ Accuracy → " .. ACCURACY_NEW_VALUE)
    else
        if originalAccuracyValue ~= nil then
            accuracyObj.Value = originalAccuracyValue
            print("✅ Accuracy restored → " .. originalAccuracyValue)
        end
    end
end
-- === INSTANT RELOAD ===
local function SetInstantReload(enabled)
    local reloadSpeedObj = GetValueInstance(RELOAD_SPEED_PATH)
    if not reloadSpeedObj then
        warn("❌ Reload speed path not found: Workspace." .. RELOAD_SPEED_PATH)
        return
    end
    if enabled then
        if originalReloadSpeedValue == nil then
            originalReloadSpeedValue = reloadSpeedObj.Value
        end
        reloadSpeedObj.Value = RELOAD_SPEED_NEW_VALUE
        print("✅ Instant Reload → " .. RELOAD_SPEED_NEW_VALUE)
    else
        if originalReloadSpeedValue ~= nil then
            reloadSpeedObj.Value = originalReloadSpeedValue
            print("✅ Reload Speed restored → " .. originalReloadSpeedValue)
        end
    end
end
-- === MAX MAG ===
local function SetMaxMag(enabled)
    local magObj = GetValueInstance(MAG_PATH)
    if not magObj then
        warn("❌ Mag path not found: Workspace." .. MAG_PATH)
        return
    end
    if enabled then
        if originalMagValue == nil then
            originalMagValue = magObj.Value
        end
        magObj.Value = MAG_NEW_VALUE
        print("✅ Max Mag → " .. MAG_NEW_VALUE)
    else
        if originalMagValue ~= nil then
            magObj.Value = originalMagValue
            print("✅ Mag restored → " .. originalMagValue)
        end
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
    if enabled then syncRecoil() print("✅ No Recoil: ON")
    else isFiring = false; recoveryTime = 0; print("✅ No Recoil: OFF") end
end
-- === RAGEBOT VISUALS (LOCAL PLAYER) ===
local original = {}
local function GetViewmodel() return Workspace:FindFirstChild("Viewmodels") and Workspace.Viewmodels:FindFirstChild("LocalViewmodel") end
local function CacheOriginal() table.clear(original); local c=lp.Character; if not c then return end; for _,d in ipairs(c:GetDescendants()) do if d:IsA("BasePart") then original[d]={Transparency=d.Transparency,Color=d.Color} elseif d:IsA("Clothing") or d:IsA("Shirt") or d:IsA("Pants") then original[d]={Transparency=d.Transparency} end end end
local function ApplyPurple(on)
    local c=lp.Character; if not c then return end
    if not on then for o,d in pairs(original) do if o then if o:IsA("BasePart") then o.Transparency=d.Transparency; o.Color=d.Color else o.Transparency=d.Transparency end end end; table.clear(original); return end
    CacheOriginal()
    for _,d in ipairs(c:GetDescendants()) do if d:IsA("BasePart") then d.Color=PURPLE_COLOR; d.Transparency=TRANSPARENCY elseif d:IsA("Clothing") or d:IsA("Shirt") or d:IsA("Pants") then d.Transparency=TRANSPARENCY end end
    local hands=lp:FindFirstChild("Items") and lp.Items:FindFirstChild("Hands")
    if hands then for _,d in ipairs(hands:GetDescendants()) do if d:IsA("BasePart") then d.Color=PURPLE_COLOR; d.Transparency=TRANSPARENCY end end end
    local vm=GetViewmodel()
    if vm then for _,d in ipairs(vm:GetDescendants()) do if d:IsA("BasePart") then d.Color=PURPLE_COLOR; d.Transparency=TRANSPARENCY end end end
end
lp.CharacterAdded:Connect(function() task.wait(0.15); if ragebotVisualsEnabled then ApplyPurple(true) end end)
-- === MENU BUILD ===
local ScreenGui=Instance.new("ScreenGui"); ScreenGui.Name="MyMenu"; ScreenGui.Parent=CoreGui; ScreenGui.DisplayOrder=100000; ScreenGui.ResetOnSpawn=false
local MainFrame=Instance.new("Frame"); MainFrame.Size=UDim2.new(0,480,0,580); MainFrame.Position=UDim2.new(0.05,0,0.05,0); MainFrame.BackgroundColor3=Color3.fromRGB(24,24,32); MainFrame.BorderSizePixel=2; MainFrame.BorderColor3=Color3.fromRGB(60,60,80); MainFrame.Active=true; MainFrame.Draggable=true; MainFrame.Visible=MenuOpen; MainFrame.Parent=ScreenGui; Instance.new("UICorner",MainFrame).CornerRadius=UDim.new(0,10)
local TabBar=Instance.new("Frame"); TabBar.Size=UDim2.new(1,0,0,45); TabBar.BackgroundColor3=Color3.fromRGB(35,35,50); TabBar.Parent=MainFrame; Instance.new("UICorner",TabBar).CornerRadius=UDim.new(0,10)
local Content=Instance.new("Frame"); Content.Size=UDim2.new(1,-20,1,-85); Content.Position=UDim2.new(0,10,0,65); Content.BackgroundTransparency=1; Content.Parent=MainFrame
local TabBtns={}
for i,n in ipairs(Tabs) do
    local btn=Instance.new("TextButton"); btn.Text=n; btn.Font=Enum.Font.GothamBold; btn.TextSize=14
    btn.TextColor3=n==CurrentTab and Color3.fromRGB(100,200,255) or Color3.fromRGB(170,170,180)
    btn.BackgroundColor3=n==CurrentTab and Color3.fromRGB(50,50,75) or Color3.fromRGB(40,40,55)
    btn.Size=UDim2.new(0.23,0,0.8,0); btn.Position=UDim2.new(0.015+((i-1)*0.25),0,0.1,0)
    btn.Parent=TabBar; Instance.new("UICorner",btn).CornerRadius=UDim.new(0,6); TabBtns[n]=btn
    btn.MouseButton1Click:Connect(function() CurrentTab=n; for name,b in pairs(TabBtns) do b.TextColor3=name==CurrentTab and Color3.fromRGB(100,200,255) or Color3.fromRGB(170,170,180); b.BackgroundColor3=name==CurrentTab and Color3.fromRGB(50,50,75) or Color3.fromRGB(40,40,55) end; UpdateContent() end)
end
function UpdateContent()
    Content:ClearAllChildren()
    if CurrentTab=="Ragebot" then
        local T=Instance.new("TextLabel"); T.Text="Ragebot"; T.Font=Enum.Font.GothamBold; T.TextColor3=Color3.fromRGB(255,255,255); T.TextSize=18; T.Size=UDim2.new(1,0,0,30); T.TextXAlignment=Enum.TextXAlignment.Left; T.BackgroundTransparency=1; T.Parent=Content
        local FG=Instance.new("Frame"); FG.Size=UDim2.new(1,0,0,55); FG.Position=UDim2.new(0,0,0,50); FG.BackgroundColor3=Color3.fromRGB(40,40,55); FG.Parent=Content; Instance.new("UICorner",FG).CornerRadius=UDim.new(0,8)
        local FL=Instance.new("TextLabel"); FL.Text="Firerate (M4 → 9999)"; FL.Font=Enum.Font.Gotham; FL.TextColor3=Color3.fromRGB(220,220,220); FL.TextSize=15; FL.Size=UDim2.new(1,-80,1,0); FL.Position=UDim2.new(0,15,0,0); FL.TextXAlignment=Enum.TextXAlignment.Left; FL.BackgroundTransparency=1; FL.Parent=FG
        local FS=Instance.new("TextButton"); FS.Size=UDim2.new(0,50,0,26); FS.Position=UDim2.new(1,-60,0.5,-13); FS.BackgroundColor3=firerateEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); FS.Text=""; FS.Parent=FG; Instance.new("UICorner",FS).CornerRadius=UDim.new(1,0)
        local FK=Instance.new("Frame"); FK.Size=UDim2.new(0,20,0,20); FK.Position=firerateEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); FK.BackgroundColor3=Color3.new(1,1,1); FK.Parent=FS; Instance.new("UICorner",FK).CornerRadius=UDim.new(1,0)
        FS.MouseButton1Click:Connect(function() firerateEnabled=not firerateEnabled; FS.BackgroundColor3=firerateEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(FK,TweenInfo.new(0.15),{Position=firerateEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); SetFirerate(firerateEnabled) end)
        local AG=Instance.new("Frame"); AG.Size=UDim2.new(1,0,0,55); AG.Position=UDim2.new(0,0,0,115); AG.BackgroundColor3=Color3.fromRGB(40,40,55); AG.Parent=Content; Instance.new("UICorner",AG).CornerRadius=UDim.new(0,8)
        local AL=Instance.new("TextLabel"); AL.Text="Accuracy (M4 → 2.3)"; AL.Font=Enum.Font.Gotham; AL.TextColor3=Color3.fromRGB(220,220,220); AL.TextSize=15; AL.Size=UDim2.new(1,-80,1,0); AL.Position=UDim2.new(0,15,0,0); AL.TextXAlignment=Enum.TextXAlignment.Left; AL.BackgroundTransparency=1; AL.Parent=AG
        local AS=Instance.new("TextButton"); AS.Size=UDim2.new(0,50,0,26); AS.Position=UDim2.new(1,-60,0.5,-13); AS.BackgroundColor3=accuracyEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); AS.Text=""; AS.Parent=AG; Instance.new("UICorner",AS).CornerRadius=UDim.new(1,0)
        local AK=Instance.new("Frame"); AK.Size=UDim2.new(0,20,0,20); AK.Position=accuracyEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); AK.BackgroundColor3=Color3.new(1,1,1); AK.Parent=AS; Instance.new("UICorner",AK).CornerRadius=UDim.new(1,0)
        AS.MouseButton1Click:Connect(function() accuracyEnabled=not accuracyEnabled; AS.BackgroundColor3=accuracyEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(AK,TweenInfo.new(0.15),{Position=accuracyEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); SetAccuracy(accuracyEnabled) end)
        local RG=Instance.new("Frame"); RG.Size=UDim2.new(1,0,0,55); RG.Position=UDim2.new(0,0,0,180); RG.BackgroundColor3=Color3.fromRGB(40,40,55); RG.Parent=Content; Instance.new("UICorner",RG).CornerRadius=UDim.new(0,8)
        local RL=Instance.new("TextLabel"); RL.Text="Instant Reload (M4 → 0)"; RL.Font=Enum.Font.Gotham; RL.TextColor3=Color3.fromRGB(220,220,220); RL.TextSize=15; RL.Size=UDim2.new(1,-80,1,0); RL.Position=UDim2.new(0,15,0,0); RL.TextXAlignment=Enum.TextXAlignment.Left; RL.BackgroundTransparency=1; RL.Parent=RG
        local RS=Instance.new("TextButton"); RS.Size=UDim2.new(0,50,0,26); RS.Position=UDim2.new(1,-60,0.5,-13); RS.BackgroundColor3=instantReloadEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); RS.Text=""; RS.Parent=RG; Instance.new("UICorner",RS).CornerRadius=UDim.new(1,0)
        local RK=Instance.new("Frame"); RK.Size=UDim2.new(0,20,0,20); RK.Position=instantReloadEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); RK.BackgroundColor3=Color3.new(1,1,1); RK.Parent=RS; Instance.new("UICorner",RK).CornerRadius=UDim.new(1,0)
        RS.MouseButton1Click:Connect(function() instantReloadEnabled=not instantReloadEnabled; RS.BackgroundColor3=instantReloadEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(RK,TweenInfo.new(0.15),{Position=instantReloadEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); SetInstantReload(instantReloadEnabled) end)
        local MG=Instance.new("Frame"); MG.Size=UDim2.new(1,0,0,55); MG.Position=UDim2.new(0,0,0,245); MG.BackgroundColor3=Color3.fromRGB(40,40,55); MG.Parent=Content; Instance.new("UICorner",MG).CornerRadius=UDim.new(0,8)
        local ML=Instance.new("TextLabel"); ML.Text="Max Mag (M4 → 999)"; ML.Font=Enum.Font.Gotham; ML.TextColor3=Color3.fromRGB(220,220,220); ML.TextSize=15; ML.Size=UDim2.new(1,-80,1,0); ML.Position=UDim2.new(0,15,0,0); ML.TextXAlignment=Enum.TextXAlignment.Left; ML.BackgroundTransparency=1; ML.Parent=MG
        local MS=Instance.new("TextButton"); MS.Size=UDim2.new(0,50,0,26); MS.Position=UDim2.new(1,-60,0.5,-13); MS.BackgroundColor3=maxMagEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); MS.Text=""; MS.Parent=MG; Instance.new("UICorner",MS).CornerRadius=UDim.new(1,0)
        local MK=Instance.new("Frame"); MK.Size=UDim2.new(0,20,0,20); MK.Position=maxMagEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); MK.BackgroundColor3=Color3.new(1,1,1); MK.Parent=MS; Instance.new("UICorner",MK).CornerRadius=UDim.new(1,0)
        MS.MouseButton1Click:Connect(function() maxMagEnabled=not maxMagEnabled; MS.BackgroundColor3=maxMagEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(MK,TweenInfo.new(0.15),{Position=maxMagEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); SetMaxMag(maxMagEnabled) end)
        local NRG=Instance.new("Frame"); NRG.Size=UDim2.new(1,0,0,55); NRG.Position=UDim2.new(0,0,0,310); NRG.BackgroundColor3=Color3.fromRGB(40,40,55); NRG.Parent=Content; Instance.new("UICorner",NRG).CornerRadius=UDim.new(0,8)
        local NRL=Instance.new("TextLabel"); NRL.Text="No Recoil"; NRL.Font=Enum.Font.Gotham; NRL.TextColor3=Color3.fromRGB(220,220,220); NRL.TextSize=15; NRL.Size=UDim2.new(1,-80,1,0); NRL.Position=UDim2.new(0,15,0,0); NRL.TextXAlignment=Enum.TextXAlignment.Left; NRL.BackgroundTransparency=1; NRL.Parent=NRG
        local NRS=Instance.new("TextButton"); NRS.Size=UDim2.new(0,50,0,26); NRS.Position=UDim2.new(1,-60,0.5,-13); NRS.BackgroundColor3=noRecoilEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); NRS.Text=""; NRS.Parent=NRG; Instance.new("UICorner",NRS).CornerRadius=UDim.new(1,0)
        local NRK=Instance.new("Frame"); NRK.Size=UDim2.new(0,20,0,20); NRK.Position=noRecoilEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); NRK.BackgroundColor3=Color3.new(1,1,1); NRK.Parent=NRS; Instance.new("UICorner",NRK).CornerRadius=UDim.new(1,0)
        NRS.MouseButton1Click:Connect(function() noRecoilEnabled=not noRecoilEnabled; NRS.BackgroundColor3=noRecoilEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(NRK,TweenInfo.new(0.15),{Position=noRecoilEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); SetNoRecoil(noRecoilEnabled) end)
        local VG=Instance.new("Frame"); VG.Size=UDim2.new(1,0,0,55); VG.Position=UDim2.new(0,0,0,375); VG.BackgroundColor3=Color3.fromRGB(40,40,55); VG.Parent=Content; Instance.new("UICorner",VG).CornerRadius=UDim.new(0,8)
        local VL=Instance.new("TextLabel"); VL.Text="Ragebot Visuals (Your Character)"; VL.Font=Enum.Font.Gotham; VL.TextColor3=Color3.fromRGB(220,220,220); VL.TextSize=15; VL.Size=UDim2.new(1,-80,1,0); VL.Position=UDim2.new(0,15,0,0); VL.TextXAlignment=Enum.TextXAlignment.Left; VL.BackgroundTransparency=1; VL.Parent=VG
        local VS=Instance.new("TextButton"); VS.Size=UDim2.new(0,50,0,26); VS.Position=UDim2.new(1,-60,0.5,-13); VS.BackgroundColor3=ragebotVisualsEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); VS.Text=""; VS.Parent=VG; Instance.new("UICorner",VS).CornerRadius=UDim.new(1,0)
        local VK=Instance.new("Frame"); VK.Size=UDim2.new(0,20,0,20); VK.Position=ragebotVisualsEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); VK.BackgroundColor3=Color3.new(1,1,1); VK.Parent=VS; Instance.new("UICorner",VK).CornerRadius=UDim.new(1,0)
        VS.MouseButton1Click:Connect(function() ragebotVisualsEnabled=not ragebotVisualsEnabled; VS.BackgroundColor3=ragebotVisualsEnabled and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(VK,TweenInfo.new(0.15),{Position=ragebotVisualsEnabled and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play(); ApplyPurple(ragebotVisualsEnabled) end)
    elseif CurrentTab=="Visuals" then
        local T=Instance.new("TextLabel"); T.Text="Visuals — Universal ESP"; T.Font=Enum.Font.GothamBold; T.TextColor3=Color3.fromRGB(255,255,255); T.TextSize=18; T.Size=UDim2.new(1,0,0,30); T.TextXAlignment=Enum.TextXAlignment.Left; T.BackgroundTransparency=1; T.Parent=Content
        -- MAIN ESP TOGGLE
        local EG=Instance.new("Frame"); EG.Size=UDim2.new(1,0,0,55); EG.Position=UDim2.new(0,0,0,50); EG.BackgroundColor3=Color3.fromRGB(40,40,55); EG.Parent=Content; Instance.new("UICorner",EG).CornerRadius=UDim.new(0,8)
        local EL=Instance.new("TextLabel"); EL.Text="Player ESP (Boxes + Names)"; EL.Font=Enum.Font.Gotham; EL.TextColor3=Color3.fromRGB(220,220,220); EL.TextSize=15; EL.Size=UDim2.new(1,-80,1,0); EL.Position=UDim2.new(0,15,0,0); EL.TextXAlignment=Enum.TextXAlignment.Left; EL.BackgroundTransparency=1; EL.Parent=EG
        local ES=Instance.new("TextButton"); ES.Size=UDim2.new(0,50,0,26); ES.Position=UDim2.new(1,-60,0.5,-13); ES.BackgroundColor3=ESP_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); ES.Text=""; ES.Parent=EG; Instance.new("UICorner",ES).CornerRadius=UDim.new(1,0)
        local EK=Instance.new("Frame"); EK.Size=UDim2.new(0,20,0,20); EK.Position=ESP_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); EK.BackgroundColor3=Color3.new(1,1,1); EK.Parent=ES; Instance.new("UICorner",EK).CornerRadius=UDim.new(1,0)
        ES.MouseButton1Click:Connect(function() ESP_ENABLED=not ESP_ENABLED; ES.BackgroundColor3=ESP_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(EK,TweenInfo.new(0.15),{Position=ESP_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play() end)
        -- TRACERS TOGGLE
        local TG=Instance.new("Frame"); TG.Size=UDim2.new(1,0,0,55); TG.Position=UDim2.new(0,0,0,115); TG.BackgroundColor3=Color3.fromRGB(40,40,55); TG.Parent=Content; Instance.new("UICorner",TG).CornerRadius=UDim.new(0,8)
        local TL=Instance.new("TextLabel"); TL.Text="Tracers"; TL.Font=Enum.Font.Gotham; TL.TextColor3=Color3.fromRGB(220,220,220); TL.TextSize=15; TL.Size=UDim2.new(1,-80,1,0); TL.Position=UDim2.new(0,15,0,0); TL.TextXAlignment=Enum.TextXAlignment.Left; TL.BackgroundTransparency=1; TL.Parent=TG
        local TS=Instance.new("TextButton"); TS.Size=UDim2.new(0,50,0,26); TS.Position=UDim2.new(1,-60,0.5,-13); TS.BackgroundColor3=TRACERS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TS.Text=""; TS.Parent=TG; Instance.new("UICorner",TS).CornerRadius=UDim.new(1,0)
        local TK=Instance.new("Frame"); TK.Size=UDim2.new(0,20,0,20); TK.Position=TRACERS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); TK.BackgroundColor3=Color3.new(1,1,1); TK.Parent=TS; Instance.new("UICorner",TK).CornerRadius=UDim.new(1,0)
        TS.MouseButton1Click:Connect(function() TRACERS_ENABLED=not TRACERS_ENABLED; TS.BackgroundColor3=TRACERS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(TK,TweenInfo.new(0.15),{Position=TRACERS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play() end)
        -- HEALTH BARS TOGGLE
        local HG=Instance.new("Frame"); HG.Size=UDim2.new(1,0,0,55); HG.Position=UDim2.new(0,0,0,180); HG.BackgroundColor3=Color3.fromRGB(40,40,55); HG.Parent=Content; Instance.new("UICorner",HG).CornerRadius=UDim.new(0,8)
        local HL=Instance.new("TextLabel"); HL.Text="Health Bars"; HL.Font=Enum.Font.Gotham; HL.TextColor3=Color3.fromRGB(220,220,220); HL.TextSize=15; HL.Size=UDim2.new(1,-80,1,0); HL.Position=UDim2.new(0,15,0,0); HL.TextXAlignment=Enum.TextXAlignment.Left; HL.BackgroundTransparency=1; HL.Parent=HG
        local HS=Instance.new("TextButton"); HS.Size=UDim2.new(0,50,0,26); HS.Position=UDim2.new(1,-60,0.5,-13); HS.BackgroundColor3=HEALTHBARS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); HS.Text=""; HS.Parent=HG; Instance.new("UICorner",HS).CornerRadius=UDim.new(1,0)
        local HK=Instance.new("Frame"); HK.Size=UDim2.new(0,20,0,20); HK.Position=HEALTHBARS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); HK.BackgroundColor3=Color3.new(1,1,1); HK.Parent=HS; Instance.new("UICorner",HK).CornerRadius=UDim.new(1,0)
        HS.MouseButton1Click:Connect(function() HEALTHBARS_ENABLED=not HEALTHBARS_ENABLED; HS.BackgroundColor3=HEALTHBARS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(HK,TweenInfo.new(0.15),{Position=HEALTHBARS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play() end)
        -- SKELETON TOGGLE ✅ NEW
        local SKG=Instance.new("Frame"); SKG.Size=UDim2.new(1,0,0,55); SKG.Position=UDim2.new(0,0,0,245); SKG.BackgroundColor3=Color3.fromRGB(40,40,55); SKG.Parent=Content; Instance.new("UICorner",SKG).CornerRadius=UDim.new(0,8)
        local SKL=Instance.new("TextLabel"); SKL.Text="Skeleton ESP (Viewmodels)"; SKL.Font=Enum.Font.Gotham; SKL.TextColor3=Color3.fromRGB(220,220,220); SKL.TextSize=15; SKL.Size=UDim2.new(1,-80,1,0); SKL.Position=UDim2.new(0,15,0,0); SKL.TextXAlignment=Enum.TextXAlignment.Left; SKL.BackgroundTransparency=1; SKL.Parent=SKG
        local SKS=Instance.new("TextButton"); SKS.Size=UDim2.new(0,50,0,26); SKS.Position=UDim2.new(1,-60,0.5,-13); SKS.BackgroundColor3=SKELETONS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); SKS.Text=""; SKS.Parent=SKG; Instance.new("UICorner",SKS).CornerRadius=UDim.new(1,0)
        local SKK=Instance.new("Frame"); SKK.Size=UDim2.new(0,20,0,20); SKK.Position=SKELETONS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10); SKK.BackgroundColor3=Color3.new(1,1,1); SKK.Parent=SKS; Instance.new("UICorner",SKK).CornerRadius=UDim.new(1,0)
        SKS.MouseButton1Click:Connect(function() SKELETONS_ENABLED=not SKELETONS_ENABLED; SKS.BackgroundColor3=SKELETONS_ENABLED and Color3.fromRGB(50,205,120) or Color3.fromRGB(80,80,100); TweenService:Create(SKK,TweenInfo.new(0.15),{Position=SKELETONS_ENABLED and UDim2.new(1,-23,0.5,-10) or UDim2.new(0,3,0.5,-10)}):Play() end)
    else
        local p=Instance.new("TextLabel"); p.Text=CurrentTab.."\n\nComing Soon"; p.Font=Enum.Font.GothamBold; p.TextColor3=Color3.fromRGB(120,120,140); p.TextSize=20; p.Size=UDim2.new(1,0,1,0); p.BackgroundTransparency=1; p.Parent=Content
    end
end
UIS.InputBegan:Connect(function(i,g) if g then return end; if i.KeyCode==Enum.KeyCode.RightShift then MenuOpen=not MenuOpen; MainFrame.Visible=MenuOpen; if MenuOpen then UpdateContent() end end end)
UpdateContent()
print("✅ Loaded — ESP Upgraded + Skeleton Toggle Added")
print("💡 RightShift = Toggle Menu")   
