-- Junior Hub | LinoriaLib v5
-- Auto Clicker + Flight + Mob ESP + Player ESP + Auto Dwarf King Quest
-- Place in StarterPlayerScripts as a LocalScript

local repo = 'https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/'

local Library      = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager  = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()

local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService       = game:GetService("RunService")
local RS               = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera

-- ========================================
--             AUTO CLICKER
-- ========================================

local AutoClickEnabled = false
local ClickDelay       = 1 / 20
local lastClick        = 0

RunService.Heartbeat:Connect(function()
    if not AutoClickEnabled then return end
    local now = tick()
    if now - lastClick >= ClickDelay then
        lastClick = now
        mouse1click()
    end
end)

-- ========================================
--               NO FOG
-- ========================================

local NoFogEnabled        = false
local FogConn             = nil
local FogDescConn         = nil
local originalFogEnd      = nil
local originalFogStart    = nil
local savedAtmosphere     = {}

local function applyNoFog()
    local Lighting = game:GetService("Lighting")
    originalFogEnd   = Lighting.FogEnd
    originalFogStart = Lighting.FogStart
    Lighting.FogEnd   = 100000
    Lighting.FogStart = 100000

    -- Zero out any Atmosphere (volumetric fog / haze)
    for _, obj in ipairs(Lighting:GetChildren()) do
        if obj:IsA("Atmosphere") then
            savedAtmosphere = {
                Density = obj.Density,
                Offset  = obj.Offset,
                Haze    = obj.Haze,
                Glare   = obj.Glare,
            }
            obj.Density = 0
            obj.Offset  = 0
            obj.Haze    = 0
            obj.Glare   = 0
        end
    end

    -- Watch for game re-adding fog or Atmosphere
    FogConn = Lighting:GetPropertyChangedSignal("FogEnd"):Connect(function()
        if NoFogEnabled then Lighting.FogEnd = 100000 end
    end)
    FogDescConn = Lighting.DescendantAdded:Connect(function(obj)
        if NoFogEnabled and obj:IsA("Atmosphere") then
            obj.Density = 0
            obj.Offset  = 0
            obj.Haze    = 0
            obj.Glare   = 0
        end
    end)
end

local function removeNoFog()
    if FogConn     then FogConn:Disconnect();     FogConn     = nil end
    if FogDescConn then FogDescConn:Disconnect(); FogDescConn = nil end
    local Lighting = game:GetService("Lighting")
    if originalFogEnd   then Lighting.FogEnd   = originalFogEnd   end
    if originalFogStart then Lighting.FogStart = originalFogStart end
    for _, obj in ipairs(Lighting:GetChildren()) do
        if obj:IsA("Atmosphere") and savedAtmosphere.Density then
            obj.Density = savedAtmosphere.Density
            obj.Offset  = savedAtmosphere.Offset
            obj.Haze    = savedAtmosphere.Haze
            obj.Glare   = savedAtmosphere.Glare
        end
    end
    savedAtmosphere = {}
end

-- ========================================
--         REMOVE TEXTURES / VISUALS
-- ========================================

local savedShadows       = nil
local savedGrassLength   = nil
local savedDecorations   = nil

-- Shadows
local function removeShadows()
    local Lighting = game:GetService("Lighting")
    savedShadows = Lighting.GlobalShadows
    Lighting.GlobalShadows = false
end
local function restoreShadows()
    local Lighting = game:GetService("Lighting")
    if savedShadows ~= nil then Lighting.GlobalShadows = savedShadows end
end

-- Grass & terrain decorations
local function removeGrass()
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if terrain then
        savedGrassLength   = terrain.GrassLength
        savedDecorations   = terrain.Decoration
        terrain.GrassLength  = 0
        terrain.Decoration   = false
    end
end
local function restoreGrass()
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if terrain then
        if savedGrassLength ~= nil then terrain.GrassLength = savedGrassLength end
        if savedDecorations ~= nil then terrain.Decoration  = savedDecorations end
    end
end

-- Part textures (sets all BasePart materials to SmoothPlastic, saves originals)
local savedMaterials  = {}
local savedTextures   = {}

local function removeTextures()
    savedMaterials = {}
    savedTextures  = {}
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            savedMaterials[obj] = obj.Material
            obj.Material = Enum.Material.SmoothPlastic
        end
        if obj:IsA("Texture") or obj:IsA("Decal") then
            savedTextures[obj] = obj.Transparency
            obj.Transparency = 1
        end
    end
end
local function restoreTextures()
    for obj, mat in pairs(savedMaterials) do
        if obj and obj.Parent then obj.Material = mat end
    end
    for obj, trans in pairs(savedTextures) do
        if obj and obj.Parent then obj.Transparency = trans end
    end
    savedMaterials = {}
    savedTextures  = {}
end

-- Post-processing effects (bloom, blur, color correction, sun rays)
local savedEffects = {}
local function removePostFX()
    savedEffects = {}
    local Lighting = game:GetService("Lighting")
    for _, obj in ipairs(Lighting:GetChildren()) do
        if obj:IsA("PostEffect") then
            savedEffects[obj] = obj.Enabled
            obj.Enabled = false
        end
    end
end
local function restorePostFX()
    for obj, state in pairs(savedEffects) do
        if obj and obj.Parent then obj.Enabled = state end
    end
    savedEffects = {}
end


-- ========================================
--           FULL BRIGHT
-- ========================================

local FullBrightEnabled    = false
local FB_savedAmbient      = nil
local FB_savedOutdoor      = nil
local FB_savedBrightness   = nil
local FB_savedColorShift   = nil
local FB_Conn              = nil

local function applyFullBright()
    local L = game:GetService("Lighting")
    FB_savedAmbient    = L.Ambient
    FB_savedOutdoor    = L.OutdoorAmbient
    FB_savedBrightness = L.Brightness
    FB_savedColorShift = L.ColorShift_Bottom
    L.Ambient          = Color3.new(1, 1, 1)
    L.OutdoorAmbient   = Color3.new(1, 1, 1)
    L.Brightness       = 2
    L.ColorShift_Bottom = Color3.new(0, 0, 0)
    -- lock it
    FB_Conn = L:GetPropertyChangedSignal("Brightness"):Connect(function()
        if FullBrightEnabled then L.Brightness = 2 end
    end)
