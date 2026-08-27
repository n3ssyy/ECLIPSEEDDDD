----------------------------------------------------------------------
--  E C L I P S E D   P R E M I U M   v 4 . 0
--  Ultimate Script for "the HORROR mansion"
--  Kill · ESP · Troll · Utility · Chaos — one script, five tabs
----------------------------------------------------------------------
--  Features (23+):
--    COMBAT : Auto Kill, Kill Special (Limbless etc), Freeze Killers,
--             Killer Spin, Auto-Dodge, God Mode, [Kill ALL Now]
--    ESP    : Player ESP, Killer ESP, Distance Labels, [Scan Killers]
--    TROLL  : Self Spin, Touch Fling, Stalker, Phantom Stalker
--    UTIL   : Fullbright, Noclip, Inf Jump, Speed, Anti-Void, Auto-Respawn
--    CHAOS  : Remote Flood, Physics Anarchy, Sound Chaos, Lag Machine,
--             [Unanchor All], [Destroy Map], [Explode Everything]
----------------------------------------------------------------------

----------------------------------------------------------------------
-- [1] SERVICES
----------------------------------------------------------------------
local CoreGui        = game:GetService("CoreGui")
local Players        = game:GetService("Players")
local RunService     = game:GetService("RunService")
local Workspace      = game:GetService("Workspace")
local UIS            = game:GetService("UserInputService")
local Lighting       = game:GetService("Lighting")
local TweenService   = game:GetService("TweenService")
local Debris         = game:GetService("Debris")

local LP = Players.LocalPlayer

----------------------------------------------------------------------
-- [2] CLEANUP OLD GUI
----------------------------------------------------------------------
local GUI_NAME = "Eclipsed_v4"
pcall(function()
    for _, g in ipairs(CoreGui:GetChildren()) do
        if g.Name == GUI_NAME then g:Destroy() end
    end
    for _, g in ipairs(LP.PlayerGui:GetChildren()) do
        if g.Name == GUI_NAME then g:Destroy() end
    end
end)

----------------------------------------------------------------------
-- [3] STATE & CONFIG
----------------------------------------------------------------------
local toggles = {
    -- Combat
    autoKill      = false,
    killSpecial   = false,
    freezeKillers = false,
    killerSpin    = false,
    autoDodge     = false,
    godMode       = false,
    -- ESP
    espPlayers    = false,
    espKillers    = false,
    espDistance    = false,
    -- Troll
    selfSpin      = false,
    touchFling    = false,
    stalker       = false,
    phantomStalker= false,
    -- Utility
    fullbright    = false,
    noclip        = false,
    infJump       = false,
    speed         = false,
    antiVoid      = false,
    autoRespawn   = false,
    -- Chaos
    remoteFlood   = false,
    physicsAnarchy= false,
    soundChaos    = false,
    lagMachine    = false,
}

local config = {
    killRadius    = 15,
    dodgeRadius   = 10,
    speedMult     = 0.5,
    flingTime     = 0.25,   -- seconds per fling
    flingForce    = 2e6,
}

local selectedTarget   = nil      -- Player target for stalker / phantom
local phantomTriggered = false    -- true when phantom stalker activates
local isFlingBusy      = false    -- fling semaphore
local spawnCFrame      = nil      -- saved spawn position (anti-void)
local dodgeCooldown    = 0        -- os.clock() of last dodge

----------------------------------------------------------------------
-- [4] HELPER FUNCTIONS
----------------------------------------------------------------------
local function getChar()
    return LP.Character
end