end

local function removeFullBright()
    if FB_Conn then FB_Conn:Disconnect(); FB_Conn = nil end
    local L = game:GetService("Lighting")
    if FB_savedAmbient    then L.Ambient          = FB_savedAmbient    end
    if FB_savedOutdoor    then L.OutdoorAmbient   = FB_savedOutdoor    end
    if FB_savedBrightness then L.Brightness       = FB_savedBrightness end
    if FB_savedColorShift then L.ColorShift_Bottom = FB_savedColorShift end
end

-- ========================================
--         TIME OF DAY CONTROL
-- ========================================

local ClockConn = nil

local function setTimeOfDay(hour)
    local L = game:GetService("Lighting")
    L.ClockTime = hour
end

local function lockTime(hour)
    if ClockConn then ClockConn:Disconnect(); ClockConn = nil end
    local L = game:GetService("Lighting")
    L.ClockTime = hour
    ClockConn = RunService.Heartbeat:Connect(function()
        L.ClockTime = hour
    end)
end

local function unlockTime()
    if ClockConn then ClockConn:Disconnect(); ClockConn = nil end
end

-- (Auto Sprint removed: handled natively by game)

-- ========================================
--           AUTO TAKE QUEST
-- ========================================

local AutoQuestEnabled = false
local AutoQuestThread  = nil

local function runQuestSequence()
    pcall(function()
        RS.Msg.RemoteEvent.RemoteEvent:FireServer(
            "\232\167\166\229\143\145\232\129\138\229\164\169",
            {"\229\147\136\229\136\169\229\155\160\231\137\185", "10010100"}
        )
    end)
    task.wait(0.4)
    pcall(function()
        RS.Msg.RemoteEvent.RemoteEvent:FireServer(
            "\232\167\166\229\143\145\232\129\138\229\164\169",
            {"\229\147\136\229\136\169\229\155\160\231\137\185", 10010501}
        )
    end)
    task.wait(0.4)
    pcall(function()
        RS.Msg.Function.TalkFunc:InvokeServer(
            "\229\143\145\230\148\190\228\187\187\229\138\161",
            {"\228\187\187\229\138\161" .. "6"}
        )
    end)
    task.wait(0.4)
    pcall(function()
        RS.Msg.RemoteFunction.Setting:InvokeServer("TASK", 1)
    end)
end

local function startAutoQuest()
    AutoQuestThread = task.spawn(function()
        while AutoQuestEnabled do
            runQuestSequence()
            task.wait(2)
        end
    end)
end

local function stopAutoQuest()
    AutoQuestEnabled = false
    if AutoQuestThread then
        task.cancel(AutoQuestThread)
        AutoQuestThread = nil
    end
end

-- ========================================
--               FLIGHT
-- ========================================

local FlyEnabled = false
local FlySpeed   = 60
local FlyConn    = nil
local FlyVel     = nil
local FlyAlign   = nil
local FlyAtt0    = nil
local FlyAtt1    = nil

local function getRoot()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Humanoid")
end

local function startFly()
    local root = getRoot()
    local hum  = getHumanoid()
    if not root or not hum then return end

    hum.PlatformStand = true

    FlyAtt0        = Instance.new("Attachment")
    FlyAtt0.Parent = root
    FlyAtt1        = Instance.new("Attachment")
    FlyAtt1.Parent = workspace.Terrain

    FlyVel                        = Instance.new("LinearVelocity")
    FlyVel.Attachment0            = FlyAtt0
    FlyVel.MaxForce               = math.huge
    FlyVel.RelativeTo             = Enum.ActuatorRelativeTo.World
    FlyVel.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
    FlyVel.VectorVelocity         = Vector3.zero
    FlyVel.Parent                 = root

    FlyAlign                    = Instance.new("AlignOrientation")
    FlyAlign.Attachment0        = FlyAtt0
    FlyAlign.Attachment1        = FlyAtt1
    FlyAlign.MaxTorque          = math.huge
    FlyAlign.MaxAngularVelocity = math.huge
    FlyAlign.Responsiveness     = 200
    FlyAlign.RigidityEnabled    = true
    FlyAlign.Parent             = root

    FlyConn = RunService.Heartbeat:Connect(function()
        local r = getRoot()
        if not r then return end

        local dir = Vector3.zero
        local cf  = Camera.CFrame

        if UserInputService:IsKeyDown(Enum.KeyCode.W)         then dir += cf.LookVector  end
        if UserInputService:IsKeyDown(Enum.KeyCode.S)         then dir -= cf.LookVector  end
        if UserInputService:IsKeyDown(Enum.KeyCode.D)         then dir += cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A)         then dir -= cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space)     then dir += Vector3.yAxis  end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir -= Vector3.yAxis  end

        FlyVel.VectorVelocity = (dir.Magnitude > 0 and dir.Unit or Vector3.zero) * FlySpeed

        local lookDir = Vector3.new(cf.LookVector.X, 0, cf.LookVector.Z)
        if lookDir.Magnitude > 0 then
            FlyAtt1.CFrame = CFrame.new(Vector3.zero, lookDir)
        end
    end)
end

local function stopFly()
    if FlyConn  then FlyConn:Disconnect();  FlyConn  = nil end
    if FlyVel   then FlyVel:Destroy();      FlyVel   = nil end
    if FlyAlign then FlyAlign:Destroy();    FlyAlign = nil end
    if FlyAtt0  then FlyAtt0:Destroy();     FlyAtt0  = nil end
    if FlyAtt1  then FlyAtt1:Destroy();     FlyAtt1  = nil end
    local hum = getHumanoid()
    if hum then hum.PlatformStand = false end
end

local function setFly(state)
    FlyEnabled = state
    if state then startFly() else stopFly() end
end

LocalPlayer.CharacterAdded:Connect(function()
    setFly(false)
    task.wait(0.1)
    if Options and Options.FlyToggle then
        Options.FlyToggle:SetValue(false)
    end
end)

-- ========================================
--               MOB ESP
-- ========================================

local ESP_CONFIG = {
    MobFolder   = "Monster",   -- folder in workspace
    MobTag      = "",
    NameColor   = Color3.fromRGB(255, 100, 100),
    HealthColor = Color3.fromRGB(50, 255, 100),
    ShowName    = true,
    ShowHealth  = true,
    ShowDistance = true,
    MaxDistance = 0,
    -- Prefix-based ID filter: any mob whose name starts with these 2-digit prefixes
    -- Covers: 12xxx 14xxx 15xxx 19xxx 20xxx 21xxx (and more)
    IDPrefixes  = {"12", "14", "15", "19", "20", "21"},
    FilterByID  = false,  -- when true only show mobs matching a prefix above
}

local MobESPEnabled = false
local MobESPFolder  = nil

local function mobMatchesIDFilter(obj)
    if not ESP_CONFIG.FilterByID then return true end
    -- Check if the model name is purely numeric and starts with a known prefix
    local name = obj.Name
    for _, prefix in ipairs(ESP_CONFIG.IDPrefixes) do
        if name:sub(1, #prefix) == prefix and tonumber(name) then
            return true
        end
    end
    return false
end

local function getMobs()
    local mobs = {}
    local folder = workspace:FindFirstChild(ESP_CONFIG.MobFolder)
    if folder then
        for _, obj in ipairs(folder:GetDescendants()) do
            if obj:IsA("Model") and obj:FindFirstChildOfClass("Humanoid") and mobMatchesIDFilter(obj) then
                table.insert(mobs, obj)
            end
        end
    end
    if ESP_CONFIG.MobTag ~= "" then
        for _, obj in ipairs(game:GetService("CollectionService"):GetTagged(ESP_CONFIG.MobTag)) do
            if obj:IsA("Model") then table.insert(mobs, obj) end
        end
    end
    return mobs
end

local MobESPUpdateConn = nil  -- live distance updater

local function makeMobGui(mob)
    local root = mob:FindFirstChild("HumanoidRootPart") or mob:FindFirstChildOfClass("BasePart")
    if not root then return end

    local bb = Instance.new("BillboardGui")
    bb.Name        = "MobESP_" .. mob.Name
    bb.Adornee     = root
    bb.AlwaysOnTop = true
    bb.Size        = UDim2.new(0, 160, 0, 70)
    bb.StudsOffset = Vector3.new(0, 3.2, 0)
    bb.MaxDistance = ESP_CONFIG.MaxDistance > 0 and ESP_CONFIG.MaxDistance or math.huge
    bb.Parent      = MobESPFolder

    -- ── Vertical HP bar (left side, thin, with outline) ──
    if ESP_CONFIG.ShowHealth then
        local hum = mob:FindFirstChildOfClass("Humanoid")

        local hpBg = Instance.new("Frame")
        hpBg.Name             = "HPBg"
        hpBg.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
        hpBg.BorderSizePixel  = 0
        hpBg.Size             = UDim2.new(0, 7, 1, 0)
        hpBg.Position         = UDim2.new(0, 0, 0, 0)
        hpBg.Parent           = bb

        local stroke = Instance.new("UIStroke")
        stroke.Color     = Color3.fromRGB(200, 200, 200)
        stroke.Thickness = 0.8
        stroke.Parent    = hpBg

        local fill = Instance.new("Frame")
        fill.Name             = "HPFill"
        fill.AnchorPoint      = Vector2.new(0, 1)
        fill.BorderSizePixel  = 0
        fill.BackgroundColor3 = Color3.fromRGB(0, 210, 60)
        fill.Size             = UDim2.new(1, 0, 1, 0)
        fill.Position         = UDim2.new(0, 0, 1, 0)
        fill.Parent           = hpBg

        if hum then
            local function updateHP()
                local pct = hum.Health / math.max(hum.MaxHealth, 1)
                fill.Size             = UDim2.new(1, 0, pct, 0)
                fill.BackgroundColor3 = Color3.fromRGB(
                    math.floor(255 * (1 - pct)),
                    math.floor(210 * pct),
                    20
                )
            end
            updateHP()
            hum:GetPropertyChangedSignal("Health"):Connect(updateHP)
        end
    end

    -- Content sits to the right of the HP bar
    local cx = 11
    local yPx = 0

    -- ── Name label ──
    if ESP_CONFIG.ShowName then
        local lbl = Instance.new("TextLabel")
        lbl.Name                   = "NameLbl"
        lbl.BackgroundTransparency = 1
        lbl.Size                   = UDim2.new(1, -cx, 0, 24)
        lbl.Position               = UDim2.new(0, cx, 0, yPx)
        lbl.Text                   = mob.Name
        lbl.TextColor3             = Color3.fromRGB(255, 255, 255)
        lbl.TextStrokeTransparency = 0.65
        lbl.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
        lbl.TextScaled             = true
        lbl.Font                   = Enum.Font.FredokaOne
        lbl.TextXAlignment         = Enum.TextXAlignment.Left
        lbl.Parent                 = bb
        yPx += 24
    end

    -- ── Distance label (updated by Heartbeat loop) ──
    if ESP_CONFIG.ShowDistance then
        local lbl = Instance.new("TextLabel")
        lbl.Name                   = "DistLbl"
        lbl.BackgroundTransparency = 1
        lbl.Size                   = UDim2.new(1, -cx, 0, 18)
        lbl.Position               = UDim2.new(0, cx, 0, yPx)
        lbl.Text                   = "0 studs"
        lbl.TextColor3             = Color3.fromRGB(255, 255, 255)
        lbl.TextStrokeTransparency = 0.65
        lbl.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
        lbl.TextScaled             = true
        lbl.Font                   = Enum.Font.GothamMedium
        lbl.TextXAlignment         = Enum.TextXAlignment.Left
        lbl.Parent                 = bb
        yPx += 18
    end

    -- ── HP text label ──
    if ESP_CONFIG.ShowHealth then
        local hum = mob:FindFirstChildOfClass("Humanoid")
        local hpLbl = Instance.new("TextLabel")
        hpLbl.Name                   = "HPLbl"
        hpLbl.BackgroundTransparency = 1
        hpLbl.Size                   = UDim2.new(1, -cx, 0, 16)
        hpLbl.Position               = UDim2.new(0, cx, 0, yPx)
        hpLbl.TextColor3             = Color3.fromRGB(255, 255, 255)
        hpLbl.TextStrokeTransparency = 0.65
        hpLbl.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
        hpLbl.TextScaled             = true
        hpLbl.Font                   = Enum.Font.GothamMedium
        hpLbl.TextXAlignment         = Enum.TextXAlignment.Left
        hpLbl.Parent                 = bb
        if hum then
            local function updateHPLbl()
                hpLbl.Text = math.floor(hum.Health) .. " / " .. math.floor(hum.MaxHealth)
            end
            updateHPLbl()
            hum:GetPropertyChangedSignal("Health"):Connect(updateHPLbl)
        end
    end
end

local function startMobESP()
    MobESPFolder        = Instance.new("Folder")
    MobESPFolder.Name   = "MobESPHolder"
    MobESPFolder.Parent = LocalPlayer.PlayerGui
    for _, mob in ipairs(getMobs()) do makeMobGui(mob) end

    local folder = workspace:FindFirstChild(ESP_CONFIG.MobFolder)
    if folder then
        folder.DescendantAdded:Connect(function(obj)
            if not MobESPEnabled then return end
            if obj:IsA("Model") and obj:FindFirstChildOfClass("Humanoid") then
                task.wait()
                makeMobGui(obj)
            end
        end)
    end

    -- Live distance updater
    MobESPUpdateConn = RunService.Heartbeat:Connect(function()
        if not MobESPEnabled or not ESP_CONFIG.ShowDistance then return end
        local myRoot = getRoot()
        if not myRoot then return end
        if not MobESPFolder then return end
        for _, bb in ipairs(MobESPFolder:GetChildren()) do
            local lbl = bb:FindFirstChild("DistLbl")
            local adornee = bb.Adornee
            if lbl and adornee then
                local dist = math.floor((adornee.Position - myRoot.Position).Magnitude)
                lbl.Text = dist .. " studs"
            end
        end
    end)
end

local function stopMobESP()
    if MobESPUpdateConn then MobESPUpdateConn:Disconnect(); MobESPUpdateConn = nil end
    if MobESPFolder then MobESPFolder:Destroy(); MobESPFolder = nil end
end

-- ========================================
--             PLAYER ESP
-- ========================================

local PlayerESP_CONFIG = {
    ShowName     = true,
    ShowHealth   = true,
    ShowTeam     = true,
    ShowDistance = true,
    MaxDistance  = 0,
    NameColor    = Color3.fromRGB(255, 255, 255),
    DistColor    = Color3.fromRGB(255, 220, 50),
}

local PlayerESPEnabled    = false
local PlayerESPFolder     = nil
local PlayerESPConns      = {}
local PlayerESPUpdateConn = nil

local function makePlayerGui(player)
    if player == LocalPlayer then return end

    local function build()
        local char = player.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end

        local old = PlayerESPFolder and PlayerESPFolder:FindFirstChild("ESP_" .. player.Name)
        if old then old:Destroy() end

        local bb = Instance.new("BillboardGui")
        bb.Name        = "ESP_" .. player.Name
        bb.Adornee     = root
        bb.AlwaysOnTop = true
        bb.Size        = UDim2.new(0, 170, 0, 70)
        bb.StudsOffset = Vector3.new(0, 3.5, 0)
        bb.MaxDistance = PlayerESP_CONFIG.MaxDistance > 0 and PlayerESP_CONFIG.MaxDistance or math.huge
        bb.Parent      = PlayerESPFolder

        -- ── Vertical HP bar — left side, white outline ──
        if PlayerESP_CONFIG.ShowHealth then
            local hum = char:FindFirstChildOfClass("Humanoid")

            local hpBg = Instance.new("Frame")
            hpBg.Name             = "HPBg"
            hpBg.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
            hpBg.BorderSizePixel  = 0
            hpBg.Size             = UDim2.new(0, 7, 1, 0)
            hpBg.Position         = UDim2.new(0, 0, 0, 0)
            hpBg.Parent           = bb

            local stroke = Instance.new("UIStroke")
            stroke.Color     = Color3.fromRGB(255, 255, 255)
            stroke.Thickness  = 0.8
            stroke.Parent    = hpBg

            local fill = Instance.new("Frame")
            fill.Name             = "HPFill"
            fill.AnchorPoint      = Vector2.new(0, 1)
            fill.BorderSizePixel  = 0
            fill.BackgroundColor3 = Color3.fromRGB(0, 210, 60)
            fill.Size             = UDim2.new(1, 0, 1, 0)
            fill.Position         = UDim2.new(0, 0, 1, 0)
            fill.Parent           = hpBg

            if hum then
                local function updateHP()
                    local pct = hum.Health / math.max(hum.MaxHealth, 1)
                    fill.Size             = UDim2.new(1, 0, pct, 0)
                    fill.BackgroundColor3 = Color3.fromRGB(
                        math.floor(255 * (1 - pct)),
                        math.floor(210 * pct),
                        20
                    )
                end
                updateHP()
                hum:GetPropertyChangedSignal("Health"):Connect(updateHP)
            end
        end

        -- Content sits 14px to the right of the HP bar
        local cx = 14
        local yPx = 0

        if PlayerESP_CONFIG.ShowName then
            local lbl = Instance.new("TextLabel")
            lbl.Name                  = "NameLbl"
            lbl.BackgroundTransparency = 1
            lbl.Size                   = UDim2.new(1, -cx, 0, 26)
            lbl.Position               = UDim2.new(0, cx, 0, yPx)
            lbl.Text                   = player.Name
            lbl.TextColor3             = PlayerESP_CONFIG.NameColor
            lbl.TextStrokeTransparency = 0.3
            lbl.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
            lbl.TextScaled             = true
            lbl.Font                   = Enum.Font.FredokaOne
            lbl.TextXAlignment         = Enum.TextXAlignment.Left
            lbl.Parent                 = bb
            yPx += 26
        end

        if PlayerESP_CONFIG.ShowDistance then
            local lbl = Instance.new("TextLabel")
            lbl.Name                  = "DistLbl"
            lbl.BackgroundTransparency = 1
            lbl.Size                   = UDim2.new(1, -cx, 0, 20)
            lbl.Position               = UDim2.new(0, cx, 0, yPx)
            lbl.Text                   = "0 studs"
            lbl.TextColor3             = PlayerESP_CONFIG.DistColor
            lbl.TextStrokeTransparency = 0.4
            lbl.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
            lbl.TextScaled             = true
            lbl.Font                   = Enum.Font.GothamMedium
            lbl.TextXAlignment         = Enum.TextXAlignment.Left
            lbl.Parent                 = bb
            yPx += 20
        end

        if PlayerESP_CONFIG.ShowTeam and player.Team then
            local lbl = Instance.new("TextLabel")
            lbl.BackgroundTransparency = 1
            lbl.Size                   = UDim2.new(1, -cx, 0, 18)
            lbl.Position               = UDim2.new(0, cx, 0, yPx)
            lbl.Text                   = "[" .. player.Team.Name .. "]"
            lbl.TextColor3             = player.Team.TeamColor.Color
            lbl.TextStrokeTransparency = 0.5
            lbl.TextScaled             = true
            lbl.Font                   = Enum.Font.GothamMedium
            lbl.TextXAlignment         = Enum.TextXAlignment.Left
            lbl.Parent                 = bb
        end
    end

    build()

    local conn = player.CharacterAdded:Connect(function()
        task.wait(0.5)
        build()
    end)
    table.insert(PlayerESPConns, conn)
end

local function startPlayerESP()
    PlayerESPFolder        = Instance.new("Folder")
    PlayerESPFolder.Name   = "PlayerESPHolder"
    PlayerESPFolder.Parent = LocalPlayer.PlayerGui

    for _, player in ipairs(Players:GetPlayers()) do
        makePlayerGui(player)
    end

    local addedConn = Players.PlayerAdded:Connect(function(player)
        if not PlayerESPEnabled then return end
        task.wait(1)
        makePlayerGui(player)
    end)
    table.insert(PlayerESPConns, addedConn)

    local removedConn = Players.PlayerRemoving:Connect(function(player)
        local gui = PlayerESPFolder and PlayerESPFolder:FindFirstChild("ESP_" .. player.Name)
        if gui then gui:Destroy() end
    end)
    table.insert(PlayerESPConns, removedConn)

    PlayerESPUpdateConn = RunService.Heartbeat:Connect(function()
        if not PlayerESPEnabled or not PlayerESP_CONFIG.ShowDistance then return end
        local myRoot = getRoot()
        if not myRoot then return end
        for _, player in ipairs(Players:GetPlayers()) do
            if player == LocalPlayer then continue end
            local char = player.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if not root then continue end
            local gui = PlayerESPFolder and PlayerESPFolder:FindFirstChild("ESP_" .. player.Name)
            local lbl = gui and gui:FindFirstChild("DistLbl")
            if lbl then
                lbl.Text = math.floor((root.Position - myRoot.Position).Magnitude) .. " studs"
            end
        end
    end)
end

local function stopPlayerESP()
    if PlayerESPUpdateConn then PlayerESPUpdateConn:Disconnect(); PlayerESPUpdateConn = nil end
    for _, conn in ipairs(PlayerESPConns) do conn:Disconnect() end
    PlayerESPConns = {}
    if PlayerESPFolder then PlayerESPFolder:Destroy(); PlayerESPFolder = nil end
end

-- ========================================
--              LINORIA UI
-- ========================================

local Window = Library:CreateWindow({
    Title    = 'Junior Hub',
    Center   = true,
    AutoShow = true,
})

-- ── Standalone menu keybind ──
-- Uses InputBegan directly so it fires reliably regardless of UI focus state.
local MenuBind = Enum.KeyCode.RightShift
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == MenuBind then
        Library:ToggleUI()
    end
end)