local function getRoot()
    local c = getChar()
    return c and (c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso"))
end

local function getHumanoid()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end

-- Find the most usable root part of any model
local function getModelRoot(model)
    local r = model:FindFirstChild("HumanoidRootPart")
    if r then return r end
    r = model:FindFirstChild("Torso")
    if r then return r end
    r = model:FindFirstChild("UpperTorso")
    if r then return r end
    r = model:FindFirstChild("Head")
    if r then return r end
    for _, p in ipairs(model:GetDescendants()) do
        if p:IsA("BasePart") then return p end
    end
    return nil
end

-- Standard killer: non-player model with alive Humanoid
local function isKiller(model)
    if not model:IsA("Model") then return false end
    if model == getChar() then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    local hum = model:FindFirstChildOfClass("Humanoid")
    return hum ~= nil and hum.Health > 0
end

-- Special NPC: looks like a character but has NO Humanoid (Limbless etc.)
local function isSpecialNPC(model)
    if not model:IsA("Model") then return false end
    if model == getChar() then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    if model:FindFirstChildOfClass("Humanoid") then return false end
    -- must have body-like parts
    return model:FindFirstChild("HumanoidRootPart") ~= nil
        or model:FindFirstChild("Torso") ~= nil
        or model:FindFirstChild("Head") ~= nil
        or model:FindFirstChild("UpperTorso") ~= nil
end

-- Any killer/NPC (union of the two above)
local function isAnyNPC(model)
    return isKiller(model) or isSpecialNPC(model)
end

-- Collect all killers currently in workspace
local function getAllKillers()
    local list = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if isAnyNPC(obj) then table.insert(list, obj) end
    end
    return list
end

----------------------------------------------------------------------
-- [5] CORE — MULTI-METHOD KILL + VOID FLING
----------------------------------------------------------------------

-- Quick kill attempt: direct property changes (no teleport)
local function quickKill(model)
    -- Method 1: Humanoid health
    local hum = model:FindFirstChildOfClass("Humanoid")
    if hum then pcall(function() hum.Health = 0 end) end

    -- Method 2: NumberValue / IntValue health fields
    for _, v in ipairs(model:GetDescendants()) do
        if v:IsA("NumberValue") or v:IsA("IntValue") then
            local n = v.Name:lower()
            if n:find("health") or n:find("hp") or n == "health" then
                pcall(function() v.Value = 0 end)
            end
        end
    end

    -- Method 3: Direct downward velocity (works if client has ownership)
    local root = getModelRoot(model)
    if root then
        pcall(function()
            root.Anchored = false
            root.AssemblyLinearVelocity  = Vector3.new(0, -config.flingForce, 0)
            root.AssemblyAngularVelocity = Vector3.new(config.flingForce, config.flingForce, config.flingForce)
        end)
    end
end

-- Full FE void fling: teleport onto target → push under map → return
local function serverVoidFling(targetRoot)
    if isFlingBusy then return false end
    if not targetRoot or not targetRoot.Parent then return false end

    local myRoot = getRoot()
    if not myRoot then return false end

    isFlingBusy = true
    local savedCFrame = myRoot.CFrame

    -- Try quick-kill first
    if targetRoot.Parent:IsA("Model") then
        quickKill(targetRoot.Parent)
    end

    -- Physics fling
    local t0 = os.clock()
    while os.clock() - t0 < config.flingTime do
        RunService.Heartbeat:Wait()
        if not targetRoot or not targetRoot.Parent then break end
        if not myRoot or not myRoot.Parent then break end

        myRoot.CFrame = targetRoot.CFrame * CFrame.new(0, -0.5, 0)
        myRoot.AssemblyLinearVelocity  = Vector3.new(0, -config.flingForce, 0)
        myRoot.AssemblyAngularVelocity = Vector3.new(config.flingForce, config.flingForce, config.flingForce)

        pcall(function()
            targetRoot.AssemblyLinearVelocity  = Vector3.new(0, -config.flingForce, 0)
            targetRoot.AssemblyAngularVelocity = Vector3.new(config.flingForce, config.flingForce, config.flingForce)
        end)
    end

    -- Restore position
    if myRoot and myRoot.Parent then
        myRoot.CFrame = savedCFrame
        myRoot.AssemblyLinearVelocity  = Vector3.zero
        myRoot.AssemblyAngularVelocity = Vector3.zero
    end

    isFlingBusy = false
    return true
end

----------------------------------------------------------------------
-- [6] GUI — SCREENGUI, MAIN FRAME, TABS
----------------------------------------------------------------------

local gui = Instance.new("ScreenGui")
gui.Name = GUI_NAME
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
pcall(function() gui.Parent = CoreGui end)
if not gui.Parent then gui.Parent = LP.PlayerGui end

-- Open button -------------------------------------------------------
local openBtn = Instance.new("TextButton")
openBtn.Size     = UDim2.new(0, 45, 0, 45)
openBtn.Position = UDim2.new(0, 15, 0, 15)
openBtn.BackgroundColor3 = Color3.new(0, 0, 0)
openBtn.BorderColor3     = Color3.new(1, 1, 1)
openBtn.BorderSizePixel  = 2
openBtn.Text        = "E"
openBtn.TextColor3  = Color3.new(1, 1, 1)
openBtn.TextScaled  = true
openBtn.Font        = Enum.Font.Code
openBtn.Parent      = gui

-- Main frame --------------------------------------------------------
local main = Instance.new("Frame")
main.Size     = UDim2.new(0, 330, 0, 470)
main.Position = UDim2.new(0.5, -165, 0.5, -235)
main.BackgroundColor3 = Color3.new(0, 0, 0)
main.BorderColor3     = Color3.new(1, 1, 1)
main.BorderSizePixel  = 2
main.Active    = true
main.Draggable = true
main.Visible   = false
main.Parent    = gui

-- Title bar ---------------------------------------------------------
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.new(1, 1, 1)
titleBar.BorderSizePixel  = 0
titleBar.Parent = main

local titleLbl = Instance.new("TextLabel")
titleLbl.Size = UDim2.new(1, -35, 1, 0)
titleLbl.Position = UDim2.new(0, 10, 0, 0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text       = "E C L I P S E D   v4"
titleLbl.TextColor3 = Color3.new(0, 0, 0)
titleLbl.Font       = Enum.Font.Code
titleLbl.TextSize   = 16
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.Parent = titleBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size     = UDim2.new(0, 30, 0, 30)
closeBtn.Position = UDim2.new(1, -30, 0, 0)
closeBtn.BackgroundColor3 = Color3.new(1, 1, 1)
closeBtn.BorderSizePixel  = 0
closeBtn.Text       = "—"
closeBtn.TextColor3 = Color3.new(0, 0, 0)
closeBtn.Font       = Enum.Font.Code
closeBtn.TextSize   = 20
closeBtn.Parent = titleBar

-- Tab bar -----------------------------------------------------------
local tabBar = Instance.new("Frame")
tabBar.Size     = UDim2.new(1, 0, 0, 26)
tabBar.Position = UDim2.new(0, 0, 0, 30)
tabBar.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
tabBar.BorderSizePixel  = 0
tabBar.Parent = main

-- Status bar --------------------------------------------------------
local statusBar = Instance.new("TextLabel")
statusBar.Size     = UDim2.new(1, 0, 0, 22)
statusBar.Position = UDim2.new(0, 0, 1, -22)
statusBar.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
statusBar.BorderColor3     = Color3.fromRGB(50, 50, 50)
statusBar.BorderSizePixel  = 1
statusBar.Text       = "Ready | Dist: 15 | Target: none"
statusBar.TextColor3 = Color3.fromRGB(120, 120, 120)
statusBar.Font       = Enum.Font.Code
statusBar.TextSize   = 10
statusBar.Parent = main

-- Tab content frames ------------------------------------------------
local TAB_NAMES  = {"COMBAT", "ESP", "TROLL", "UTIL", "CHAOS"}
local tabFrames  = {}
local tabButtons = {}
local activeTab  = "COMBAT"

-- content top = 30 title + 26 tabs = 56
-- content bot = 22 status
-- content h   = 470 - 56 - 22 = 392

for _, name in ipairs(TAB_NAMES) do
    local scroll = Instance.new("ScrollingFrame")
    scroll.Name = name .. "_Tab"
    scroll.Size     = UDim2.new(1, -10, 0, 392)
    scroll.Position = UDim2.new(0, 5, 0, 56)
    scroll.BackgroundColor3       = Color3.new(0, 0, 0)
    scroll.BorderSizePixel        = 0
    scroll.ScrollBarThickness     = 3
    scroll.ScrollBarImageColor3   = Color3.new(1, 1, 1)
    scroll.CanvasSize             = UDim2.new(0, 0, 0, 0)
    scroll.AutomaticCanvasSize    = Enum.AutomaticSize.Y
    scroll.Visible = (name == activeTab)
    scroll.Parent  = main

    local layout = Instance.new("UIListLayout")
    layout.Padding   = UDim.new(0, 4)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent    = scroll

    local pad = Instance.new("UIPadding")
    pad.PaddingTop    = UDim.new(0, 4)
    pad.PaddingBottom = UDim.new(0, 4)
    pad.Parent = scroll

    tabFrames[name] = scroll
end

-- Tab buttons -------------------------------------------------------
for i, name in ipairs(TAB_NAMES) do
    local btn = Instance.new("TextButton")
    btn.Size     = UDim2.new(1 / #TAB_NAMES, 0, 1, 0)
    btn.Position = UDim2.new((i - 1) / #TAB_NAMES, 0, 0, 0)
    btn.BackgroundColor3 = (name == activeTab) and Color3.new(1, 1, 1) or Color3.fromRGB(25, 25, 25)
    btn.BorderSizePixel  = 1
    btn.BorderColor3     = Color3.fromRGB(40, 40, 40)
    btn.Text       = name
    btn.TextColor3 = (name == activeTab) and Color3.new(0, 0, 0) or Color3.fromRGB(130, 130, 130)
    btn.Font       = Enum.Font.Code
    btn.TextSize   = 11
    btn.Parent     = tabBar

    btn.MouseButton1Click:Connect(function()
        activeTab = name
        for n, f in pairs(tabFrames) do f.Visible = (n == name) end
        for n, b in pairs(tabButtons) do
            if n == name then
                b.BackgroundColor3 = Color3.new(1, 1, 1)
                b.TextColor3       = Color3.new(0, 0, 0)
            else
                b.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
                b.TextColor3       = Color3.fromRGB(130, 130, 130)
            end
        end
    end)

    tabButtons[name] = btn
end

-- Open / Close ------------------------------------------------------
openBtn.MouseButton1Click:Connect(function()
    main.Visible    = true
    openBtn.Visible = false
end)

closeBtn.MouseButton1Click:Connect(function()
    main.Visible    = false
    openBtn.Visible = true
end)

----------------------------------------------------------------------
-- [7] NOTIFICATION SYSTEM
----------------------------------------------------------------------
local notifyStack = 0

local function notify(text, duration)
    duration = duration or 3
    notifyStack = notifyStack + 1
    local idx = notifyStack

    local lbl = Instance.new("TextLabel")
    lbl.Size     = UDim2.new(0, 270, 0, 24)
    lbl.Position = UDim2.new(1, -280, 1, -10 - ((idx - 1) % 6) * 28)
    lbl.BackgroundColor3 = Color3.new(0, 0, 0)
    lbl.BorderColor3     = Color3.new(1, 1, 1)
    lbl.BorderSizePixel  = 1
    lbl.Text       = "  " .. text
    lbl.TextColor3 = Color3.new(1, 1, 1)
    lbl.Font       = Enum.Font.Code
    lbl.TextSize   = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = gui

    task.delay(duration, function()
        for i = 1, 10 do
            pcall(function()
                lbl.BackgroundTransparency = i * 0.1
                lbl.TextTransparency       = i * 0.1
            end)
            task.wait(0.03)
        end
        pcall(function() lbl:Destroy() end)
        notifyStack = math.max(0, notifyStack - 1)
    end)
end

----------------------------------------------------------------------
-- [8] UI FACTORIES
----------------------------------------------------------------------
local orderCounters = {}
for _, n in ipairs(TAB_NAMES) do orderCounters[n] = 0 end

-- Section label
local function createLabel(tabName, text)
    orderCounters[tabName] = orderCounters[tabName] + 1
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -8, 0, 18)
    lbl.BackgroundTransparency = 1
    lbl.Text       = "— " .. text .. " —"
    lbl.TextColor3 = Color3.fromRGB(80, 80, 80)
    lbl.Font       = Enum.Font.Code
    lbl.TextSize   = 10
    lbl.LayoutOrder = orderCounters[tabName]
    lbl.Parent = tabFrames[tabName]
end

-- Toggle button
local function createToggle(tabName, text, key, callback)
    orderCounters[tabName] = orderCounters[tabName] + 1

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -8, 0, 30)
    btn.BackgroundColor3 = Color3.new(0, 0, 0)
    btn.BorderColor3     = Color3.fromRGB(50, 50, 50)
    btn.BorderSizePixel  = 1
    btn.Text       = "  " .. text
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font       = Enum.Font.Code
    btn.TextSize   = 12
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.LayoutOrder    = orderCounters[tabName]
    btn.Parent = tabFrames[tabName]

    -- Tiny indicator on the right
    local dot = Instance.new("Frame")
    dot.Size     = UDim2.new(0, 6, 0, 18)
    dot.Position = UDim2.new(1, -16, 0.5, -9)
    dot.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    dot.BorderSizePixel  = 0
    dot.Parent = btn

    local function refresh()
        if toggles[key] then
            btn.BackgroundColor3 = Color3.new(1, 1, 1)
            btn.TextColor3       = Color3.new(0, 0, 0)
            dot.BackgroundColor3 = Color3.fromRGB(0, 220, 0)
        else
            btn.BackgroundColor3 = Color3.new(0, 0, 0)
            btn.TextColor3       = Color3.new(1, 1, 1)
            dot.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        end
    end

    btn.MouseButton1Click:Connect(function()
        toggles[key] = not toggles[key]
        refresh()
        notify(text .. (toggles[key] and "  ON" or "  OFF"), 2)
        if callback then callback(toggles[key]) end
    end)

    return btn, refresh
end

-- Action button (one-shot)
local function createAction(tabName, text, callback)
    orderCounters[tabName] = orderCounters[tabName] + 1

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -8, 0, 30)
    btn.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
    btn.BorderColor3     = Color3.fromRGB(70, 70, 70)
    btn.BorderSizePixel  = 1
    btn.Text       = "  ▶ " .. text
    btn.TextColor3 = Color3.fromRGB(200, 200, 200)
    btn.Font       = Enum.Font.Code
    btn.TextSize   = 12
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.LayoutOrder    = orderCounters[tabName]
    btn.Parent = tabFrames[tabName]

    btn.MouseButton1Click:Connect(function()
        if callback then callback() end
    end)
    return btn
end

-- Numeric input field
local function createInput(tabName, label, default, onSubmit)
    orderCounters[tabName] = orderCounters[tabName] + 1

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 30)
    frame.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
    frame.BorderColor3     = Color3.fromRGB(50, 50, 50)
    frame.BorderSizePixel  = 1
    frame.LayoutOrder      = orderCounters[tabName]
    frame.Parent = tabFrames[tabName]

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.6, 0, 1, 0)
    lbl.Position = UDim2.new(0, 8, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text       = label
    lbl.TextColor3 = Color3.fromRGB(140, 140, 140)
    lbl.Font       = Enum.Font.Code
    lbl.TextSize   = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local box = Instance.new("TextBox")
    box.Size     = UDim2.new(0.32, -4, 0, 20)
    box.Position = UDim2.new(0.66, 0, 0.5, -10)
    box.BackgroundColor3 = Color3.new(0, 0, 0)
    box.BorderColor3     = Color3.new(1, 1, 1)
    box.BorderSizePixel  = 1
    box.Text       = tostring(default)
    box.TextColor3 = Color3.new(1, 1, 1)
    box.Font       = Enum.Font.Code
    box.TextSize   = 11
    box.ClearTextOnFocus = false
    box.Parent = frame

    box.FocusLost:Connect(function()
        local v = tonumber(box.Text)
        if v and onSubmit then
            onSubmit(v)
            box.Text = tostring(v)
            notify(label .. " = " .. tostring(v), 2)
        end
    end)
    return box
end

----------------------------------------------------------------------
-- [9] PLAYER SELECTOR (overlay panel)
----------------------------------------------------------------------
local playerSelector = Instance.new("Frame")
playerSelector.Size     = UDim2.new(0, 210, 0, 320)
playerSelector.Position = UDim2.new(0.5, -105, 0.5, -160)
playerSelector.BackgroundColor3 = Color3.new(0, 0, 0)
playerSelector.BorderColor3     = Color3.new(1, 1, 1)
playerSelector.BorderSizePixel  = 2
playerSelector.Visible  = false
playerSelector.ZIndex   = 20
playerSelector.Active   = true
playerSelector.Draggable = true
playerSelector.Parent   = gui

local psTitle = Instance.new("TextLabel")
psTitle.Size = UDim2.new(1, 0, 0, 24)
psTitle.BackgroundColor3 = Color3.new(1, 1, 1)
psTitle.BorderSizePixel  = 0
psTitle.Text       = " SELECT TARGET"
psTitle.TextColor3 = Color3.new(0, 0, 0)
psTitle.Font       = Enum.Font.Code
psTitle.TextSize   = 12
psTitle.TextXAlignment = Enum.TextXAlignment.Left
psTitle.ZIndex = 20
psTitle.Parent = playerSelector

local psClose = Instance.new("TextButton")
psClose.Size     = UDim2.new(0, 24, 0, 24)
psClose.Position = UDim2.new(1, -24, 0, 0)
psClose.BackgroundTransparency = 1
psClose.Text       = "X"
psClose.TextColor3 = Color3.new(0, 0, 0)
psClose.Font       = Enum.Font.Code
psClose.TextSize   = 13
psClose.ZIndex = 20
psClose.Parent = playerSelector
psClose.MouseButton1Click:Connect(function() playerSelector.Visible = false end)

local psScroll = Instance.new("ScrollingFrame")
psScroll.Size     = UDim2.new(1, -6, 1, -28)
psScroll.Position = UDim2.new(0, 3, 0, 26)
psScroll.BackgroundTransparency = 1
psScroll.BorderSizePixel        = 0
psScroll.ScrollBarThickness     = 2
psScroll.AutomaticCanvasSize    = Enum.AutomaticSize.Y
psScroll.ZIndex = 20
psScroll.Parent = playerSelector

local psLayout = Instance.new("UIListLayout")
psLayout.Padding   = UDim.new(0, 3)
psLayout.SortOrder = Enum.SortOrder.Name
psLayout.Parent    = psScroll

local function openPlayerSelector(callback)
    -- Clear old
    for _, c in ipairs(psScroll:GetChildren()) do
        if c:IsA("TextButton") then c:Destroy() end
    end

    -- "None" option
    local noneBtn = Instance.new("TextButton")
    noneBtn.Size = UDim2.new(1, -4, 0, 26)
    noneBtn.BackgroundColor3 = Color3.fromRGB(30, 0, 0)
    noneBtn.BorderColor3     = Color3.fromRGB(60, 60, 60)
    noneBtn.BorderSizePixel  = 1
    noneBtn.Text       = "  ✕  None (cancel)"
    noneBtn.TextColor3 = Color3.fromRGB(255, 90, 90)
    noneBtn.Font       = Enum.Font.Code
    noneBtn.TextSize   = 11
    noneBtn.TextXAlignment = Enum.TextXAlignment.Left
    noneBtn.ZIndex = 20
    noneBtn.Parent = psScroll
    noneBtn.MouseButton1Click:Connect(function()
        callback(nil)
        playerSelector.Visible = false
    end)

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP then
            local pb = Instance.new("TextButton")
            pb.Size = UDim2.new(1, -4, 0, 26)
            pb.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
            pb.BorderColor3     = Color3.fromRGB(60, 60, 60)
            pb.BorderSizePixel  = 1
            pb.Text       = "  " .. p.DisplayName .. "  (@" .. p.Name .. ")"
            pb.TextColor3 = Color3.new(1, 1, 1)
            pb.Font       = Enum.Font.Code
            pb.TextSize   = 11
            pb.TextXAlignment = Enum.TextXAlignment.Left
            pb.ZIndex = 20
            pb.Parent = psScroll
            pb.MouseButton1Click:Connect(function()
                callback(p)
                playerSelector.Visible = false
            end)
        end
    end
    playerSelector.Visible = true
end

----------------------------------------------------------------------
-- [10] POPULATE TABS
----------------------------------------------------------------------

-- ╔══════════════════════════════════════════════════════════════╗
-- ║  C O M B A T                                                ║
-- ╚══════════════════════════════════════════════════════════════╝
createLabel("COMBAT", "KILL")

createToggle("COMBAT", "Auto Kill (Radius)", "autoKill")

createToggle("COMBAT", "Kill Special (Limbless+)", "killSpecial")

createAction("COMBAT", "Kill ALL Now", function()
    notify("Killing everything...", 3)
    task.spawn(function()
        local targets = getAllKillers()
        for _, k in ipairs(targets) do
            quickKill(k) -- instant pass first
        end
        -- Then fling any survivors
        for _, k in ipairs(targets) do
            if k.Parent then
                local r = getModelRoot(k)
                if r then
                    serverVoidFling(r)
                    task.wait(0.3)
                end
            end
        end
        notify("Kill ALL complete (" .. #targets .. " targets)", 3)
    end)
end)

createLabel("COMBAT", "CONTROL")

createToggle("COMBAT", "Freeze Killers", "freezeKillers", function(state)
    if not state then
        -- Unfreeze everything we anchored
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isAnyNPC(obj) then
                local r = getModelRoot(obj)
                if r then pcall(function() r.Anchored = false end) end
            end
        end
        notify("Killers unfrozen", 2)
    end
end)

createToggle("COMBAT", "Killer Spin", "killerSpin")

createLabel("COMBAT", "DEFENSE")

createToggle("COMBAT", "Auto-Dodge", "autoDodge")
createToggle("COMBAT", "God Mode", "godMode")

createLabel("COMBAT", "SETTINGS")
createInput("COMBAT", "Kill Radius", config.killRadius, function(v) config.killRadius = v end)
createInput("COMBAT", "Dodge Radius", config.dodgeRadius, function(v) config.dodgeRadius = v end)
createInput("COMBAT", "Fling Time (sec)", config.flingTime, function(v) config.flingTime = v end)

-- ╔══════════════════════════════════════════════════════════════╗
-- ║  E S P                                                      ║
-- ╚══════════════════════════════════════════════════════════════╝
createToggle("ESP", "Player ESP (blue)", "espPlayers")
createToggle("ESP", "Killer ESP (red)", "espKillers")
createToggle("ESP", "Distance Labels", "espDistance")

createAction("ESP", "Scan Killers Now", function()
    local names = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if isAnyNPC(obj) then
            table.insert(names, obj.Name)
        end
    end
    if #names > 0 then
        notify("Found " .. #names .. ": " .. table.concat(names, ", "), 8)
    else
        notify("No killers/NPCs detected", 3)
    end
end)

-- ╔══════════════════════════════════════════════════════════════╗
-- ║  T R O L L                                                  ║
-- ╚══════════════════════════════════════════════════════════════╝
createToggle("TROLL", "Self Spin", "selfSpin")
createToggle("TROLL", "Touch Fling Players", "touchFling")

createToggle("TROLL", "Stalker (follow)", "stalker", function(state)
    if state then
        openPlayerSelector(function(p)
            selectedTarget = p
            if p then
                notify("Stalking: " .. p.Name, 3)
            else
                toggles.stalker = false
                notify("Stalker cancelled", 2)
            end
        end)
    else
        -- don't clear selectedTarget if phantom uses it
        if not toggles.phantomStalker then selectedTarget = nil end
    end
end)

createToggle("TROLL", "Phantom Stalker", "phantomStalker", function(state)
    if state then
        phantomTriggered = false
        openPlayerSelector(function(p)
            selectedTarget = p
            if p then
                notify("Phantom: waiting for " .. p.Name .. " to touch you...", 5)
            else
                toggles.phantomStalker = false
                notify("Phantom cancelled", 2)
            end
        end)
    else
        phantomTriggered = false
        if not toggles.stalker then selectedTarget = nil end
    end
end)

-- ╔══════════════════════════════════════════════════════════════╗
-- ║  U T I L I T Y                                              ║
-- ╚══════════════════════════════════════════════════════════════╝
createToggle("UTIL", "Fullbright", "fullbright", function(state)
    if state then
        pcall(function()
            Lighting.Ambient        = Color3.new(1, 1, 1)
            Lighting.OutdoorAmbient = Color3.new(1, 1, 1)
            Lighting.Brightness     = 2
            Lighting.ClockTime      = 14
            Lighting.FogEnd         = 1e6
            Lighting.GlobalShadows  = false
            for _, v in ipairs(Lighting:GetDescendants()) do
                if v:IsA("Atmosphere") or v:IsA("BloomEffect") or v:IsA("BlurEffect")
                   or v:IsA("ColorCorrectionEffect") or v:IsA("DepthOfFieldEffect")
                   or v:IsA("SunRaysEffect") then
                    v.Enabled = false
                end
            end
        end)
    else
        pcall(function()
            Lighting.Ambient        = Color3.fromRGB(70, 70, 70)
            Lighting.OutdoorAmbient = Color3.fromRGB(70, 70, 70)
            Lighting.Brightness     = 1
            Lighting.GlobalShadows  = true
            Lighting.FogEnd         = 100000
            for _, v in ipairs(Lighting:GetDescendants()) do
                if v:IsA("PostEffect") or v:IsA("Atmosphere") then
                    pcall(function() v.Enabled = true end)
                end
            end
        end)
    end
end)

createToggle("UTIL", "Noclip", "noclip")
createToggle("UTIL", "Infinite Jump", "infJump")
createToggle("UTIL", "Speed Boost", "speed")
createToggle("UTIL", "Anti-Void", "antiVoid", function(state)
    if state then
        local r = getRoot()
        if r then
            spawnCFrame = r.CFrame
            notify("Spawn saved at current pos", 2)
        end
    end
end)
createToggle("UTIL", "Auto-Respawn", "autoRespawn")
createInput("UTIL", "Speed Multiplier", config.speedMult, function(v) config.speedMult = v end)

-- ╔══════════════════════════════════════════════════════════════╗
-- ║  C H A O S                                                  ║
-- ╚══════════════════════════════════════════════════════════════╝
createLabel("CHAOS", "CONTINUOUS")
createToggle("CHAOS", "Remote Flood", "remoteFlood")
createToggle("CHAOS", "Physics Anarchy", "physicsAnarchy")
createToggle("CHAOS", "Sound Chaos", "soundChaos")
createToggle("CHAOS", "Lag Machine", "lagMachine")

createLabel("CHAOS", "ONE-SHOT")
createAction("CHAOS", "Unanchor Everything", function()
    notify("Unanchoring...", 2)
    local count = 0
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Anchored then
            pcall(function() obj.Anchored = false; count = count + 1 end)
        end
    end
    notify("Unanchored " .. count .. " parts", 3)
end)

createAction("CHAOS", "Destroy Map Parts", function()
    notify("Destroying map...", 2)
    local count = 0
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            local par = obj.Parent
            if par and not Players:GetPlayerFromCharacter(par) and par ~= getChar() then
                pcall(function() obj:Destroy(); count = count + 1 end)
            end
        end
    end
    notify("Destroyed " .. count .. " parts", 3)
end)

createAction("CHAOS", "Explode Everything", function()
    notify("BOOM!", 2)
    local count = 0
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and math.random() < 0.08 then
            pcall(function()
                local exp = Instance.new("Explosion")
                exp.Position      = obj.Position
                exp.BlastRadius   = 40
                exp.BlastPressure = 800000
                exp.Parent        = Workspace
                count = count + 1
            end)
        end
    end
    notify(count .. " explosions created", 3)
end)

createAction("CHAOS", "Crash Sounds", function()
    notify("Corrupting all audio...", 2)
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("Sound") then
            pcall(function()
                obj.Volume        = 10
                obj.PlaybackSpeed = math.random() * 5
                obj.Looped        = true
                obj:Play()
            end)
        end
    end
    -- Create 50 extra loud sounds
    for i = 1, 50 do
        pcall(function()
            local s = Instance.new("Sound")
            s.SoundId       = "rbxassetid://12222084"  -- default oof
            s.Volume        = 10
            s.PlaybackSpeed = math.random() * 6
            s.Looped        = true
            s.Parent        = Workspace
            s:Play()
        end)
    end
    notify("Audio corrupted", 3)
end)

----------------------------------------------------------------------
-- [11] ESP LOGIC
----------------------------------------------------------------------
local function updateESP()
    -- ===== KILLER ESP =====
    if toggles.espKillers then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isAnyNPC(obj) then
                -- Highlight
                if not obj:FindFirstChild("_KESP") then
                    local hl = Instance.new("Highlight")
                    hl.Name             = "_KESP"
                    hl.FillColor        = Color3.new(1, 0, 0)
                    hl.FillTransparency = 0.5
                    hl.OutlineColor     = Color3.new(1, 1, 1)
                    hl.DepthMode        = Enum.HighlightDepthMode.AlwaysOnTop
                    hl.Parent           = obj
                end

                -- Distance label
                if toggles.espDistance then
                    local myR = getRoot()
                    local eR  = getModelRoot(obj)
                    if myR and eR then
                        local dist = math.floor((myR.Position - eR.Position).Magnitude)
                        local bb = obj:FindFirstChild("_DLabel")
                        if not bb then
                            bb = Instance.new("BillboardGui")
                            bb.Name         = "_DLabel"
                            bb.Size         = UDim2.new(0, 120, 0, 40)
                            bb.StudsOffset  = Vector3.new(0, 3.5, 0)
                            bb.AlwaysOnTop  = true
                            bb.Parent       = obj

                            local tl = Instance.new("TextLabel")
                            tl.Name = "L"
                            tl.Size = UDim2.new(1, 0, 1, 0)
                            tl.BackgroundTransparency = 1
                            tl.TextColor3             = Color3.new(1, 1, 1)
                            tl.TextStrokeTransparency = 0
                            tl.TextStrokeColor3       = Color3.new(0, 0, 0)
                            tl.Font     = Enum.Font.Code
                            tl.TextSize = 14
                            tl.Parent   = bb
                        end
                        local l = bb:FindFirstChild("L")
                        if l then l.Text = obj.Name .. "\n[" .. dist .. " studs]" end
                    end
                end
            end
        end
    else
        for _, obj in ipairs(Workspace:GetDescendants()) do
            pcall(function()
                local e = obj:FindFirstChild("_KESP")
                if e then e:Destroy() end
            end)
        end
    end

    -- Remove distance labels if off
    if not toggles.espDistance or not toggles.espKillers then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            pcall(function()
                local d = obj:FindFirstChild("_DLabel")
                if d then d:Destroy() end
            end)
        end
    end

    -- ===== PLAYER ESP =====
    if toggles.espPlayers then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                if not p.Character:FindFirstChild("_PESP") then
                    local hl = Instance.new("Highlight")
                    hl.Name             = "_PESP"
                    hl.FillColor        = Color3.fromRGB(0, 120, 255)
                    hl.FillTransparency = 0.5
                    hl.OutlineColor     = Color3.new(1, 1, 1)
                    hl.DepthMode        = Enum.HighlightDepthMode.AlwaysOnTop
                    hl.Parent           = p.Character
                end
            end
        end
    else
        for _, p in ipairs(Players:GetPlayers()) do
            pcall(function()
                if p.Character then
                    local e = p.Character:FindFirstChild("_PESP")
                    if e then e:Destroy() end
                end
            end)
        end
    end
end

----------------------------------------------------------------------
-- [12] MAIN HEARTBEAT LOOP
----------------------------------------------------------------------
local tick_ = 0

RunService.Heartbeat:Connect(function()
    tick_ = tick_ + 1

    local myChar = getChar()
    local myRoot = getRoot()
    local myHum  = getHumanoid()

    -- ============================
    --  C O M B A T
    -- ============================

    -- God Mode
    if toggles.godMode and myHum then
        pcall(function() myHum.Health = myHum.MaxHealth end)
    end

    -- Auto Kill — every 3 ticks
    if toggles.autoKill and myRoot and tick_ % 3 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isKiller(obj) then
                local kR = getModelRoot(obj)
                if kR then
                    local dist = (myRoot.Position - kR.Position).Magnitude
                    if dist <= config.killRadius then
                        quickKill(obj) -- instant attempt
                        if not isFlingBusy then
                            task.spawn(serverVoidFling, kR)
                        end
                    end
                end
            end
        end
    end

    -- Kill Special (Limbless etc.) — every 5 ticks
    if toggles.killSpecial and myRoot and tick_ % 5 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isSpecialNPC(obj) then
                local kR = getModelRoot(obj)
                if kR then
                    local dist = (myRoot.Position - kR.Position).Magnitude
                    if dist <= config.killRadius * 2 then
                        quickKill(obj)
                        if not isFlingBusy then
                            task.spawn(serverVoidFling, kR)
                        end
                    end
                end
            end
        end
    end

    -- Freeze Killers — every 5 ticks
    if toggles.freezeKillers and tick_ % 5 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isAnyNPC(obj) then
                local r = getModelRoot(obj)
                if r then pcall(function() r.Anchored = true end) end
            end
        end
    end

    -- Killer Spin — every 2 ticks
    if toggles.killerSpin and tick_ % 2 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isAnyNPC(obj) then
                local r = getModelRoot(obj)
                if r then
                    pcall(function()
                        r.Anchored = false
                        r.AssemblyAngularVelocity = Vector3.new(0, 200, 0)
                    end)
                end
            end
        end
    end

    -- Auto-Dodge (with cooldown)
    if toggles.autoDodge and myRoot and os.clock() > dodgeCooldown then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isAnyNPC(obj) then
                local kR = getModelRoot(obj)
                if kR then
                    local dist = (myRoot.Position - kR.Position).Magnitude
                    if dist < config.dodgeRadius then
                        local dir = (myRoot.Position - kR.Position)
                        if dir.Magnitude > 0.1 then
                            dir = dir.Unit
                        else
                            dir = Vector3.new(1, 0, 0)
                        end
                        myRoot.CFrame = myRoot.CFrame + dir * (config.dodgeRadius * 2)
                        dodgeCooldown = os.clock() + 0.4
                        break -- dodge once per cooldown
                    end
                end
            end
        end
    end

    -- ============================
    --  T R O L L
    -- ============================

    -- Self Spin
    if toggles.selfSpin and myRoot then
        myRoot.AssemblyAngularVelocity = Vector3.new(0, 20000, 0)
    end

    -- Touch Fling Players
    if toggles.touchFling and myRoot and tick_ % 2 == 0 then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local pR = p.Character:FindFirstChild("HumanoidRootPart")
                if pR then
                    local dist = (myRoot.Position - pR.Position).Magnitude
                    if dist <= 8 and not isFlingBusy then
                        task.spawn(serverVoidFling, pR)
                    end
                end
            end
        end
    end

    -- Stalker — follow target instantly
    if toggles.stalker and not toggles.phantomStalker and selectedTarget and myRoot then
        local tc = selectedTarget.Character
        if tc then
            local tR = tc:FindFirstChild("HumanoidRootPart")
            if tR then
                myRoot.CFrame = tR.CFrame * CFrame.new(0, 0, 3)
            end
        end
    end

    -- Phantom Stalker — wait for touch, then follow from behind smoothly
    if toggles.phantomStalker and selectedTarget and myRoot then
        local tc = selectedTarget.Character
        if tc then
            local tR = tc:FindFirstChild("HumanoidRootPart")
            if tR then
                if not phantomTriggered then
                    -- Check "touch" (distance < 5)
                    local dist = (myRoot.Position - tR.Position).Magnitude
                    if dist < 5 then
                        phantomTriggered = true
                        notify("PHANTOM ACTIVATED", 4)
                    end
                else
                    -- Follow from behind with smooth lerp
                    local behindCF = tR.CFrame * CFrame.new(0, 0, 4)
                    myRoot.CFrame = myRoot.CFrame:Lerp(behindCF, 0.25)
                end
            end
        end
    end

    -- ============================
    --  U T I L I T Y
    -- ============================

    -- Speed Boost
    if toggles.speed and myHum and myRoot then
        if myHum.MoveDirection.Magnitude > 0 then
            myRoot.CFrame = myRoot.CFrame + (myHum.MoveDirection * config.speedMult)
        end
    end

    -- Anti-Void
    if toggles.antiVoid and myRoot and spawnCFrame then
        if myRoot.Position.Y < -150 then
            myRoot.CFrame = spawnCFrame
            notify("Anti-Void: saved!", 2)
        end
    end

    -- ============================
    --  E S P  (every 12 ticks)
    -- ============================
    if tick_ % 12 == 0 then
        pcall(updateESP)
    end

    -- ============================
    --  C H A O S
    -- ============================

    -- Physics Anarchy — every 8 ticks
    if toggles.physicsAnarchy and tick_ % 8 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if obj:IsA("BasePart") and not Players:GetPlayerFromCharacter(obj.Parent)
               and obj.Parent ~= myChar then
                pcall(function()
                    if obj.Anchored then obj.Anchored = false end
                    obj.AssemblyLinearVelocity = Vector3.new(
                        math.random(-600, 600),
                        math.random(200, 1000),
                        math.random(-600, 600)
                    )
                end)
            end
        end
    end

    -- Sound Chaos — every 20 ticks
    if toggles.soundChaos and tick_ % 20 == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if obj:IsA("Sound") then
                pcall(function()
                    obj.Volume        = 10
                    obj.PlaybackSpeed = math.random() * 5
                    obj.Looped        = true
                    if not obj.Playing then obj:Play() end
                end)
            end
        end
    end

    -- Lag Machine — every 4 ticks
    if toggles.lagMachine and myRoot and tick_ % 4 == 0 then
        for i = 1, 25 do
            pcall(function()
                local p = Instance.new("Part")
                p.Size     = Vector3.new(
                    math.random(1, 4),
                    math.random(1, 4),
                    math.random(1, 4)
                )
                p.Position = myRoot.Position + Vector3.new(
                    math.random(-25, 25),
                    math.random(5, 40),
                    math.random(-25, 25)
                )
                p.Anchored   = false
                p.CanCollide = true
                p.Material   = Enum.Material.Neon
                p.BrickColor = BrickColor.Random()
                p.Parent     = Workspace
                p.AssemblyLinearVelocity = Vector3.new(
                    math.random(-200, 200),
                    math.random(-200, 200),
                    math.random(-200, 200)
                )
                Debris:AddItem(p, 4)
            end)
        end
    end

    -- ============================
    --  S T A T U S   B A R
    -- ============================
    if tick_ % 30 == 0 then
        local tName = selectedTarget and selectedTarget.Name or "none"
        local pState = phantomTriggered and "ACTIVE" or "wait"
        local extra = ""
        if toggles.phantomStalker then
            extra = " | Phantom: " .. pState
        end
        statusBar.Text = string.format(
            "R: %d | Spd: %.1f | Target: %s%s",
            config.killRadius, config.speedMult, tName, extra
        )
    end
end)

----------------------------------------------------------------------
-- [13] STEPPED LOOP (Noclip)
----------------------------------------------------------------------
RunService.Stepped:Connect(function()
    if toggles.noclip and getChar() then
        for _, part in ipairs(getChar():GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end
end)

----------------------------------------------------------------------
-- [14] INFINITE JUMP
----------------------------------------------------------------------
UIS.JumpRequest:Connect(function()
    if toggles.infJump then
        local hum = getHumanoid()
        if hum then
            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
        end
    end
end)

----------------------------------------------------------------------
-- [15] REMOTE FLOOD (coroutine)
----------------------------------------------------------------------
local remoteFloodRunning = false

task.spawn(function()
    while task.wait(0.5) do
        if toggles.remoteFlood and not remoteFloodRunning then
            remoteFloodRunning = true
            task.spawn(function()
                -- Collect all remotes once
                local remotes = {}
                for _, obj in ipairs(game:GetDescendants()) do
                    if obj:IsA("RemoteEvent") then
                        table.insert(remotes, obj)
                    end
                end
                notify("Remote Flood: " .. #remotes .. " targets found", 3)

                while toggles.remoteFlood do
                    for _, r in ipairs(remotes) do
                        if not toggles.remoteFlood then break end
                        pcall(function() r:FireServer() end)
                        pcall(function() r:FireServer(math.random(1, 1e7)) end)
                        pcall(function() r:FireServer("", true, math.random(), {}) end)
                        pcall(function() r:FireServer(LP, math.huge, nil) end)
                    end
                    task.wait() -- yield to prevent client crash
                end
                remoteFloodRunning = false
                notify("Remote Flood stopped", 2)
            end)
        end
    end
end)

----------------------------------------------------------------------
-- [16] AUTO-RESPAWN
----------------------------------------------------------------------
task.spawn(function()
    while task.wait(0.5) do
        if toggles.autoRespawn then
            local hum = getHumanoid()
            if hum and hum.Health <= 0 then
                task.wait(0.5)
                -- Try multiple respawn methods
                pcall(function() LP:LoadCharacter() end)
                pcall(function()
                    local gui = LP:FindFirstChildOfClass("PlayerGui")
                    if gui then
                        for _, v in ipairs(gui:GetDescendants()) do
                            if v:IsA("TextButton") and v.Text:lower():find("reset") then
                                pcall(function() v:Activate() end)
                            end
                        end
                    end
                end)
            end
        end
    end
end)

----------------------------------------------------------------------
-- [17] PLAYER EVENTS
----------------------------------------------------------------------

-- Handle target leaving
Players.PlayerRemoving:Connect(function(p)
    if p == selectedTarget then
        selectedTarget = nil
        phantomTriggered = false
        notify("Target left the server", 3)
    end
end)

-- Save spawn on character load
LP.CharacterAdded:Connect(function(char)
    task.wait(1)
    local r = char:FindFirstChild("HumanoidRootPart")
    if r then
        spawnCFrame = r.CFrame
    end
end)

-- Save initial spawn
task.defer(function()
    local r = getRoot()
    if r and not spawnCFrame then
        spawnCFrame = r.CFrame
    end
end)

----------------------------------------------------------------------
-- [18] KEYBINDS
----------------------------------------------------------------------
UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    -- RightShift: toggle menu
    if input.KeyCode == Enum.KeyCode.RightShift then
        main.Visible    = not main.Visible
        openBtn.Visible = not main.Visible
    end

    -- F4: emergency stop all toggles
    if input.KeyCode == Enum.KeyCode.F4 then
        for k, _ in pairs(toggles) do
            toggles[k] = false
        end
        phantomTriggered = false
        selectedTarget = nil
        isFlingBusy = false
        notify("EMERGENCY STOP — all OFF", 4)
    end
end)

----------------------------------------------------------------------
-- [19] WELCOME
----------------------------------------------------------------------
notify("E C L I P S E D  v4 loaded", 4)
notify("RightShift = menu | F4 = emergency stop", 4)

print("===========================================")
print("  ECLIPSED PREMIUM v4.0")
print("  23+ features loaded")
print("  Keybinds: RShift (menu), F4 (stop all)")
print("===========================================")
end)
end