local Tabs = {
    Combat    = Window:AddTab('Combat'),
    Flying    = Window:AddTab('Flying'),
    MobESP    = Window:AddTab('Mob ESP'),
    PlayerESP = Window:AddTab('Player ESP'),
    Teleport  = Window:AddTab('Teleport'),
    Misc      = Window:AddTab('Misc'),
    Settings  = Window:AddTab('Settings'),
}

-- ===== COMBAT TAB =====

local ClickGroup = Tabs.Combat:AddLeftGroupbox('Auto Clicker')

ClickGroup:AddToggle('AutoClickToggle', {
    Text    = 'Enable Auto Clicker',
    Default = false,
    Tooltip = 'Spams M1 at your chosen CPS',
    Callback = function(value) AutoClickEnabled = value end,
}):AddKeyPicker('AutoClickKeybind', {
    Default  = 'F2',
    Mode     = 'Toggle',
    Text     = 'Auto Clicker',
    Callback = function()
        local new = not AutoClickEnabled
        AutoClickEnabled = new
        Options.AutoClickToggle:SetValue(new)
    end,
})

ClickGroup:AddSlider('CPSSlider', {
    Text     = 'Clicks Per Second',
    Default  = 20,
    Min      = 1,
    Max      = 50,
    Rounding = 0,
    Suffix   = ' CPS',
    Callback = function(v) ClickDelay = 1 / v end,
})

-- Auto Quest
local QuestGroup = Tabs.Combat:AddLeftGroupbox('Auto Dwarf King Quest')

QuestGroup:AddToggle('AutoQuestToggle', {
    Text    = 'Enable Auto Dwarf King Quest',
    Default = false,
    Tooltip = 'Runs all 4 NPC steps to accept the Dwarf King quest',
    Callback = function(value)
        AutoQuestEnabled = value
        if value then startAutoQuest() else stopAutoQuest() end
    end,
}):AddKeyPicker('AutoQuestKeybind', {
    Default  = 'None',
    Mode     = 'Toggle',
    Text     = 'Auto Dwarf King Quest',
    Callback = function()
        local new = not AutoQuestEnabled
        AutoQuestEnabled = new
        Options.AutoQuestToggle:SetValue(new)
        if new then startAutoQuest() else stopAutoQuest() end
    end,
})

QuestGroup:AddButton('Take Quest Once', function()
    task.spawn(runQuestSequence)
    Library:Notify('Quest sequence fired!', 2)
end)

QuestGroup:AddLabel('Loop re-takes every 2s.')

-- ===== FLYING TAB =====

local FlyGroup = Tabs.Flying:AddLeftGroupbox('Flight')

-- KEY FIX: keybind is chained via AddKeyPicker (not standalone AddKeybind)
-- The callback only flips state — the toggle Callback handles start/stop
FlyGroup:AddToggle('FlyToggle', {
    Text    = 'Enable Flight',
    Default = false,
    Tooltip = 'Lets your character fly freely',
    Callback = function(value)
        setFly(value)
    end,
}):AddKeyPicker('FlyKeybind', {
    Default  = 'F1',
    Mode     = 'Toggle',
    Text     = 'Flight',
    Callback = function()
        local new = not FlyEnabled
        setFly(new)
        Options.FlyToggle:SetValue(new)
    end,
})

FlyGroup:AddSlider('FlySpeedSlider', {
    Text     = 'Flight Speed',
    Default  = 60,
    Min      = 10,
    Max      = 500,
    Rounding = 0,
    Suffix   = ' studs/s',
    Callback = function(v) FlySpeed = v end,
})

-- ===== MOB ESP TAB =====

local MobESPGroup = Tabs.MobESP:AddLeftGroupbox('Mob ESP')

MobESPGroup:AddToggle('MobESPToggle', {
    Text    = 'Enable Mob ESP',
    Default = false,
    Tooltip = 'Shows name + health above all mobs',
    Callback = function(value)
        MobESPEnabled = value
        if value then startMobESP() else stopMobESP() end
    end,
}):AddKeyPicker('MobESPKeybind', {
    Default  = 'None',
    Mode     = 'Toggle',
    Text     = 'Mob ESP',
    Callback = function()
        local new = not MobESPEnabled
        MobESPEnabled = new
        Options.MobESPToggle:SetValue(new)
        if new then startMobESP() else stopMobESP() end
    end,
})

MobESPGroup:AddToggle('MobShowNames', {
    Text    = 'Show Names',
    Default = true,
    Callback = function(value) ESP_CONFIG.ShowName = value end,
})

MobESPGroup:AddToggle('MobShowHealth', {
    Text    = 'Show Health Bars',
    Default = true,
    Callback = function(value) ESP_CONFIG.ShowHealth = value end,
})

MobESPGroup:AddSlider('MobESPDistance', {
    Text     = 'Max Distance',
    Default  = 0,
    Min      = 0,
    Max      = 2000,
    Rounding = 0,
    Suffix   = ' studs (0 = ∞)',
    Callback = function(value)
        ESP_CONFIG.MaxDistance = value
        if MobESPEnabled then stopMobESP(); startMobESP() end
    end,
})

MobESPGroup:AddToggle('MobShowDistance', {
    Text    = 'Show Distance',
    Default = true,
    Callback = function(value)
        ESP_CONFIG.ShowDistance = value
        if MobESPEnabled then stopMobESP(); startMobESP() end
    end,
})

MobESPGroup:AddToggle('MobFilterByID', {
    Text    = 'Filter by Mob ID',
    Default = false,
    Tooltip = 'Only show mobs matching known ID prefixes: 12xxx 14xxx 15xxx 19xxx 20xxx 21xxx',
    Callback = function(value)
        ESP_CONFIG.FilterByID = value
        if MobESPEnabled then stopMobESP(); startMobESP() end
    end,
})

MobESPGroup:AddInput('MobFolderInput', {
    Text        = 'Mob Folder Name',
    Default     = 'Monster',
    Finished    = true,
    Placeholder = 'e.g. Mobs, Enemies, NPCs',
    Callback    = function(value)
        ESP_CONFIG.MobFolder = value
        if MobESPEnabled then stopMobESP(); startMobESP() end
    end,
})

-- ===== PLAYER ESP TAB =====

local PlayerESPGroup = Tabs.PlayerESP:AddLeftGroupbox('Player ESP')

PlayerESPGroup:AddToggle('PlayerESPToggle', {
    Text    = 'Enable Player ESP',
    Default = false,
    Tooltip = 'Shows name, distance and vertical HP bar above players',
    Callback = function(value)
        PlayerESPEnabled = value
        if value then startPlayerESP() else stopPlayerESP() end
    end,
}):AddKeyPicker('PlayerESPKeybind', {
    Default  = 'None',
    Mode     = 'Toggle',
    Text     = 'Player ESP',
    Callback = function()
        local new = not PlayerESPEnabled
        PlayerESPEnabled = new
        Options.PlayerESPToggle:SetValue(new)
        if new then startPlayerESP() else stopPlayerESP() end
    end,
})

PlayerESPGroup:AddToggle('PlayerShowNames', {
    Text    = 'Show Names',
    Default = true,
    Callback = function(value)
        PlayerESP_CONFIG.ShowName = value
        if PlayerESPEnabled then stopPlayerESP(); startPlayerESP() end
    end,
})

PlayerESPGroup:AddToggle('PlayerShowDistance', {
    Text    = 'Show Distance',
    Default = true,
    Callback = function(value)
        PlayerESP_CONFIG.ShowDistance = value
        if PlayerESPEnabled then stopPlayerESP(); startPlayerESP() end
    end,
})

PlayerESPGroup:AddToggle('PlayerShowHealth', {
    Text    = 'Show Health Bar',
    Default = true,
    Callback = function(value)
        PlayerESP_CONFIG.ShowHealth = value
        if PlayerESPEnabled then stopPlayerESP(); startPlayerESP() end
    end,
})

PlayerESPGroup:AddToggle('PlayerShowTeam', {
    Text    = 'Show Team Name',
    Default = true,
    Callback = function(value)
        PlayerESP_CONFIG.ShowTeam = value
        if PlayerESPEnabled then stopPlayerESP(); startPlayerESP() end
    end,
})

PlayerESPGroup:AddSlider('PlayerESPDistance', {
    Text     = 'Max Distance',
    Default  = 0,
    Min      = 0,
    Max      = 2000,
    Rounding = 0,
    Suffix   = ' studs (0 = ∞)',
    Callback = function(value)
        PlayerESP_CONFIG.MaxDistance = value
        if PlayerESPEnabled then stopPlayerESP(); startPlayerESP() end
    end,
})

-- ========================================
--           NPC TELEPORT
-- ========================================

local NPC_LIST = {
    { label = "Chest TP",  id = "101" },
    { label = "Chest 1",   id = "103" },
    { label = "Chest 2",   id = "104" },
    { label = "Chest 3",   id = "105" },
    { label = "Chest 4",   id = "106" },
    { label = "Chest 5",   id = "107" },
    { label = "Chest 6",   id = "108" },
    { label = "Chest 7",   id = "109" },
    { label = "Chest 8",   id = "110" },
}

-- Search the entire workspace recursively for an NPC with a given ID name.
-- Works for every player since we scan workspace, not a client-only folder.
local function findNPCAnywhere(npcId)
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj.Name == npcId and obj:IsA("Model") then
            return obj
        end
    end
    return nil
end

local function tpToNPC(npcId)
    local root = getRoot()
    if not root then
        Library:Notify("No character!", 2)
        return
    end
    local npc = findNPCAnywhere(npcId)
    if not npc then
        Library:Notify("Chest didn't respawn yet!", 2)
        return
    end
    local npcRoot = npc:FindFirstChild("HumanoidRootPart")
    if not npcRoot then
        for _, v in ipairs(npc:GetDescendants()) do
            if v:IsA("BasePart") then npcRoot = v; break end
        end
    end
    if npcRoot then
        root.CFrame = npcRoot.CFrame + Vector3.new(0, 3, 0)
        Library:Notify("Teleported to chest!", 2)
    else
        Library:Notify("Chest didn't respawn yet!", 2)
    end
end

-- ===== TELEPORT TAB =====

local TPGroup = Tabs.Teleport:AddLeftGroupbox('Chest Teleport')
TPGroup:AddLabel('Teleports you to chests in the world.')
TPGroup:AddLabel('Searches all of workspace automatically.')
TPGroup:AddLabel('')

for _, entry in ipairs(NPC_LIST) do
    local capturedId = entry.id
    TPGroup:AddButton(entry.label, function()
        tpToNPC(capturedId)
    end)
end

local TPRightGroup = Tabs.Teleport:AddRightGroupbox('Custom TP')
TPRightGroup:AddLabel('Enter any NPC ID manually:')
TPRightGroup:AddInput('CustomTPInput', {
    Text        = 'NPC ID',
    Default     = '',
    Finished    = false,
    Placeholder = 'e.g. 108',
    Callback    = function() end,
})
TPRightGroup:AddButton('Teleport', function()
    local id = Options.CustomTPInput.Value
    if id and id ~= '' then
        tpToNPC(id)
    else
        Library:Notify('Enter an NPC ID first!', 2)
    end
end)

-- ===== MISC TAB =====

local MiscFeatGroup = Tabs.Misc:AddLeftGroupbox('Visual Tweaks')

MiscFeatGroup:AddToggle('NoFogToggle', {
    Text    = 'No Fog',
    Default = false,
    Tooltip = 'Removes fog and zeroes out Atmosphere haze/density',
    Callback = function(value)
        NoFogEnabled = value
        if value then applyNoFog() else removeNoFog() end
    end,
}):AddKeyPicker('NoFogKeybind', {
    Default  = 'None',
    Mode     = 'Toggle',
    Text     = 'No Fog',
    Callback = function()
        local new = not NoFogEnabled
        NoFogEnabled = new
        Options.NoFogToggle:SetValue(new)
        if new then applyNoFog() else removeNoFog() end
    end,
})

MiscFeatGroup:AddToggle('NoShadowsToggle', {
    Text    = 'No Shadows',
    Default = false,
    Tooltip = 'Disables GlobalShadows — can boost FPS significantly',
    Callback = function(value)
        if value then removeShadows() else restoreShadows() end
    end,
})

MiscFeatGroup:AddToggle('NoGrassToggle', {
    Text    = 'No Grass / Decorations',
    Default = false,
    Tooltip = 'Sets grass length to 0 and disables terrain decorations',
    Callback = function(value)
        if value then removeGrass() else restoreGrass() end
    end,
})

MiscFeatGroup:AddToggle('NoTexturesToggle', {
    Text    = 'No Textures',
    Default = false,
    Tooltip = 'Sets all part materials to SmoothPlastic and hides Decals/Textures',
    Callback = function(value)
        if value then removeTextures() else restoreTextures() end
    end,
})

MiscFeatGroup:AddToggle('NoPostFXToggle', {
    Text    = 'No Post-Processing FX',
    Default = false,
    Tooltip = 'Disables Bloom, Blur, ColorCorrection, SunRays etc.',
    Callback = function(value)
        if value then removePostFX() else restorePostFX() end
    end,
})

MiscFeatGroup:AddLabel('Tip: combine No Shadows + No Grass')
MiscFeatGroup:AddLabel('for a big FPS boost.')

local BrightGroup = Tabs.Misc:AddRightGroupbox('Lighting')

BrightGroup:AddToggle('FullBrightToggle', {
    Text    = 'Full Bright',
    Default = false,
    Tooltip = 'Sets ambient/brightness to max so you can see everywhere',
    Callback = function(value)
        FullBrightEnabled = value
        if value then applyFullBright() else removeFullBright() end
    end,
})

BrightGroup:AddSlider('TimeSlider', {
    Text     = 'Time of Day',
    Default  = 14,
    Min      = 0,
    Max      = 24,
    Rounding = 1,
    Suffix   = ':00',
    Callback = function(value)
        lockTime(value)
    end,
})

BrightGroup:AddButton('Unlock Time', function()
    unlockTime()
    Library:Notify('Time unlocked.', 2)
end)

BrightGroup:AddLabel('0 = midnight, 12 = noon, 18 = dusk.')

-- (Auto Sprint UI removed)

-- ===== SETTINGS TAB =====

local UIGroup = Tabs.Settings:AddLeftGroupbox('Menu')

UIGroup:AddKeybind('MenuOpenKeybind', {
    Text    = 'Open / Close Menu',
    Default = 'RightShift',
    Mode    = 'Toggle',
    Callback = function()
        -- handled by standalone InputBegan bind to avoid double-toggle
    end,
})

UIGroup:AddKeybind('HideUIKeybind', {
    Text    = 'Hide / Show UI',
    Default = 'Delete',
    Mode    = 'Toggle',
    Callback = function()
        task.defer(function() Library:ToggleUI() end)
    end,
})

UIGroup:AddToggle('MenuKeybindToggle', {
    Text    = 'Show Keybind List',
    Default = true,
    Tooltip = 'Shows/hides the keybind display in the corner',
    Callback = function(value)
        Library.KeybindFrame.Visible = value
    end,
})

UIGroup:AddToggle('TransparentBGToggle', {
    Text    = 'Transparent Background',
    Default = false,
    Tooltip = 'Makes the menu window semi-transparent',
    Callback = function(value)
        pcall(function()
            Library.MainFrame.BackgroundTransparency = value and 0.35 or 0
        end)
    end,
})

local NotifGroup = Tabs.Settings:AddLeftGroupbox('Notifications')

NotifGroup:AddToggle('NotifEnabled', {
    Text    = 'Enable Notifications',
    Default = true,
    Callback = function(value)
        Library.NotificationsEnabled = value
    end,
})

NotifGroup:AddSlider('NotifDuration', {
    Text     = 'Notification Duration',
    Default  = 3,
    Min      = 1,
    Max      = 10,
    Rounding = 0,
    Suffix   = 's',
    Callback = function(value)
        Library.DefaultNotifDuration = value
    end,
})

local PlayerGroup = Tabs.Settings:AddRightGroupbox('Player')

PlayerGroup:AddSlider('WalkSpeedSlider', {
    Text     = 'Walk Speed',
    Default  = 16,
    Min      = 1,
    Max      = 300,
    Rounding = 0,
    Suffix   = ' stud/s',
    Callback = function(value)
        local hum = getHumanoid()
        if hum then hum.WalkSpeed = value end
    end,
})

PlayerGroup:AddSlider('JumpPowerSlider', {
    Text     = 'Jump Power',
    Default  = 50,
    Min      = 0,
    Max      = 300,
    Rounding = 0,
    Suffix   = '',
    Callback = function(value)
        local hum = getHumanoid()
        if hum then
            hum.UseJumpPower = true
            hum.JumpPower    = value
        end
    end,
})

PlayerGroup:AddToggle('InfiniteJumpToggle', {
    Text    = 'Infinite Jump',
    Default = false,
    Tooltip = 'Press Space to jump even while airborne',
    Callback = function(value)
        _G.DaxinInfJump = value
    end,
})

-- One-time connection for infinite jump
UserInputService.JumpRequest:Connect(function()
    if _G.DaxinInfJump then
        local hum = getHumanoid()
        if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end)

local UtilGroup = Tabs.Settings:AddRightGroupbox('Utilities')

UtilGroup:AddButton('Rejoin', function()
    game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
end)

UtilGroup:AddButton('Reset Character', function()
    local hum = getHumanoid()
    if hum then hum.Health = 0 end
end)

UtilGroup:AddButton('Copy UserId', function()
    pcall(function() setclipboard(tostring(LocalPlayer.UserId)) end)
    Library:Notify('UserId: ' .. LocalPlayer.UserId, 3)
end)

UtilGroup:AddButton('Copy Game ID', function()
    pcall(function() setclipboard(tostring(game.PlaceId)) end)
    Library:Notify('Place ID: ' .. game.PlaceId, 3)
end)

UtilGroup:AddButton('Stop All Features', function()
    AutoClickEnabled = false
    NoFogEnabled     = false
    _G.DaxinInfJump  = false
    stopAutoQuest()
    setFly(false)
    stopMobESP()
    stopPlayerESP()
    removeNoFog()
    restoreShadows()
    restoreGrass()
    restoreTextures()
    restorePostFX()
    MobESPEnabled    = false
    PlayerESPEnabled = false
    Options.AutoClickToggle:SetValue(false)
    Options.AutoQuestToggle:SetValue(false)
    Options.FlyToggle:SetValue(false)
    Options.MobESPToggle:SetValue(false)
    Options.PlayerESPToggle:SetValue(false)
    Options.NoFogToggle:SetValue(false)
    Options.NoShadowsToggle:SetValue(false)
    Options.NoGrassToggle:SetValue(false)
    Options.NoTexturesToggle:SetValue(false)
    Options.NoPostFXToggle:SetValue(false)
    Options.InfiniteJumpToggle:SetValue(false)
    Options.FullBrightToggle:SetValue(false)
    removeFullBright()
    unlockTime()
    FullBrightEnabled   = false
    Library:Notify('All features stopped.', 2)
end)

UtilGroup:AddLabel('')
UtilGroup:AddLabel('"Stop All" resets everything.')

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})

ThemeManager:SetFolder('JuniorHub')
SaveManager:SetFolder('JuniorHub/configs')

ThemeManager:ApplyToTab(Tabs.Settings)
SaveManager:ApplyToTab(Tabs.Settings)

SaveManager:LoadAutoloadConfig()

Library:Notify('Junior Hub loaded!', 3)
