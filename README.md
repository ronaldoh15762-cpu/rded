--[[
    Fear Duels GUI - PC & Mobile Support with Hotkeys
    Quick button layout (4 rows):
        (0,0)=CARRY SPEED, (1,0)=AUTO LEFT, 
        (0,1)=BAT COUNTER, (1,1)=AUTO RIGHT,
        (0,2)=LAGGER MODE, (1,2)=BAT AIMBOT,
        (0,3)=TP DOWN,     (1,3)=DROP BR
    
    HOTKEYS:
    - Z: Toggle TP DOWN (Auto Teleport)
    - X: Toggle DROP BR (Auto Drop)
    - V: Toggle Auto Left
    - B: Toggle Auto Right
    - N: Toggle Bat Counter
    - M: Toggle Bat Aimbot
    - G: Manual Drop (same as DROP BR button)
    - H: Manual TP Down
    - K: Toggle Carry Speed
    - L: Toggle Lagger Mode

--]]
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local Lighting = game:GetService("Lighting")
local LP = Players.LocalPlayer
-- Safe file functions (fixed: pcall always returns true, must check type directly)
local function hasFileFuncs()
    return type(isfile) == "function" and type(readfile) == "function" and type(writefile) == "function"
end
local function safeWriteFile(name, data)
    if not hasFileFuncs() then return end
    local ok, err = pcall(writefile, name, data)
    if not ok then
        warn("[FearDuel] Save failed: " .. tostring(err))
    end
end
local function safeReadFile(name)
    if not hasFileFuncs() then return nil end
    local existsOk, exists = pcall(isfile, name)
    if not existsOk or not exists then return nil end
    local ok, res = pcall(readfile, name)
    if ok then return res end
    return nil
end
pcall(function()
    local pg = LP:FindFirstChild("PlayerGui")
    if pg then
        local old = pg:FindFirstChild("FearDuelGUI")
        if old then old:Destroy() end
    end
end)
-- State
local State = {
    normalSpeed = 60, carrySpeed = 30, laggerSpeed = 60, oxgSpeed = 60, jumpVelocity = 57,
    infJumpEnabled = false, antiRagdollEnabled = false,
    fpsBoostEnabled = false, guiVisible = true,
    isStealing = false, stealStartTime = nil, lastStealTick = 0,
    brainrotReturnEnabled = false, brainrotReturnCooldown = false,
    brainrotReturnSide = nil, lastKnownHealth = 100,
    autoLeftEnabled = false, autoRightEnabled = false,
    _tplnProgress = false, detectedBaseSide = nil,
    lastMoveDir = Vector3.new(0,0,0),
    animEnabled = false,
    mobilesStealing = false,
    _sideDetecting = false,
    carrySpeedActive = false,
    laggerModeEnabled = false,
    oxgModeEnabled = false,

    uiLocked = false,
    quickLocked = false,
    accentTheme = "White", -- default theme name
    batCounterEnabled = true,
    desyncEnabled = false,
    medusaCounterEnabled = false,
    batAimbotEnabled = false,
    batAimbotMeleeOffset = 2,
    tpDownEnabled = false,
    dropBREnabled = false,
    unwalkEnabled = false,
    infJumpMode = "manual",
    fovValue = 110,
    -- Auto TP State
    autoTPEnabled = false,
    autoTPHeight = 20,
    -- GUI Scale
    guiScale = 0.7,
    quickScale = 1.0,
    quickBtnSize = 60,
    moveableMode = false,
    duelLaggerActive = false,
    duelLaggerThread = nil,
    _laggerTableIncrease = 25,
}
local syncInfJumpChips = nil
-- ========== BINDS ==========
local Binds = {
    tpDown        = { key = "Z",  pad = nil },
    dropBR        = { key = "X",  pad = nil },
    autoLeft      = { key = "V",  pad = nil },
    autoRight     = { key = "B",  pad = nil },
    batCounter    = { key = "N",  pad = nil },
    batAimbot     = { key = "M",  pad = nil },
    manualDrop    = { key = "G",  pad = nil },
    manualTpDown  = { key = "H",  pad = nil },
    carrySpeed    = { key = "K",  pad = nil },
    laggerMode    = { key = "L",  pad = "ButtonL3" },
    oxgMode       = { key = "K2", pad = nil },

    duelLagger    = { key = nil,  pad = nil },

    instaGrab     = { key = nil,  pad = nil },
    infJump       = { key = nil,  pad = nil },
    instaReset    = { key = nil,  pad = nil },
}
local BIND_LABELS = {
    tpDown       = "TP Down",       dropBR       = "Drop BR",
    autoLeft     = "Auto Left",     autoRight    = "Auto Right",
    batCounter   = "Bat Counter",   batAimbot    = "Bat Aimbot",
    manualDrop   = "Manual Drop",   manualTpDown = "Manual TP Down",
    carrySpeed   = "Carry Speed",   laggerMode   = "Lagger Mode",
    oxgMode      = "OXG Mode",
    instaGrab    = "Auto Steal",    infJump      = "Inf Jump",
    instaReset   = "Insta Reset",   duelLagger   = "Duel Lagger",
}
local BIND_ORDER = {
    "tpDown","dropBR","autoLeft","autoRight",
    "batCounter","batAimbot","manualDrop","manualTpDown",
    "carrySpeed","laggerMode","oxgMode","instaGrab","infJump","instaReset","duelLagger",
}
local function getBindDisplay(action)
    local b = Binds[action]
    local k = b.key or "—"
    local p = b.pad or "—"
    return k .. " / " .. p
end
local Steal = {
    AutoStealEnabled = false, StealRadius = 60, StealDuration = 1.3,
    Data = {}, plotCache = {}, plotCacheTime = {}, cachedPrompts = {}, promptCacheTime = 0,
}
local CARRY_DETECTION_RADIUS = 17
local PLOT_CACHE_DURATION = 0.5
local PROMPT_CACHE_REFRESH = 0.15
local STEAL_COOLDOWN = 0.1
local medusaDebounce = false
local medusaLastUsed = 0
local medusaConns = {}
local POS = {
    L1 = Vector3.new(-476.48,-6.28,92.73), L2 = Vector3.new(-483.12,-4.95,94.80),
    R1 = Vector3.new(-476.16,-6.52,25.62), R2 = Vector3.new(-483.04,-5.09,23.14),
    RETURN_L = Vector3.new(-475.27,-6.99,94.54),
    RETURN_R = Vector3.new(-475.22,-6.99,23.63),
}
local Conns = { autoSteal = nil, antiRag = nil, autoLeft = nil, autoRight = nil, float = nil,
    progress = nil, heartbeat = nil, batCounter = nil, batAimbot = nil, drop = nil, autoTP = nil }
local h, hrp, speedLbl
local setAutoLeft, setAutoRight
local setInstaGrab, setInfJump, setAntiRag, setFps
local setAnimToggle, setLockUI
local setNormalSpeed, setCarrySpeed, setLaggerSpeed, setGrabRadius, setStealDuration
local startAntiRagdoll, stopAntiRagdoll
local applyFPSBoost, startAutoSteal, stopAutoSteal
local startAutoLeft, stopAutoLeft, startAutoRight, stopAutoRight
local progressFill, progressPct
local setTpDown, setDropBR
local doInstaReset  -- forward ref for keybind
local setOxgMode

local setFovSlider  -- forward ref for FOV slider visual update
local setAutoTP, setAutoTPHeight  -- forward refs for Auto TP
-- ========== AUTO TP FUNCTION ==========
local autoTPConn = nil
local function startAutoTP()
    if autoTPConn then return end
    autoTPConn = RunService.Heartbeat:Connect(function()
        if not State.autoTPEnabled then return end
        if State.batAimbotEnabled then return end
        pcall(function()
            local char = LP.Character
            if not char then return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            local hum = char:FindFirstChildOfClass("Humanoid")
            if not hum then return end
            -- Only TP if falling and above height threshold
            if hum.FloorMaterial == Enum.Material.Air and hrp.Position.Y > State.autoTPHeight then
                local rp = RaycastParams.new()
                rp.FilterDescendantsInstances = { char }
                rp.FilterType = Enum.RaycastFilterType.Exclude
                local result = workspace:Raycast(hrp.Position, Vector3.new(0, -2000, 0), rp)
                if result then
                    local groundY = result.Position.Y
                    local offset = hum.HipHeight + (hrp.Size.Y / 2) + 0.5
                    hrp.CFrame = CFrame.new(hrp.Position.X, groundY + offset, hrp.Position.Z)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                end
            end
        end)
    end)
end
local function stopAutoTP()
    if autoTPConn then
        autoTPConn:Disconnect()
        autoTPConn = nil
    end
end
-- ========== INSTA RESET (top-level, callable from keybind) ==========
local _antiDieConns = {}  -- declared here so doInstaReset can reference it
local _irDebounce = false
local _irTpPos = CFrame.new(1000003.56, 999999.69, 8.17)
local function _irRestoreCamera()
    pcall(function()
        local ch = LP.Character
        local hm = ch and ch:FindFirstChildOfClass("Humanoid")
        workspace.CurrentCamera.CameraType = Enum.CameraType.Custom
        if hm then workspace.CurrentCamera.CameraSubject = hm end
    end)
end
doInstaReset = function()
    if _irDebounce then return end
    local c = LP.Character
    if not c then return end
    local hrp2 = c:FindFirstChild("HumanoidRootPart")
    local hum2 = c:FindFirstChildOfClass("Humanoid")
    if not hrp2 or not hum2 then return end
    _irDebounce = true
    pcall(function() hum2.WalkSpeed = 16 end)
    pcall(function() hrp2.AssemblyLinearVelocity = Vector3.new(0,0,0) end)
    local carpet = c:FindFirstChild("Flying Carpet")
    if carpet then pcall(function() carpet:Destroy() end) end
    pcall(function()
        local cam = workspace.CurrentCamera
        cam.CameraType = Enum.CameraType.Scriptable
        cam.CFrame = cam.CFrame
    end)
    pcall(function()
        for _, part in ipairs(c:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
    -- Temporarily stop anti-die from blocking the kill
    local savedConns = _antiDieConns
    _antiDieConns = {}
    for _, conn in ipairs(savedConns) do pcall(function() conn:Disconnect() end) end
    pcall(function() hum2:SetStateEnabled(Enum.HumanoidStateType.Dead, true) end)
    pcall(function() hum2.Health = 0 end)
    pcall(function() hrp2.CFrame = _irTpPos end)
    pcall(function() c:BreakJoints() end)
    LP.CharacterAdded:Once(function()
        task.wait(0.1); _irRestoreCamera(); _irDebounce = false
    end)
    task.delay(5, function() if _irDebounce then _irDebounce = false; _irRestoreCamera() end end)
end

-- ========== FOV ==========
local fovConnection = nil
local function applyFOV()
    local cam = workspace.CurrentCamera
    if cam then cam.FieldOfView = State.fovValue end
end
local function hookFOV()
    if fovConnection then pcall(function() fovConnection:Disconnect() end) end
    local cam = workspace.CurrentCamera
    if not cam then return end
    applyFOV()
    fovConnection = cam:GetPropertyChangedSignal("FieldOfView"):Connect(function()
        if cam.FieldOfView ~= State.fovValue then
            cam.FieldOfView = State.fovValue
        end
    end)
end
workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(hookFOV)
LP.CharacterAdded:Connect(function()
    task.wait(0.1)
    hookFOV()
end)
hookFOV()
-- ========== TP DOWN FUNCTION (ONE-SHOT) ==========
local _tpDownLastUsed = 0
local _tpDownCooldown = 0.1
local function tpDownNow()
    local now = tick()
    if now - _tpDownLastUsed < _tpDownCooldown then return end

    local char = LP.Character
    if not char then return end

    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    local floorMat = hum.FloorMaterial
    if floorMat ~= Enum.Material.Air then return end

    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = { char }
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude

    local result = workspace:Raycast(root.Position, Vector3.new(0, -2000, 0), raycastParams)
    if not result then return end

    local currentY = root.Position.Y
    local groundY = result.Position.Y
    local offset = hum.HipHeight + (root.Size.Y / 2) + 0.1
    local safeGroundY = groundY + offset

    local distToGround = currentY - safeGroundY
    if distToGround < 1 then return end

    _tpDownLastUsed = now
    root.CFrame = CFrame.new(root.Position.X, safeGroundY, root.Position.Z)
    root.AssemblyLinearVelocity = Vector3.new(root.AssemblyLinearVelocity.X, 0, root.AssemblyLinearVelocity.Z)
end
-- Auto TP DOWN loop
local tpDownConn = nil
local function startTpDown()
    if tpDownConn then return end
    tpDownConn = RunService.Heartbeat:Connect(function()
        if State.tpDownEnabled then
            tpDownNow()
        end
    end)
end
local function stopTpDown()
    if tpDownConn then
        tpDownConn:Disconnect()
        tpDownConn = nil
    end
end
-- ========== DROP BRAINROT FUNCTION ==========
-- ========== DROP BRAINROT FUNCTION ==========
local dropBrainrotActive = false
local function dropBrainrotNow()
    if dropBrainrotActive then return end

    local char = LP.Character
    if not char then return end

    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    dropBrainrotActive = true
    local t0 = tick()
    local DROP_ASCEND_DURATION = 0.2
    local DROP_ASCEND_SPEED = 150
    local dc

    dc = RunService.Heartbeat:Connect(function()
        local r = char and char:FindFirstChild("HumanoidRootPart")
        if not r then
            dc:Disconnect()
            dropBrainrotActive = false
            return
        end

        if tick() - t0 >= DROP_ASCEND_DURATION then
            dc:Disconnect()

            local rp = RaycastParams.new()
            rp.FilterDescendantsInstances = { char }
            rp.FilterType = Enum.RaycastFilterType.Exclude
            local rr = workspace:Raycast(r.Position, Vector3.new(0, -2000, 0), rp)

            if rr then
                local hum = char:FindFirstChildOfClass("Humanoid")
                local off = (hum and hum.HipHeight or 2) + (r.Size.Y / 2)
                r.CFrame = CFrame.new(r.Position.X, rr.Position.Y + off, r.Position.Z)
                r.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
            end

            dropBrainrotActive = false
            return
        end

        r.AssemblyLinearVelocity = Vector3.new(r.AssemblyLinearVelocity.X, DROP_ASCEND_SPEED, r.AssemblyLinearVelocity.Z)
    end)
end
-- Auto DROP BR loop
local dropBRConn = nil
local function startDropBR()
    if dropBRConn then return end
    dropBRConn = RunService.Heartbeat:Connect(function()
        if State.dropBREnabled then
            dropBrainrotNow()
        end
    end)
end
local function stopDropBR()
    if dropBRConn then
        dropBRConn:Disconnect()
        dropBRConn = nil
    end
    dropBrainrotActive = false
end
-- ========== DARK MODE ==========
local nightModeEnabled = false
local defBrightness = Lighting.Brightness
local defClockTime = Lighting.ClockTime
local defOutdoorAmbient = Lighting.OutdoorAmbient
local defExposureComp = Lighting.ExposureCompensation

local function enableDarkMode()
    nightModeEnabled = true
    local sky = Lighting:FindFirstChild("fearDarkSky") or Instance.new("Sky")
    sky.Name = "fearDarkSky"
    sky.SkyboxBk = "rbxassetid://159454299"; sky.SkyboxDn = "rbxassetid://159454296"
    sky.SkyboxFt = "rbxassetid://159454293"; sky.SkyboxLf = "rbxassetid://159454286"
    sky.SkyboxRt = "rbxassetid://159454289"; sky.SkyboxUp = "rbxassetid://159454291"
    sky.Parent = Lighting
    Lighting.Brightness = 0; Lighting.ClockTime = 0
    Lighting.ExposureCompensation = -2
    Lighting.OutdoorAmbient = Color3.fromRGB(0, 0, 0)
end

local function disableDarkMode()
    nightModeEnabled = false
    local s = Lighting:FindFirstChild("fearDarkSky"); if s then s:Destroy() end
    Lighting.Brightness = defBrightness; Lighting.ClockTime = defClockTime
    Lighting.ExposureCompensation = defExposureComp; Lighting.OutdoorAmbient = defOutdoorAmbient
end

-- ========== UNWALK (NO ANIMATIONS) ==========
local _unwalkAnimations = {}
local unwalkConn = nil

local function _disableAnimations()
    local char = LP.Character; if not char then return end
    local hum2 = char:FindFirstChildOfClass("Humanoid"); if not hum2 then return end
    for _, track in pairs(_unwalkAnimations) do pcall(function() track:Stop() end) end
    _unwalkAnimations = {}
    local animator = hum2:FindFirstChildOfClass("Animator")
    if animator then
        for _, track in pairs(animator:GetPlayingAnimationTracks()) do
            track:Stop()
            table.insert(_unwalkAnimations, track)
        end
    end
end

local function startUnwalk()
    _disableAnimations()
    if unwalkConn then unwalkConn:Disconnect() end
    unwalkConn = RunService.Heartbeat:Connect(function()
        if not State.unwalkEnabled then return end
        _disableAnimations()
    end)
end

local function stopUnwalk()
    if unwalkConn then unwalkConn:Disconnect(); unwalkConn = nil end
    _unwalkAnimations = {}
    local c = LP.Character
    if c then
        local anim = c:FindFirstChild("Animate")
        if anim and anim:IsA("LocalScript") and anim.Disabled then
            anim.Disabled = false
        end
    end
end

-- ========== DESYNC ==========
local function applyDesync(state)
    if state then
        if raknet and typeof(raknet.desync) == "function" then
            raknet.desync(true)
        elseif _G.raknet and typeof(_G.raknet.desync) == "function" then
            _G.raknet.desync(true)
        end
    else
        if raknet and typeof(raknet.desync) == "function" then
            raknet.desync(false)
        elseif _G.raknet and typeof(_G.raknet.desync) == "function" then
            _G.raknet.desync(false)
        end
    end
end
local function onCharacterAddedDesync(char)
    if State.desyncEnabled then
        task.wait(0.5)
        applyDesync(true)
    end
end
LP.CharacterAdded:Connect(onCharacterAddedDesync)
if LP.Character then task.spawn(function() onCharacterAddedDesync(LP.Character) end) end
-- ========== MEDUSA ==========
local function findMedusa()
    local char = LP.Character
    if not char then return nil end
    for _, t in ipairs(char:GetChildren()) do
        if t:IsA("Tool") and t.Name:lower():find("medusa") then return t end
    end
    local bp = LP:FindFirstChild("Backpack")
    if bp then
        for _, t in ipairs(bp:GetChildren()) do
            if t:IsA("Tool") and t.Name:lower():find("medusa") then return t end
        end
    end
    return nil
end
local function useMedusa()
    if medusaDebounce or tick() - medusaLastUsed < 25 then return end
    local char = LP.Character
    if not char then return end
    medusaDebounce = true
    local med = findMedusa()
    if med then
        if med.Parent ~= char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then hum:EquipTool(med) end
        end
        pcall(function() med:Activate() end)
        medusaLastUsed = tick()
    end
    medusaDebounce = false
end
local function setupMedusa(char)
    for _, c in pairs(medusaConns) do pcall(function() c:Disconnect() end) end
    medusaConns = {}
    if not char then return end
    local function onAnchor(part)
        return part:GetPropertyChangedSignal("Anchored"):Connect(function()
            if State.medusaCounterEnabled and part.Anchored and part.Transparency == 1 then
                useMedusa()
            end
        end)
    end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then table.insert(medusaConns, onAnchor(part)) end
    end
    table.insert(medusaConns, char.DescendantAdded:Connect(function(part)
        if part:IsA("BasePart") then table.insert(medusaConns, onAnchor(part)) end
    end))
end
local function stopMedusaCounter()
    for _, c in pairs(medusaConns) do pcall(function() c:Disconnect() end) end
    medusaConns = {}
end
local function getHRP()
    local char = LP.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end
local function getHum()
    local char = LP.Character
    return char and char:FindFirstChildOfClass("Humanoid")
end
-- ========== BAT AIMBOT ==========
local AIMBOT_SPEED = 56.5
local function findBat()
    local c = LP.Character
    if not c then return nil end
    local bp = LP:FindFirstChildOfClass("Backpack")
    
    for _, ch in ipairs(c:GetChildren()) do
        if ch:IsA("Tool") and (ch.Name:lower():find("bat") or ch.Name:lower():find("slap")) then
            return ch
        end
    end
    if bp then
        for _, ch in ipairs(bp:GetChildren()) do
            if ch:IsA("Tool") and (ch.Name:lower():find("bat") or ch.Name:lower():find("slap")) then
                return ch
            end
        end
    end
    
    local SlapList = {
        "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap",
        "Emerald Slap", "Ruby Slap", "Dark Matter Slap", "Flame Slap",
        "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
    }
    for _, name in ipairs(SlapList) do
        local t = c:FindFirstChild(name) or (bp and bp:FindFirstChild(name))
        if t then return t end
    end
    return nil
end
local _closestPlayerCache, _closestPlayerCacheTime = nil, 0
local CLOSEST_PLAYER_REFRESH = 0.12
local function getClosestPlayer()
    local now = tick()
    if now - _closestPlayerCacheTime < CLOSEST_PLAYER_REFRESH then return _closestPlayerCache end
    local c = LP.Character
    if not c then _closestPlayerCache = nil; _closestPlayerCacheTime = now; return nil end
    local hrp = c:FindFirstChild("HumanoidRootPart")
    if not hrp then _closestPlayerCache = nil; _closestPlayerCacheTime = now; return nil end
    local closest, bestDist = nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            local tr = p.Character:FindFirstChild("HumanoidRootPart")
            if tr then
                local d = (hrp.Position - tr.Position).Magnitude
                if d < bestDist then
                    bestDist = d
                    closest = p
                end
            end
        end
    end
    _closestPlayerCache = closest
    _closestPlayerCacheTime = now
    return closest
end
-- Detect if player is currently holding a brainrot (any tool that isn't a bat/slap/medusa)
local function isHoldingBrainrot()
    local char = LP.Character
    if not char then return false end
    for _, obj in ipairs(char:GetChildren()) do
        if obj:IsA("Tool") then
            local n = obj.Name:lower()
            if not n:find("bat") and not n:find("slap") and not n:find("medusa") then
                return true
            end
        end
    end
    return false
end
local function dropIfHoldingBrainrot()
    if isHoldingBrainrot() then
        State.dropBREnabled = true
        dropBrainrotNow()
        task.delay(0.3, function()
            State.dropBREnabled = false
            stopDropBR()
        end)
    end
end
-- ========== BAT AIMBOT ==========
local aimbotLockedTarget   = nil
local aimbotNoTargetSince  = nil

local aimbotHighlight = Instance.new("Highlight")
aimbotHighlight.Name             = "AimbotESP"
aimbotHighlight.FillColor        = Color3.fromRGB(255, 0, 0)
aimbotHighlight.OutlineColor     = Color3.fromRGB(255, 255, 255)
aimbotHighlight.FillTransparency = 0.5
aimbotHighlight.OutlineTransparency = 0
pcall(function() aimbotHighlight.Parent = game:GetService("CoreGui") end)
if not aimbotHighlight.Parent then aimbotHighlight.Parent = LP:WaitForChild("PlayerGui") end

local function _aimbotTargetValid(tc)
    if not tc or not tc.Parent then return false end
    local hum = tc:FindFirstChildOfClass("Humanoid")
    local hrp2 = tc:FindFirstChild("HumanoidRootPart")
    return hum and hrp2
end

local function _aimbotGetTarget(myHRP)
    -- Clear locked target if their character was removed (reset)
    if aimbotLockedTarget and not _aimbotTargetValid(aimbotLockedTarget) then
        aimbotLockedTarget = nil
    end
    if aimbotLockedTarget then
        return aimbotLockedTarget:FindFirstChild("HumanoidRootPart"), aimbotLockedTarget
    end
    local best, bestHRP, bestDist = nil, nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and _aimbotTargetValid(p.Character) then
            local tHRP = p.Character:FindFirstChild("HumanoidRootPart")
            local d = (tHRP.Position - myHRP.Position).Magnitude
            if d < bestDist then bestDist = d; bestHRP = tHRP; best = p.Character end
        end
    end
    aimbotLockedTarget = best
    return bestHRP, best
end

-- Anti-die: prevents death during aimbot
local function _activateAntiDie()
    for _, c in ipairs(_antiDieConns) do pcall(function() c:Disconnect() end) end
    _antiDieConns = {}
    local char = LP.Character; if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid"); if not hum then return end
    hum.BreakJointsOnDeath = false
    hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
    table.insert(_antiDieConns, hum:GetPropertyChangedSignal("Health"):Connect(function()
        if hum.Health <= 0 then hum.Health = hum.MaxHealth end
    end))
end

local function startBatAimbot()
    if Conns.batAimbot then
        Conns.batAimbot:Disconnect()
        Conns.batAimbot = nil
    end
    _activateAntiDie()

    Conns.batAimbot = RunService.Heartbeat:Connect(function()
        local char = LP.Character; if not char or not char.Parent then return end
        local myHRP = char:FindFirstChild("HumanoidRootPart"); if not myHRP then return end
        local myH = char:FindFirstChildOfClass("Humanoid"); if not myH or myH.Health <= 0 then return end

        if not State.batAimbotEnabled then
            aimbotHighlight.Adornee = nil
            return
        end

        myH.AutoRotate = false

        -- Auto equip bat
        local bat = findBat()
        if bat and bat.Parent ~= char then pcall(function() myH:EquipTool(bat) end) end

        -- Absolute movement override: forces character into an active unwalked state
        myH:Move(Vector3.new(0, 0, 0), false)

        local tHRP, tChar = _aimbotGetTarget(myHRP)
        if tHRP and tChar and tHRP.Parent and tChar.Parent then
            aimbotNoTargetSince = nil
            aimbotHighlight.Adornee = tChar

            -- Velocity prediction
            local tVel        = tHRP.AssemblyLinearVelocity
            local predictTime = math.clamp(tVel.Magnitude / 150, 0.05, 0.2)
            local predicted   = tHRP.Position + tVel * predictTime

            -- Offsets: behind target + above head (matching OG)
            local behindOffset  = -tHRP.CFrame.LookVector * State.batAimbotMeleeOffset
            local headTopOffset = Vector3.new(0, 2.6, 0)
            local standPos      = predicted + behindOffset + headTopOffset

            local moveDir = standPos - myHRP.Position

            -- CFrame lookAt with -15° pitch (OG style)
            local lookTarget = Vector3.new(predicted.X, myHRP.Position.Y, predicted.Z)
            if (lookTarget - myHRP.Position).Magnitude > 0.1 then
                myHRP.CFrame = CFrame.lookAt(myHRP.Position, lookTarget) * CFrame.Angles(math.rad(-15), 0, 0)
            end

            if moveDir.Magnitude > 1 then
                myHRP.AssemblyLinearVelocity = moveDir.Unit * AIMBOT_SPEED
            else
                myHRP.AssemblyLinearVelocity = tVel
            end
        else
            if not aimbotNoTargetSince then aimbotNoTargetSince = tick() end
            if tick() - aimbotNoTargetSince > 1.5 then
                aimbotLockedTarget = nil
                if myHRP and myHRP.Parent then
                    myHRP.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                end
                aimbotHighlight.Adornee = nil
            end
        end
    end)
end

local function stopBatAimbot()
    if Conns.batAimbot then
        Conns.batAimbot:Disconnect()
        Conns.batAimbot = nil
    end
    local c2 = LP.Character
    local r = c2 and c2:FindFirstChild("HumanoidRootPart")
    local hum2 = c2 and c2:FindFirstChildOfClass("Humanoid")
    if r then
        r.AssemblyLinearVelocity = Vector3.zero
    end
    if hum2 then hum2.AutoRotate = true end
    aimbotLockedTarget  = nil
    aimbotNoTargetSince = nil
    aimbotHighlight.Adornee = nil
end
-- ========== BAT COUNTER ==========
local function findBatCounter()
    local char = LP.Character
    if not char then return nil end
    local bp = LP:FindFirstChildOfClass("Backpack")
    local list = {"Bat","Slap","Iron Slap","Gold Slap","Diamond Slap","Emerald Slap","Ruby Slap","Dark Matter Slap","Flame Slap","Nuclear Slap","Galaxy Slap","Glitched Slap"}
    for _, n in ipairs(list) do
        local t = char:FindFirstChild(n) or (bp and bp:FindFirstChild(n))
        if t then return t end
    end
    for _, ch in ipairs(char:GetChildren()) do
        if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end
    end
    if bp then
        for _, ch in ipairs(bp:GetChildren()) do
            if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end
        end
    end
    return nil
end
local _nearestAttackerCache, _nearestAttackerCacheTime = nil, 0
local function getNearestAttacker()
    local now = tick()
    if now - _nearestAttackerCacheTime < CLOSEST_PLAYER_REFRESH then return _nearestAttackerCache end
    local char = LP.Character
    local myHRP = char and char:FindFirstChild("HumanoidRootPart")
    if not myHRP then _nearestAttackerCache = nil; _nearestAttackerCacheTime = now; return nil end
    local closest, closestDist = nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                local dist = (hrp.Position - myHRP.Position).Magnitude
                if dist < closestDist then
                    closestDist = dist
                    closest = p
                end
            end
        end
    end
    _nearestAttackerCache = closest
    _nearestAttackerCacheTime = now
    return closest
end
-- ========== BAT COUNTER (always on) ==========
Conns.batCounter = RunService.Heartbeat:Connect(function()
    if not State.batCounterEnabled then return end
    local char = LP.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local st = hum:GetState()
    local isRagdolled = st == Enum.HumanoidStateType.Physics or st == Enum.HumanoidStateType.Ragdoll or st == Enum.HumanoidStateType.FallingDown
    if isRagdolled then
        task.spawn(function()
            local root = char:FindFirstChild("HumanoidRootPart")
            local bat = findBatCounter()
            if not bat then return end
            if bat.Parent ~= char then
                local hum2 = char:FindFirstChildOfClass("Humanoid")
                if hum2 then pcall(function() hum2:EquipTool(bat) end) end
            end
            if root then
                local target = getNearestAttacker()
                local tHRP = target and target.Character and target.Character:FindFirstChild("HumanoidRootPart")
                if tHRP then
                    local dir = (tHRP.Position - root.Position).Unit
                    local flatDir = Vector3.new(dir.X, 0, dir.Z)
                    local isShiftLocked = UIS.MouseBehavior == Enum.MouseBehavior.LockCenter
                    if isShiftLocked then
                        local cam = workspace.CurrentCamera
                        local camFlat = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
                        if camFlat.Magnitude > 0 then flatDir = camFlat.Unit end
                    end
                    root.CFrame = CFrame.lookAt(root.Position, root.Position + flatDir)
                    root.AssemblyLinearVelocity = dir * 75
                end
            end
            pcall(function() bat:Activate() end)
            task.wait(0.15)
            pcall(function() bat:Activate() end)
        end)
        task.wait(0.5)
    end
end)
local L1 = Vector3.new(-476.48, -6.28, 92.73)
local L2 = Vector3.new(-483.12, -4.95, 94.80)
local R1 = Vector3.new(-476.16, -6.52, 25.62)
local R2 = Vector3.new(-483.04, -5.09, 23.14)
-- Quick-button frame refs so the auto loops can flash them on waypoint hits
local _autoLeftBtnFrame = nil
local _autoRightBtnFrame = nil
-- Gray color scheme for buttons
local C_BTN_WAYPOINT = Color3.fromRGB(45, 45, 45)
local C_BTN_ON_COLOR  = Color3.fromRGB(100, 100, 100)
local C_BTN_OFF = Color3.fromRGB(40, 40, 40)
local function flashWaypointBtn(frame)
    if not frame or not frame.Parent then return end
    TweenService:Create(frame, TweenInfo.new(0.08), {BackgroundColor3 = C_BTN_WAYPOINT}):Play()
    task.delay(0.18, function()
        if frame and frame.Parent then
            TweenService:Create(frame, TweenInfo.new(0.12), {BackgroundColor3 = C_BTN_ON_COLOR}):Play()
        end
    end)
end
local function turnOffBtn(frame)
    if not frame or not frame.Parent then return end
    TweenService:Create(frame, TweenInfo.new(0.15), {BackgroundColor3 = C_BTN_OFF}):Play()
end
local autoLeftConn = nil
local autoLeftPhase = 1
local function stopAutoLeft()
    if autoLeftConn then
        autoLeftConn:Disconnect()
        autoLeftConn = nil
    end
    autoLeftPhase = 1
    State.autoLeftEnabled = false
    turnOffBtn(_autoLeftBtnFrame)
    local hum = getHum()
    if hum then hum:Move(Vector3.zero, false) end
    local root = getHRP()
    if root then
        root.AssemblyLinearVelocity = Vector3.new(0, root.AssemblyLinearVelocity.Y, 0)
    end
end
local function startAutoLeft()
    dropIfHoldingBrainrot()
    if autoLeftConn then stopAutoLeft() end
    if State.autoRightEnabled then
        State.autoRightEnabled = false
        if setAutoRight then setAutoRight(false) end
        stopAutoRight()
    end
    State.autoLeftEnabled = true
    autoLeftPhase = 1
    autoLeftConn = RunService.Heartbeat:Connect(function()
        if not State.autoLeftEnabled then return end
        local root = getHRP()
        local hum = getHum()
        if not root or not hum then return end
        local spd = State.normalSpeed
        if autoLeftPhase == 1 then
            local d = Vector3.new(L1.X - root.Position.X, 0, L1.Z - root.Position.Z)
            if d.Magnitude < 1 then
                autoLeftPhase = 2
                return
            end
            local md = d.Unit
            hum:Move(md, false)
            root.AssemblyLinearVelocity = Vector3.new(md.X * spd, root.AssemblyLinearVelocity.Y, md.Z * spd)
        elseif autoLeftPhase == 2 then
            local d = Vector3.new(L2.X - root.Position.X, 0, L2.Z - root.Position.Z)
            if d.Magnitude < 1 then
                hum:Move(Vector3.zero, false)
                root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                stopAutoLeft()
                return
            end
            local md = d.Unit
            hum:Move(md, false)
            root.AssemblyLinearVelocity = Vector3.new(md.X * spd, root.AssemblyLinearVelocity.Y, md.Z * spd)
        end
    end)
end
local autoRightConn = nil
local autoRightPhase = 1
local function stopAutoRight()
    if autoRightConn then
        autoRightConn:Disconnect()
        autoRightConn = nil
    end
    autoRightPhase = 1
    State.autoRightEnabled = false
    turnOffBtn(_autoRightBtnFrame)
    local hum = getHum()
    if hum then hum:Move(Vector3.zero, false) end
    local root = getHRP()
    if root then
        root.AssemblyLinearVelocity = Vector3.new(0, root.AssemblyLinearVelocity.Y, 0)
    end
end
local function startAutoRight()
    dropIfHoldingBrainrot()
    if autoRightConn then stopAutoRight() end
    if State.autoLeftEnabled then
        State.autoLeftEnabled = false
        if setAutoLeft then setAutoLeft(false) end
        stopAutoLeft()
    end
    State.autoRightEnabled = true
    autoRightPhase = 1
    autoRightConn = RunService.Heartbeat:Connect(function()
        if not State.autoRightEnabled then return end
        local root = getHRP()
        local hum = getHum()
        if not root or not hum then return end
        local spd = State.normalSpeed
        if autoRightPhase == 1 then
            local d = Vector3.new(R1.X - root.Position.X, 0, R1.Z - root.Position.Z)
            if d.Magnitude < 1 then
                autoRightPhase = 2
                return
            end
            local md = d.Unit
            hum:Move(md, false)
            root.AssemblyLinearVelocity = Vector3.new(md.X * spd, root.AssemblyLinearVelocity.Y, md.Z * spd)
        elseif autoRightPhase == 2 then
            local d = Vector3.new(R2.X - root.Position.X, 0, R2.Z - root.Position.Z)
            if d.Magnitude < 1 then
                hum:Move(Vector3.zero, false)
                root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                stopAutoRight()
                return
            end
            local md = d.Unit
            hum:Move(md, false)
            root.AssemblyLinearVelocity = Vector3.new(md.X * spd, root.AssemblyLinearVelocity.Y, md.Z * spd)
        end
    end)
end
-- Black & White clean theme
local C_BG       = Color3.fromRGB(6, 6, 6)
local C_PANEL    = Color3.fromRGB(12, 12, 12)
local C_ROW      = Color3.fromRGB(18, 18, 18)
local C_BORDER   = Color3.fromRGB(60, 60, 60)
local C_BORDER2  = Color3.fromRGB(40, 40, 40)
local C_HEADER   = Color3.fromRGB(10, 10, 10)
local C_ACCENT   = Color3.fromRGB(255, 255, 255)
local C_ACCENT2  = Color3.fromRGB(210, 210, 210)
local C_DIM      = Color3.fromRGB(90, 90, 90)
local C_WHITE    = Color3.fromRGB(255, 255, 255)
local C_ON_BG    = Color3.fromRGB(255, 255, 255)
local C_OFF_BG   = Color3.fromRGB(18, 18, 18)
local C_KEY_BG   = Color3.fromRGB(16, 16, 16)
local C_TAB_ACT  = Color3.fromRGB(20, 20, 20)
local C_PROGRESS = Color3.fromRGB(200, 200, 200)

-- ========== COLOUR THEMES ==========
local THEMES = {
    { name = "White",   accent = Color3.fromRGB(255,255,255), accent2 = Color3.fromRGB(210,210,210), glow = Color3.fromRGB(180,180,180), btnOn = Color3.fromRGB(255,255,255), btnText = Color3.fromRGB(255,255,255) },
    { name = "Gray",    accent = Color3.fromRGB(180,180,200), accent2 = Color3.fromRGB(210,210,230), glow = Color3.fromRGB(100,100,120), btnOn = Color3.fromRGB(90,90,100),  btnText = Color3.fromRGB(200,200,210) },
    { name = "Purple",  accent = Color3.fromRGB(170,130,255), accent2 = Color3.fromRGB(200,170,255), glow = Color3.fromRGB(130,80,220),  btnOn = Color3.fromRGB(100,60,180), btnText = Color3.fromRGB(200,170,255) },
    { name = "Blue",    accent = Color3.fromRGB(100,160,255), accent2 = Color3.fromRGB(140,195,255), glow = Color3.fromRGB(50,120,220),  btnOn = Color3.fromRGB(40,90,180),  btnText = Color3.fromRGB(140,195,255) },
    { name = "Green",   accent = Color3.fromRGB(100,210,150), accent2 = Color3.fromRGB(140,240,180), glow = Color3.fromRGB(50,170,100),  btnOn = Color3.fromRGB(35,130,75),  btnText = Color3.fromRGB(140,240,180) },
    { name = "Red",     accent = Color3.fromRGB(255,110,110), accent2 = Color3.fromRGB(255,150,150), glow = Color3.fromRGB(200,60,60),   btnOn = Color3.fromRGB(160,45,45),  btnText = Color3.fromRGB(255,150,150) },
    { name = "Pink",    accent = Color3.fromRGB(255,140,200), accent2 = Color3.fromRGB(255,180,220), glow = Color3.fromRGB(210,80,150),  btnOn = Color3.fromRGB(170,55,115), btnText = Color3.fromRGB(255,180,220) },
    { name = "Cyan",    accent = Color3.fromRGB(80,215,230),  accent2 = Color3.fromRGB(120,235,245), glow = Color3.fromRGB(30,170,185),  btnOn = Color3.fromRGB(20,130,145), btnText = Color3.fromRGB(120,235,245) },
}
local function getThemeByName(name)
    for _, t in ipairs(THEMES) do if t.name == name then return t end end
    return THEMES[1]
end
-- Applied at GUI build time; also stored so live-update functions can read it
local _activeTheme = getThemeByName("White")
local function applyTheme(theme)
    _activeTheme = theme
    C_ACCENT    = theme.accent
    C_ACCENT2   = theme.accent2
    C_PROGRESS  = theme.accent
    State.accentTheme = theme.name
end

-- Animations
local Anims = {
    idle1 = "rbxassetid://133806214992291", idle2 = "rbxassetid://94970088341563",
    walk = "rbxassetid://707897309", run = "rbxassetid://707861613",
    jump = "rbxassetid://116936326516985", fall = "rbxassetid://116936326516985",
    climb = "rbxassetid://116936326516985", swim = "rbxassetid://116936326516985",
    swimidle = "rbxassetid://116936326516985",
}
local animHeartbeatConn, originalAnims
local function isPackAnim(id)
    if not id then return false end
    for _, v in pairs(Anims) do if v == id then return true end end
    return false
end
local function saveOriginalAnims(char)
    local anim = char:FindFirstChild("Animate")
    if not anim then return end
    local function g(o) return o and o.AnimationId or nil end
    local ids = {
        idle1 = g(anim.idle and anim.idle.Animation1), idle2 = g(anim.idle and anim.idle.Animation2),
        walk = g(anim.walk and anim.walk.WalkAnim), run = g(anim.run and anim.run.RunAnim),
        jump = g(anim.jump and anim.jump.JumpAnim), fall = g(anim.fall and anim.fall.FallAnim),
        climb = g(anim.climb and anim.climb.ClimbAnim), swim = g(anim.swim and anim.swim.Swim),
        swimidle = g(anim.swimidle and anim.swimidle.SwimIdle),
    }
    if not isPackAnim(ids.walk) then originalAnims = ids end
end
local function applyAnimPack(char)
    local anim = char:FindFirstChild("Animate")
    if not anim then return end
    local function s(o, id) if o then o.AnimationId = id end end
    s(anim.idle and anim.idle.Animation1, Anims.idle1)
    s(anim.idle and anim.idle.Animation2, Anims.idle2)
    s(anim.walk and anim.walk.WalkAnim, Anims.walk)
    s(anim.run and anim.run.RunAnim, Anims.run)
    s(anim.jump and anim.jump.JumpAnim, Anims.jump)
    s(anim.fall and anim.fall.FallAnim, Anims.fall)
    s(anim.climb and anim.climb.ClimbAnim, Anims.climb)
    s(anim.swim and anim.swim.Swim, Anims.swim)
    s(anim.swimidle and anim.swimidle.SwimIdle, Anims.swimidle)
end
local function restoreOriginalAnims(char)
    if not originalAnims then return end
    local anim = char:FindFirstChild("Animate")
    if not anim then return end
    local function s(o, id) if o and id then o.AnimationId = id end end
    s(anim.idle and anim.idle.Animation1, originalAnims.idle1)
    s(anim.idle and anim.idle.Animation2, originalAnims.idle2)
    s(anim.walk and anim.walk.WalkAnim, originalAnims.walk)
    s(anim.run and anim.run.RunAnim, originalAnims.run)
    s(anim.jump and anim.jump.JumpAnim, originalAnims.jump)
    s(anim.fall and anim.fall.FallAnim, originalAnims.fall)
    s(anim.climb and anim.climb.ClimbAnim, originalAnims.climb)
    s(anim.swim and anim.swim.Swim, originalAnims.swim)
    s(anim.swimidle and anim.swimidle.SwimIdle, originalAnims.swimidle)
    local hum2 = char:FindFirstChildOfClass("Humanoid")
    if hum2 then
        for _, t in ipairs(hum2:GetPlayingAnimationTracks()) do t:Stop(0) end
        hum2:ChangeState(Enum.HumanoidStateType.Running)
    end
end
local function startAnimToggle()
    if animHeartbeatConn then animHeartbeatConn:Disconnect(); animHeartbeatConn = nil end
    local char = LP.Character
    if char then
        saveOriginalAnims(char)
        applyAnimPack(char)
        local hum2 = char:FindFirstChildOfClass("Humanoid")
        if hum2 then
            for _, t in ipairs(hum2:GetPlayingAnimationTracks()) do t:Stop(0) end
            hum2:ChangeState(Enum.HumanoidStateType.Running)
        end
    end
    -- Re-apply on character added instead of every frame
end
local function stopAnimToggle()
    if animHeartbeatConn then animHeartbeatConn:Disconnect() end
    State.animEnabled = false
    local char = LP.Character
    if char then restoreOriginalAnims(char) end
end
-- Mobile carry detection
LP:GetAttributeChangedSignal("Stealing"):Connect(function()
    State.mobilesStealing = LP:GetAttribute("Stealing") == true
end)
local function isNearPodiumWithPrompt()
    local char = LP.Character
    local hrpL = char and char:FindFirstChild("HumanoidRootPart")
    if not hrpL then return false end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    for _, plot in ipairs(plots:GetChildren()) do
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if not podiums then continue end
        for _, podium in ipairs(podiums:GetChildren()) do
            if not podium:IsA("Model") then continue end
            local base = podium:FindFirstChild("Base")
            if not base then continue end
            local sp = base:FindFirstChild("Spawn")
            if not sp then continue end
            if (hrpL.Position - podium:GetPivot().Position).Magnitude > CARRY_DETECTION_RADIUS then continue end
            local att = sp:FindFirstChild("PromptAttachment")
            if not att then continue end
            for _, obj in ipairs(att:GetChildren()) do
                if obj:IsA("ProximityPrompt") and obj.Enabled then return true end
            end
        end
    end
    return false
end
local function getMobileCarryState()
    return State.mobilesStealing or isNearPodiumWithPrompt()
end
-- Plot detection
local function getPlotPosition(plot)
    if not plot then return nil end
    if plot.PrimaryPart then return plot.PrimaryPart.Position end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local p = sign:IsA("BasePart") and sign or sign:FindFirstChildWhichIsA("BasePart")
        if p then return p.Position end
    end
    local sum, count = Vector3.new(0,0,0), 0
    for _, obj in plot:GetDescendants() do
        if obj:IsA("BasePart") then
            sum += obj.Position
            count += 1
        end
    end
    return count > 0 and (sum / count) or nil
end
local function findMyPlot()
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local nameLower = (LP.DisplayName or LP.Name):lower()
    for _, plot in plots:GetChildren() do
        local sign = plot:FindFirstChild("PlotSign")
        if not sign then continue end
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") and yb.Enabled then return plot end
        local sg = sign:FindFirstChild("SurfaceGui")
        if not sg then continue end
        local fr = sg:FindFirstChild("Frame")
        if not fr then continue end
        local lbl = fr:FindFirstChild("TextLabel")
        if lbl and typeof(lbl.Text) == "string" and lbl.Text ~= "" then
            local t = lbl.Text:lower()
            if t:find(nameLower, 1, true) and t:find("'s base", 1, true) then return plot end
        end
    end
    return nil
end
local function getSideByPlayerPos()
    local char = LP.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local pos = root.Position
    local dR = (pos - POS.RETURN_R).Magnitude
    local dL = (pos - POS.RETURN_L).Magnitude
    if math.abs(dR - dL) > 10 then return dR < dL and "right" or "left" end
    return nil
end
local function detectBaseSideAsync(callback)
    task.spawn(function()
        local myPlot = nil
        for _ = 1, 30 do
            myPlot = findMyPlot()
            if myPlot then break end
            task.wait(1)
        end
        local side = nil
        if myPlot then
            local bp = getPlotPosition(myPlot)
            if bp then
                side = (bp - POS.RETURN_R).Magnitude < (bp - POS.RETURN_L).Magnitude and "right" or "left"
            end
        end
        if not side then side = getSideByPlayerPos() end
        side = side or "left"
        State.detectedBaseSide = side
        if callback then callback(side) end
    end)
end
local function resetBaseSide() State.detectedBaseSide = nil end
local function doReturnTeleport(targetPos)
    if State.brainrotReturnCooldown then return end
    State.brainrotReturnCooldown = true
    State._tplnProgress = true
    if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft = nil end
    if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight = nil end
    State.autoLeftEnabled = false
    State.autoRightEnabled = false
    if setAutoLeft then setAutoLeft(false) end
    if setAutoRight then setAutoRight(false) end
    task.spawn(function()
        local char = LP.Character
        if char then
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then
                root.CFrame = CFrame.new(targetPos)
                root.AssemblyLinearVelocity = Vector3.zero
                local hum2 = char:FindFirstChildOfClass("Humanoid")
                if hum2 then hum2:Move(Vector3.zero, false) end
            end
        end
        State._tplnProgress = false
        task.wait(0.2)
        State.brainrotReturnCooldown = false
    end)
end
startAntiRagdoll = function()
    if Conns.antiRag then Conns.antiRag:Disconnect(); Conns.antiRag = nil end
    Conns.antiRag = RunService.Heartbeat:Connect(function()
        if not State.antiRagdollEnabled then return end
        local char = LP.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum2 = char:FindFirstChildOfClass("Humanoid")
        if hum2 then
            local st = hum2:GetState()
            if st == Enum.HumanoidStateType.Physics or st == Enum.HumanoidStateType.Ragdoll or st == Enum.HumanoidStateType.FallingDown then
                hum2:ChangeState(Enum.HumanoidStateType.Running)
                workspace.CurrentCamera.CameraSubject = hum2
                pcall(function()
                    local PlayerModule = LP.PlayerScripts:FindFirstChild("PlayerModule")
                    if PlayerModule then
                        local Controls = require(PlayerModule:FindFirstChild("ControlModule"))
                        Controls:Enable()
                    end
                end)
                if root then root.Velocity = Vector3.new(0,0,0); root.RotVelocity = Vector3.new(0,0,0) end
            end
        end
        for _, obj in ipairs(char:GetDescendants()) do
            if obj:IsA("Motor6D") and obj.Enabled == false then obj.Enabled = true end
        end
    end)
end
stopAntiRagdoll = function()
    if Conns.antiRag then Conns.antiRag:Disconnect(); Conns.antiRag = nil end
    State.antiRagdollEnabled = false
end
applyFPSBoost = function()
    local function proc(v)
        pcall(function()
            if v:IsA("Decal") or v:IsA("Texture") then v.Transparency = 1
            elseif v:IsA("Fire") or v:IsA("Smoke") or v:IsA("Sparkles") or v:IsA("ParticleEmitter") then v.Enabled = false
            elseif v:IsA("Attachment") then v.Visible = false
            elseif v:IsA("MeshPart") then v.CastShadow = false; v.DoubleSided = false; v.RenderFidelity = Enum.RenderFidelity.Performance
            elseif v:IsA("BasePart") then v.CastShadow = false; v.Material = Enum.Material.Plastic; v.Reflectance = 0
            end
        end)
    end
    for _, v in pairs(workspace:GetDescendants()) do proc(v) end
    workspace.DescendantAdded:Connect(function(v)
        if State.fpsBoostEnabled then task.spawn(proc, v) end
    end)
end
-- ========== AUTO STEAL ==========
local function isMyPlotByName(plotName)
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(plotName)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then return yb.Enabled == true end
    end
    return false
end
local function findNearestPrompt()
    local char = LP.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local nearest, dist = nil, math.huge
    for _, plot in ipairs(plots:GetChildren()) do
        if isMyPlotByName(plot.Name) then continue end
        local pods = plot:FindFirstChild("AnimalPodiums")
        if not pods then continue end
        for _, pod in ipairs(pods:GetChildren()) do
            local base = pod:FindFirstChild("Base")
            if not base then continue end
            local spawn = base:FindFirstChild("Spawn")
            if not spawn then continue end
            local d = (spawn.Position - root.Position).Magnitude
            if d <= Steal.StealRadius and d < dist then
                local att = spawn:FindFirstChild("PromptAttachment")
                if att then
                    for _, p in ipairs(att:GetChildren()) do
                        if p:IsA("ProximityPrompt") and p.ActionText and p.ActionText:find("Steal") then
                            nearest, dist = p, d
                        end
                    end
                end
            end
        end
    end
    return nearest
end
local function executeSteal(prompt, name)
    if State.isStealing then return end
    if not Steal.Data[prompt] then
        Steal.Data[prompt] = {hold = {}, trigger = {}, ready = true}
        if getconnections then
            for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                if c.Function then table.insert(Steal.Data[prompt].hold, c.Function) end
            end
            for _, c in ipairs(getconnections(prompt.Triggered)) do
                if c.Function then table.insert(Steal.Data[prompt].trigger, c.Function) end
            end
        end
    end
    local data = Steal.Data[prompt]; if not data.ready then return end
    data.ready = false; State.isStealing = true; State.stealStartTime = tick()
    if Conns.progress then Conns.progress:Disconnect() end
    Conns.progress = RunService.Heartbeat:Connect(function()
        if not State.isStealing then Conns.progress:Disconnect(); Conns.progress = nil; return end
        local prog = math.clamp((tick() - State.stealStartTime) / Steal.StealDuration, 0, 1)
        if progressFill then progressFill.Size = UDim2.new(prog, 0, 1, 0) end
        if progressPct then progressPct.Text = math.floor(prog * 100) .. "%" end
    end)
    task.spawn(function()
        for _, fn in ipairs(data.hold) do task.spawn(fn) end
        task.wait(Steal.StealDuration)
        for _, fn in ipairs(data.trigger) do task.spawn(fn) end
        if Conns.progress then Conns.progress:Disconnect(); Conns.progress = nil end
        if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
        if progressPct then progressPct.Text = "0%" end
        data.ready = true; State.isStealing = false
    end)
end
startAutoSteal = function()
    if Conns.autoSteal then return end
    Conns.autoSteal = RunService.Heartbeat:Connect(function()
        if not Steal.AutoStealEnabled or State.isStealing then return end
        local p = findNearestPrompt(); if p then executeSteal(p) end
    end)
end
stopAutoSteal = function()
    if Conns.autoSteal then Conns.autoSteal:Disconnect(); Conns.autoSteal = nil end
    if Conns.progress then Conns.progress:Disconnect(); Conns.progress = nil end
    Steal.AutoStealEnabled = false; State.isStealing = false
    Steal.plotCache = {}; Steal.plotCacheTime = {}; Steal.cachedPrompts = {}
    if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
    if progressPct then progressPct.Text = "0%" end
end
-- Saved positions for moveable mode (persists across rebuilds)
-- _btnSavedPos[idx] = UDim2 absolute position for each button (when individually draggable)
-- _btnSavedPos.containerPos = UDim2 for the whole container (normal mode)
local _btnSavedPos = {}
-- Auto Save
local _autoSaveCooldown = false
local function autoSave()
    if not hasFileFuncs() then return end
    if _autoSaveCooldown then return end
    _autoSaveCooldown = true
    task.delay(0.3, function()
        _autoSaveCooldown = false
        pcall(function()
            local bindsSafe = {}
            for action, b in pairs(Binds) do
                bindsSafe[action] = {
                    key = (type(b.key) == "string") and b.key or "",
                    pad = (type(b.pad) == "string") and b.pad or "",
                }
            end
            local cfg = {
                normalSpeed = State.normalSpeed, carrySpeed = State.carrySpeed, laggerSpeed = State.laggerSpeed, oxgSpeed = State.oxgSpeed,
                autoStealEnabled = Steal.AutoStealEnabled, grabRadius = Steal.StealRadius, stealDuration = Steal.StealDuration,
                infJump = State.infJumpEnabled, infJumpMode = State.infJumpMode, antiRagdoll = State.antiRagdollEnabled,
                fpsBoost = State.fpsBoostEnabled, brainrotReturnEnabled = State.brainrotReturnEnabled,
                animEnabled = State.animEnabled,
                carrySpeedActive = State.carrySpeedActive,
                laggerModeEnabled = State.laggerModeEnabled, oxgModeEnabled = State.oxgModeEnabled, jumpVelocity = State.jumpVelocity, uiLocked = State.uiLocked,
                batCounterEnabled = State.batCounterEnabled,
                desyncEnabled = State.desyncEnabled,
                medusaCounterEnabled = State.medusaCounterEnabled,
                batAimbotEnabled = State.batAimbotEnabled,
                batAimbotAutoDrop = State.batAimbotAutoDrop,
                tpDownEnabled = State.tpDownEnabled,
                dropBREnabled = State.dropBREnabled,
                autoLeftEnabled = State.autoLeftEnabled,
                autoRightEnabled = State.autoRightEnabled,
                fovValue = State.fovValue,
                autoTPEnabled = State.autoTPEnabled,
                autoTPHeight = State.autoTPHeight,
                accentTheme = State.accentTheme,
                quickBtnSize = State.quickBtnSize,
                moveableMode = State.moveableMode,
                quickLocked = State.quickLocked,
                binds = bindsSafe,
                btnSavedPos = (function()
                    local posData = {}
                    -- Save container position
                    if _btnSavedPos.containerPos then
                        posData.containerPos = {
                            xs = _btnSavedPos.containerPos.X.Scale,
                            xo = _btnSavedPos.containerPos.X.Offset,
                            ys = _btnSavedPos.containerPos.Y.Scale,
                            yo = _btnSavedPos.containerPos.Y.Offset,
                        }
                    end
                    -- Save individual button positions (moveable mode)
                    for i = 1, 8 do
                        if _btnSavedPos[i] then
                            posData["btn"..i] = {
                                xs = _btnSavedPos[i].X.Scale,
                                xo = _btnSavedPos[i].X.Offset,
                                ys = _btnSavedPos[i].Y.Scale,
                                yo = _btnSavedPos[i].Y.Offset,
                            }
                        end
                    end
                    return posData
                end)(),
            }
            safeWriteFile("FearDuelConfig.json", HttpService:JSONEncode(cfg))
        end)
    end)
end
local _loadedCfg = nil
local function loadConfigData()
    if not hasFileFuncs() then return end
    local data = safeReadFile("FearDuelConfig.json")
    if not data then return end
    local ok, cfg = pcall(function() return HttpService:JSONDecode(data) end)
    if not ok or not cfg then return end
    _loadedCfg = cfg
    if cfg.normalSpeed and type(cfg.normalSpeed) == "number" then State.normalSpeed = cfg.normalSpeed end
    if cfg.carrySpeed and type(cfg.carrySpeed) == "number" then State.carrySpeed = cfg.carrySpeed end
    if cfg.laggerSpeed and type(cfg.laggerSpeed) == "number" then State.laggerSpeed = cfg.laggerSpeed end
    if cfg.oxgSpeed and type(cfg.oxgSpeed) == "number" then State.oxgSpeed = cfg.oxgSpeed end
    if cfg.jumpVelocity and type(cfg.jumpVelocity) == "number" then State.jumpVelocity = math.clamp(cfg.jumpVelocity, 1, 500) end
    if cfg.infJumpMode == "manual" or cfg.infJumpMode == "hold" then State.infJumpMode = cfg.infJumpMode end
    if syncInfJumpChips then syncInfJumpChips() end

    if cfg.grabRadius and type(cfg.grabRadius) == "number" then Steal.StealRadius = cfg.grabRadius end
    if cfg.stealDuration and type(cfg.stealDuration) == "number" then Steal.StealDuration = math.clamp(cfg.stealDuration, 0.05, 5) end
    if cfg.infJump then State.infJumpEnabled = true end
    if cfg.antiRagdoll then State.antiRagdollEnabled = true end
    if cfg.fpsBoost then State.fpsBoostEnabled = true end
    if cfg.brainrotReturnEnabled then State.brainrotReturnEnabled = true end
    if cfg.animEnabled then State.animEnabled = true end
    if cfg.carrySpeedActive then State.carrySpeedActive = true end
    if cfg.laggerModeEnabled then State.laggerModeEnabled = true end
    if cfg.oxgModeEnabled then State.oxgModeEnabled = true end

    if cfg.uiLocked ~= nil then State.uiLocked = cfg.uiLocked end
    if cfg.batCounterEnabled then State.batCounterEnabled = true end
    if cfg.desyncEnabled then State.desyncEnabled = true end
    if cfg.medusaCounterEnabled then State.medusaCounterEnabled = true end
    if cfg.batAimbotAutoDrop ~= nil then State.batAimbotAutoDrop = cfg.batAimbotAutoDrop end
    if cfg.batAimbotEnabled then
        State.batAimbotEnabled = true
        startBatAimbot()
    end
    if cfg.tpDownEnabled then
        State.tpDownEnabled = true
        startTpDown()
    end
    if cfg.fovValue and type(cfg.fovValue) == "number" then
        State.fovValue = math.clamp(cfg.fovValue, 10, 180)
        applyFOV()
    end
    if cfg.dropBREnabled then
        State.dropBREnabled = true
        startDropBR()
    end
    -- Load Auto TP settings
    if cfg.autoTPEnabled ~= nil then
        State.autoTPEnabled = cfg.autoTPEnabled
        if State.autoTPEnabled then startAutoTP() else stopAutoTP() end
    end
    if cfg.autoTPHeight and type(cfg.autoTPHeight) == "number" then
        State.autoTPHeight = cfg.autoTPHeight
    end
    if cfg.accentTheme and type(cfg.accentTheme) == "string" then
        applyTheme(getThemeByName(cfg.accentTheme))
    end
    if cfg.quickBtnSize and type(cfg.quickBtnSize) == "number" then
        State.quickBtnSize = math.clamp(math.floor(cfg.quickBtnSize), 30, 150)
    end
    if cfg.moveableMode ~= nil then State.moveableMode = cfg.moveableMode end
    if cfg.quickLocked ~= nil then State.quickLocked = cfg.quickLocked end
    -- Load saved button positions (moveable mode)
    if cfg.btnSavedPos and type(cfg.btnSavedPos) == "table" then
        local pd = cfg.btnSavedPos
        if pd.containerPos and type(pd.containerPos) == "table" then
            local xo = pd.containerPos.xo or 0
            local yo = pd.containerPos.yo or 0
            -- Discard if saved position is clearly off-screen (e.g. from old tab layout)
            local screenW = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize.X or 1920
            local screenH = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize.Y or 1080
            if xo > -screenW and xo < screenW and yo > -screenH and yo < screenH then
                _btnSavedPos.containerPos = UDim2.new(
                    pd.containerPos.xs or 1, xo,
                    pd.containerPos.ys or 0.5, yo
                )
            end
            -- else leave nil so rebuildQuickButtons uses the default position
        end
        for i = 1, 8 do
            local key = "btn"..i
            if pd[key] and type(pd[key]) == "table" then
                _btnSavedPos[i] = UDim2.new(
                    pd[key].xs or 1, pd[key].xo or 0,
                    pd[key].ys or 0.5, pd[key].yo or 0
                )
            end
        end
    end
    -- Load binds (fixed: validate types, guard nulls, and strip reserved gamepad buttons)
    if cfg.binds and type(cfg.binds) == "table" then
        for action, v in pairs(cfg.binds) do
            if Binds[action] and type(v) == "table" then
                local k = v.key
                local p = v.pad
                local savedKey = (type(k) == "string" and k ~= "" and k ~= "null") and k or nil
                local padVal = (type(p) == "string" and p ~= "" and p ~= "null") and p or nil
                -- Only overwrite default if a real saved value exists
                if savedKey ~= nil then Binds[action].key = savedKey end
                if padVal ~= nil then
                    local reservedPadButtons = { ButtonA=true, ButtonSelect=true, ButtonStart=true, ButtonR3=true }
                    Binds[action].pad = (not reservedPadButtons[padVal]) and padVal or nil
                end
            end
        end
    end
end
-- ========== QUICK ACTION BUTTONS ==========
local quickPanel
local quickBtnFrames_global = {} -- track individual button panels for cleanup
local function rebuildQuickButtons()
    local QW, QH, QG = State.quickBtnSize, State.quickBtnSize, 8
    local _btnCorner = math.max(8, math.floor(QW * 0.21))
    local _btnTextSize = math.max(9, math.floor(QW * 0.168))
    local C_BTN = Color3.fromRGB(0, 0, 0)
    local C_BTN_ON = Color3.fromRGB(255, 255, 255)
    local C_BTN_TEXT = Color3.fromRGB(255, 255, 255)

    local gui = LP.PlayerGui:FindFirstChild("FearDuelGUI")
    if not gui then return end

    -- Fade out and destroy old container
    local oldContainer = gui:FindFirstChild("QuickBtnContainer")
    if oldContainer then
        TweenService:Create(oldContainer, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {BackgroundTransparency = 1}):Play()
        for _, child in ipairs(oldContainer:GetDescendants()) do
            if child:IsA("Frame") or child:IsA("TextButton") or child:IsA("TextLabel") then
                pcall(function()
                    TweenService:Create(child, TweenInfo.new(0.18), {BackgroundTransparency = 1}):Play()
                    if child:IsA("TextButton") or child:IsA("TextLabel") then
                        TweenService:Create(child, TweenInfo.new(0.18), {TextTransparency = 1}):Play()
                    end
                end)
            end
        end
        task.delay(0.2, function() pcall(function() oldContainer:Destroy() end) end)
    end

    -- Clean up old tracked frames
    for _, f in ipairs(quickBtnFrames_global) do
        pcall(function() f:Destroy() end)
    end
    quickBtnFrames_global = {}
    if quickPanel then quickPanel:Destroy(); quickPanel = nil end
    local oldBg = gui:FindFirstChild("QuickActionsBg")
    if oldBg then oldBg:Destroy() end

    local CONT_PAD = 10
    local CONT_W = QW * 2 + QG + CONT_PAD * 2
    local CONT_H = QH * 4 + QG * 3 + CONT_PAD * 2

    -- Saved container position persists across rebuilds
    if not _btnSavedPos.containerPos then
        _btnSavedPos.containerPos = UDim2.new(1, -(CONT_W + 60), 0.5, -CONT_H / 2)
    end

    local btnContainer = Instance.new("Frame", gui)
    btnContainer.Name = "QuickBtnContainer"
    btnContainer.Size = UDim2.new(0, CONT_W, 0, CONT_H)
    btnContainer.Position = _btnSavedPos.containerPos
    btnContainer.BorderSizePixel = 0
    btnContainer.ZIndex = 25
    btnContainer.Active = true  -- Enabled for dragging
    btnContainer.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
    btnContainer.BackgroundTransparency = 0.6
    Instance.new("UICorner", btnContainer).CornerRadius = UDim.new(0, 10)
    local btnContStroke = Instance.new("UIStroke", btnContainer)
    btnContStroke.Color = Color3.fromRGB(55, 55, 65)
    btnContStroke.Thickness = 1.2

    if State.moveableMode then
        btnContainer.BackgroundTransparency = 1
        btnContStroke.Transparency = 1
    else
        btnContainer.BackgroundTransparency = 0.6
        btnContStroke.Transparency = 0
    end

    -- Background image on quick buttons panel (subtle)
    local qbBgImg = Instance.new("ImageLabel", btnContainer)
    qbBgImg.Name = "QBBackgroundImage"
    qbBgImg.Size = UDim2.new(1, 0, 1, 0)
    qbBgImg.Position = UDim2.new(0, 0, 0, 0)
    qbBgImg.BackgroundTransparency = 1
    qbBgImg.Image = "rbxassetid://112977078041259"
    qbBgImg.ScaleType = Enum.ScaleType.Crop
    qbBgImg.ImageTransparency = State.moveableMode and 1 or 0.35
    qbBgImg.ZIndex = 24
    Instance.new("UICorner", qbBgImg).CornerRadius = UDim.new(0, 10)

    -- ── DRAG directly on the container (no visible handle or lock button) ───
    local LOCK_H = 0
    local qbLocked = State.quickLocked

    local function setQbLocked(locked)
        qbLocked = locked
    end

    local _qbDragging, _qbDragStart, _qbStartPos = false, nil, nil
    btnContainer.InputBegan:Connect(function(inp)
        if qbLocked then return end
        if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then
            _qbDragging = true
            _qbDragStart = inp.Position
            _qbStartPos = btnContainer.Position
            inp.Changed:Connect(function()
                if inp.UserInputState == Enum.UserInputState.End then _qbDragging = false end
            end)
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not _qbDragging then return end
        if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseMovement then
            local dx = inp.Position.X - _qbDragStart.X
            local dy = inp.Position.Y - _qbDragStart.Y
            local newPos = UDim2.new(
                _qbStartPos.X.Scale, _qbStartPos.X.Offset + dx,
                _qbStartPos.Y.Scale, _qbStartPos.Y.Offset + dy
            )
            btnContainer.Position = newPos
            _btnSavedPos.containerPos = newPos
            autoSave()
        end
    end)

    local function makeDraggableButton(label, idx)
        local col = (idx - 1) % 2
        local row = math.floor((idx - 1) / 2)
        local defaultOffX = CONT_PAD + col * (QW + QG)
        local defaultOffY = CONT_PAD + row * (QH + QG)

        -- In moveableMode each button is parented directly to the ScreenGui so it
        -- can be positioned anywhere on screen independently.
        local parent = State.moveableMode and gui or btnContainer
        local f = Instance.new("Frame", parent)
        f.Name = "QuickBtn_" .. idx
        f.Size = UDim2.new(0, QW, 0, QH)
        f.BorderSizePixel = 0
        f.Active = false  -- Fixed: was eating jump/shiftlock input; TextButton child handles clicks
        f.ZIndex = 30
        Instance.new("UICorner", f).CornerRadius = UDim.new(0, _btnCorner)

        if State.moveableMode then
            -- Restore saved absolute position or fall back to default grid position
            if _btnSavedPos[idx] then
                f.Position = _btnSavedPos[idx]
            else
                local containerAbsX = _btnSavedPos.containerPos.X.Offset
                local containerAbsY = _btnSavedPos.containerPos.Y.Offset
                f.Position = UDim2.new(
                    _btnSavedPos.containerPos.X.Scale,
                    containerAbsX + defaultOffX,
                    _btnSavedPos.containerPos.Y.Scale,
                    containerAbsY + defaultOffY
                )
            end
            f.BackgroundColor3 = C_BTN
            f.BackgroundTransparency = 0

            -- TextButton receives all input (sits on top), so attach drag to it
            local btn = Instance.new("TextButton", f)
            btn.Size = UDim2.new(1, 0, 1, 0)
            btn.BackgroundTransparency = 1
            btn.Text = label
            btn.TextColor3 = C_BTN_TEXT
            btn.Font = Enum.Font.GothamBlack
            btn.TextSize = _btnTextSize
            btn.TextWrapped = true
            btn.AutoButtonColor = false
            btn.Selectable = false
            btn.ZIndex = 34
            btn.TextXAlignment = Enum.TextXAlignment.Center

            local dragging, dragStart, startPos = false, nil, nil
            local wasDragged = false
            btn.InputBegan:Connect(function(inp)
                if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then
                    dragging = true
                    wasDragged = false
                    dragStart = inp.Position
                    startPos = f.Position
                    inp.Changed:Connect(function()
                        if inp.UserInputState == Enum.UserInputState.End then
                            dragging = false
                        end
                    end)
                end
            end)
            UIS.InputChanged:Connect(function(inp)
                if not dragging then return end
                if State.quickLocked then return end
                if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseMovement then
                    local dx = inp.Position.X - dragStart.X
                    local dy = inp.Position.Y - dragStart.Y
                    if math.abs(dx) > 6 or math.abs(dy) > 6 then
                        wasDragged = true
                    end
                    local newPos = UDim2.new(startPos.X.Scale, startPos.X.Offset + dx, startPos.Y.Scale, startPos.Y.Offset + dy)
                    f.Position = newPos
                    _btnSavedPos[idx] = newPos
                    autoSave()
                end
            end)
            table.insert(quickBtnFrames_global, f)

            return f, btn, function() return wasDragged end
        else
            -- Normal mode: buttons live inside the container at fixed grid positions
            f.Position = UDim2.new(0, defaultOffX, 0, defaultOffY)
            f.BackgroundColor3 = C_BTN
            f.BackgroundTransparency = 0

            local btn = Instance.new("TextButton", f)
            btn.Size = UDim2.new(1, 0, 1, 0)
            btn.BackgroundTransparency = 1
            btn.Text = label
            btn.TextColor3 = C_BTN_TEXT
            btn.Font = Enum.Font.GothamBlack
            btn.TextSize = _btnTextSize
            btn.TextWrapped = true
            btn.AutoButtonColor = false
            btn.Selectable = false
            btn.ZIndex = 34
            btn.TextXAlignment = Enum.TextXAlignment.Center

            table.insert(quickBtnFrames_global, f)
            return f, btn, function() return false end
        end
    end

    local function makeButton(label, idx)
        return makeDraggableButton(label, idx)
    end
    local function setButtonState(frame, button, isOn)
        if isOn then
            TweenService:Create(frame, TweenInfo.new(0.1), {BackgroundColor3 = C_BTN_ON, BackgroundTransparency = 0}):Play()
            TweenService:Create(button, TweenInfo.new(0.1), {TextColor3 = Color3.fromRGB(0, 0, 0)}):Play()
        else
            TweenService:Create(frame, TweenInfo.new(0.15), {BackgroundColor3 = C_BTN, BackgroundTransparency = 0}):Play()
            TweenService:Create(button, TweenInfo.new(0.15), {TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        end
    end
    -- ========== BUTTON LAYOUT ==========
    local btnFrames, btnButtons, btnWasDragged = {}, {}, {}

    -- Row1: DROP BR | AUTO LEFT
    -- Row2: BAT AIMBOT | AUTO RIGHT
    -- Row3: CARRY SPEED | LAGGER MODE
    -- Row4: TP DOWN | INSTA RESET
    btnFrames[1], btnButtons[1], btnWasDragged[1] = makeButton("DROP\nBR", 1)
    btnFrames[2], btnButtons[2], btnWasDragged[2] = makeButton("AUTO\nLEFT", 2)
    btnFrames[3], btnButtons[3], btnWasDragged[3] = makeButton("BAT\nAIMBOT", 3)
    btnFrames[4], btnButtons[4], btnWasDragged[4] = makeButton("AUTO\nRIGHT", 4)
    _autoLeftBtnFrame  = btnFrames[2]
    _autoRightBtnFrame = btnFrames[4]
    btnFrames[5], btnButtons[5], btnWasDragged[5] = makeButton("CARRY\nSPEED", 5)
    btnFrames[6], btnButtons[6], btnWasDragged[6] = makeButton("LAGGER\nMODE", 6)
    btnFrames[7], btnButtons[7], btnWasDragged[7] = makeButton("TP\nDOWN", 7)
    btnFrames[8], btnButtons[8], btnWasDragged[8] = makeButton("INSTA\nRESET", 8)

    local function playCounterSound()
        pcall(function()
            local s = Instance.new("Sound")
            s.SoundId = "rbxassetid://6895079813"
            s.Volume = 0.4
            s.Parent = workspace
            s:Play()
            task.delay(s.TimeLength + 0.1, function() s:Destroy() end)
        end)
    end

    -- btn1 = DROP BR (one-shot)
    btnButtons[1].Activated:Connect(function()
        if btnWasDragged[1]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        State.dropBREnabled = true
        dropBrainrotNow()
        TweenService:Create(btnFrames[1], TweenInfo.new(0.1), {BackgroundColor3 = C_BTN_ON, BackgroundTransparency = 0}):Play()
        task.delay(0.25, function()
            State.dropBREnabled = false
            stopDropBR()
            TweenService:Create(btnFrames[1], TweenInfo.new(0.15), {BackgroundColor3 = C_BTN, BackgroundTransparency = 0}):Play()
        end)
    end)
    -- btn2 = AUTO LEFT
    btnButtons[2].Activated:Connect(function()
        if btnWasDragged[2]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        if not State.autoLeftEnabled then
            if State.batAimbotEnabled then
                State.batAimbotEnabled = false
                stopBatAimbot()
                setButtonState(btnFrames[3], btnButtons[3], false)
            end
            if State.autoRightEnabled then
                stopAutoRight()
                setButtonState(btnFrames[4], btnButtons[4], false)
            end
            startAutoLeft()
        else
            stopAutoLeft()
        end
        setButtonState(btnFrames[2], btnButtons[2], State.autoLeftEnabled)
        autoSave()
    end)
    -- btn3 = BAT AIMBOT
    btnButtons[3].Activated:Connect(function()
        if btnWasDragged[3]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        State.batAimbotEnabled = not State.batAimbotEnabled
        if State.batAimbotEnabled then
            if State.autoLeftEnabled then
                State.autoLeftEnabled = false
                stopAutoLeft()
                setButtonState(btnFrames[2], btnButtons[2], false)
            end
            if State.autoRightEnabled then
                State.autoRightEnabled = false
                stopAutoRight()
                setButtonState(btnFrames[4], btnButtons[4], false)
            end
            if State.batAimbotAutoDrop then
                dropBrainrotActive = false
                State.dropBREnabled = true
                dropBrainrotNow()
                task.delay(0.3, function() State.dropBREnabled = false; stopDropBR() end)
            end
            startBatAimbot()
        else
            stopBatAimbot()
        end
        setButtonState(btnFrames[3], btnButtons[3], State.batAimbotEnabled)
        autoSave()
    end)
    -- btn4 = AUTO RIGHT
    btnButtons[4].Activated:Connect(function()
        if btnWasDragged[4]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        if not State.autoRightEnabled then
            if State.batAimbotEnabled then
                State.batAimbotEnabled = false
                stopBatAimbot()
                setButtonState(btnFrames[3], btnButtons[3], false)
            end
            if State.autoLeftEnabled then
                stopAutoLeft()
                setButtonState(btnFrames[2], btnButtons[2], false)
            end
            startAutoRight()
        else
            stopAutoRight()
        end
        setButtonState(btnFrames[4], btnButtons[4], State.autoRightEnabled)
        autoSave()
    end)
    -- btn5 = CARRY SPEED
    btnButtons[5].Activated:Connect(function()
        if btnWasDragged[5]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        State.carrySpeedActive = not State.carrySpeedActive
        if State.carrySpeedActive and State.laggerModeEnabled then
            State.laggerModeEnabled = false
            setButtonState(btnFrames[6], btnButtons[6], false)
        end
        setButtonState(btnFrames[5], btnButtons[5], State.carrySpeedActive)
        autoSave()
    end)
    -- btn6 = LAGGER MODE
    btnButtons[6].Activated:Connect(function()
        if btnWasDragged[6]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        State.laggerModeEnabled = not State.laggerModeEnabled
        if State.laggerModeEnabled and State.carrySpeedActive then
            State.carrySpeedActive = false
            setButtonState(btnFrames[5], btnButtons[5], false)
        end
        setButtonState(btnFrames[6], btnButtons[6], State.laggerModeEnabled)
        autoSave()
    end)
    -- btn7 = TP DOWN (one-shot)
    btnButtons[7].Activated:Connect(function()
        if btnWasDragged[7]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        tpDownNow()
        TweenService:Create(btnFrames[7], TweenInfo.new(0.1), {BackgroundColor3 = C_BTN_ON, BackgroundTransparency = 0}):Play()
        task.delay(0.25, function()
            TweenService:Create(btnFrames[7], TweenInfo.new(0.15), {BackgroundColor3 = C_BTN, BackgroundTransparency = 0}):Play()
        end)
    end)
    -- btn8 = INSTA RESET (one-shot)
    btnButtons[8].Activated:Connect(function()
        if btnWasDragged[8]() then return end
        _G._fearGuiClickActive = true; task.delay(0.1, function() _G._fearGuiClickActive = false end)
        doInstaReset()
        TweenService:Create(btnFrames[8], TweenInfo.new(0.1), {BackgroundColor3 = C_BTN_ON, BackgroundTransparency = 0}):Play()
        task.delay(0.4, function()
            TweenService:Create(btnFrames[8], TweenInfo.new(0.15), {BackgroundColor3 = C_BTN, BackgroundTransparency = 0}):Play()
        end)
    end)
    -- Wire up setTpDown/setDropBR so hotkeys can update button visuals
    setTpDown = function(on) setButtonState(btnFrames[7], btnButtons[7], on) end
    setDropBR = function(on) setButtonState(btnFrames[1], btnButtons[1], on) end
    -- Restore visual states from current State so colors survive theme/scale rebuilds
    setButtonState(btnFrames[1], btnButtons[1], State.dropBREnabled)
    setButtonState(btnFrames[2], btnButtons[2], State.autoLeftEnabled)
    setButtonState(btnFrames[3], btnButtons[3], State.batAimbotEnabled)
    setButtonState(btnFrames[4], btnButtons[4], State.autoRightEnabled)
    setButtonState(btnFrames[5], btnButtons[5], State.carrySpeedActive)
    setButtonState(btnFrames[6], btnButtons[6], State.laggerModeEnabled)
end

-- ========== HOTKEY HANDLER ==========
local function setupHotkeys()
    -- Shared action executor used by both keyboard and controller
    local function doAction(action)
        if action == "tpDown" then
            -- One-shot: teleport immediately, never stay toggled on
            tpDownNow()
            State.tpDownEnabled = false
            stopTpDown()
            if setTpDown then setTpDown(false) end
        elseif action == "dropBR" then
            -- One-shot: fire immediately then turn itself off
            State.dropBREnabled = true
            dropBrainrotNow()
            State.dropBREnabled = false
            stopDropBR()
            if setDropBR then setDropBR(false) end
        elseif action == "autoLeft" then
            if not State.autoLeftEnabled then
                if State.batAimbotEnabled then State.batAimbotEnabled = false; stopBatAimbot() end
                if State.autoRightEnabled then State.autoRightEnabled = false; stopAutoRight() end
                startAutoLeft()
            else stopAutoLeft() end
            if setAutoLeft then setAutoLeft(State.autoLeftEnabled) end
        elseif action == "autoRight" then
            if not State.autoRightEnabled then
                if State.batAimbotEnabled then State.batAimbotEnabled = false; stopBatAimbot() end
                if State.autoLeftEnabled then State.autoLeftEnabled = false; stopAutoLeft() end
                startAutoRight()
            else stopAutoRight() end
            if setAutoRight then setAutoRight(State.autoRightEnabled) end
        elseif action == "batAimbot" then
            State.batAimbotEnabled = not State.batAimbotEnabled
            if State.batAimbotEnabled then
                if State.autoLeftEnabled then State.autoLeftEnabled = false; stopAutoLeft() end
                if State.autoRightEnabled then State.autoRightEnabled = false; stopAutoRight() end
                if State.batAimbotAutoDrop then
                    dropBrainrotActive = false
                    State.dropBREnabled = true
                    dropBrainrotNow()
                    task.delay(0.3, function() State.dropBREnabled = false; stopDropBR() end)
                end
                startBatAimbot()
            else stopBatAimbot() end
        elseif action == "manualDrop" then
            dropBrainrotNow()
        elseif action == "manualTpDown" then
            tpDownNow()
        elseif action == "carrySpeed" then
            State.carrySpeedActive = not State.carrySpeedActive
            if State.carrySpeedActive and State.laggerModeEnabled then
                State.laggerModeEnabled = false
                if setButtonState and btnFrames and btnButtons then
                    setButtonState(btnFrames[7], btnButtons[7], false)
                end
            end
        elseif action == "laggerMode" then
            State.laggerModeEnabled = not State.laggerModeEnabled
            if State.laggerModeEnabled and State.carrySpeedActive then
                State.carrySpeedActive = false
                if setButtonState and btnFrames and btnButtons then
                    setButtonState(btnFrames[1], btnButtons[1], false)
                end
            end
        elseif action == "oxgMode" then
            State.oxgModeEnabled = not State.oxgModeEnabled
            if setOxgMode then setOxgMode(State.oxgModeEnabled) end
            autoSave()
        elseif action == "instaGrab" then
            Steal.AutoStealEnabled = not Steal.AutoStealEnabled
            if Steal.AutoStealEnabled then pcall(startAutoSteal) else stopAutoSteal() end
            if setInstaGrab then setInstaGrab(Steal.AutoStealEnabled) end
        elseif action == "infJump" then
            State.infJumpEnabled = not State.infJumpEnabled
            if setInfJump then setInfJump(State.infJumpEnabled) end
        elseif action == "instaReset" then
            doInstaReset()
        elseif action == "duelLagger" then
            if State._toggleDuelLaggerFn then State._toggleDuelLaggerFn() end
        end
        autoSave()
    end

    -- Keyboard input (skip when bind overlay is active)
    UIS.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        if _G._fearBindOverlayActive then return end
        if _G._fearGuiClickActive then return end
        local keyName = input.KeyCode.Name
        for action, b in pairs(Binds) do
            if b.key and b.key == keyName then
                doAction(action)
            end
        end
    end)

    -- Controller / gamepad input (skip when bind overlay is active)
    UIS.InputBegan:Connect(function(input, gameProcessed)
        if _G._fearBindOverlayActive then return end
        if _G._fearGuiClickActive then return end
        if input.UserInputType ~= Enum.UserInputType.Gamepad1 then return end
        -- NOTE: do NOT check gameProcessed here — Roblox always marks gamepad buttons
        -- as gameProcessed=true, which would silently block every controller bind
        local padName = input.KeyCode.Name
        for action, b in pairs(Binds) do
            if b.pad and b.pad == padName then
                doAction(action)
            end
        end
    end)
end
-- ========== MAIN UI ==========
local gui, mainPanel
local bindRowRefs = {}  -- promoted so applyGuiFromConfig can refresh bind labels
local _searchRows = {}  -- outside rebuildUI so doSearch always sees the latest rows
local function rebuildUI()
    if gui then gui:Destroy() end

    -- Apply saved theme so all C_ACCENT / C_ACCENT2 values are correct for this build
    applyTheme(getThemeByName(State.accentTheme or "Gray"))

    local PW, PH, HDR, TBH = math.floor(480 * State.guiScale), math.floor(560 * State.guiScale), 62, 36
    local SBH = 34  -- search bar height
    local SIDE_W = 100 -- sidebar width
    local CONT_Y = SBH + 8 -- offset inside content area for pages (below search bar)
    local CW = PW - SIDE_W - 1 -- content width for centering elements
    gui = Instance.new("ScreenGui")
    gui.Name = "FearDuelGUI"
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 100
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = LP:WaitForChild("PlayerGui")
    
    local function makeDraggable(frame)
        local dragging, dragInput, dragStart, startPos = false, nil, nil, nil
        frame.InputBegan:Connect(function(inp)
            if State.uiLocked then return end
            if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true; dragStart = inp.Position; startPos = frame.Position
                inp.Changed:Connect(function() if inp.UserInputState == Enum.UserInputState.End then dragging = false end end)
            end
        end)
        frame.InputChanged:Connect(function(inp)
            if State.uiLocked then return end
            if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseMovement then dragInput = inp end
        end)
        UIS.InputChanged:Connect(function(inp)
            if State.uiLocked then return end
            if inp == dragInput and dragging then
                local dx = inp.Position.X - dragStart.X
                local dy = inp.Position.Y - dragStart.Y
                frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + dx, startPos.Y.Scale, startPos.Y.Offset + dy)
            end
        end)
    end
    
    mainPanel = Instance.new("Frame", gui)
    mainPanel.Name = "MainPanel"
    mainPanel.Size = UDim2.new(0, PW, 0, PH)
    mainPanel.AnchorPoint = Vector2.new(0.5, 0.5)
    mainPanel.Position = UDim2.new(0.5, 0, 0.45, 0)
    mainPanel.BackgroundColor3 = C_BG
    mainPanel.BackgroundTransparency = 1
    mainPanel.BorderSizePixel = 0
    mainPanel.Active = false  -- Fixed: was intercepting jump/shiftlock when panel open
    mainPanel.ClipsDescendants = false
    Instance.new("UICorner", mainPanel).CornerRadius = UDim.new(0, 14)

    -- Background image
    local bgImage = Instance.new("ImageLabel", mainPanel)
    bgImage.Name = "BackgroundImage"
    bgImage.Size = UDim2.new(1, 0, 1, 0)
    bgImage.Position = UDim2.new(0, 0, 0, 0)
    bgImage.BackgroundTransparency = 1
    bgImage.Image = "rbxassetid://112977078041259"
    bgImage.ScaleType = Enum.ScaleType.Crop
    bgImage.ImageTransparency = 0.15
    bgImage.ZIndex = 1
    Instance.new("UICorner", bgImage).CornerRadius = UDim.new(0, 14)

    local mainStroke = Instance.new("UIStroke", mainPanel)
    mainStroke.Color = C_BORDER
    mainStroke.Transparency = 1
    mainStroke.Thickness = 1.5

    -- UIScale for open animation
    local mainScale = Instance.new("UIScale", mainPanel)
    mainScale.Scale = 0.82

    -- Animate in: scale up + fade in
    task.defer(function()
        TweenService:Create(mainPanel, TweenInfo.new(0.32, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {BackgroundTransparency = 0}):Play()
        TweenService:Create(mainStroke, TweenInfo.new(0.32, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Transparency = 0}):Play()
        TweenService:Create(mainScale, TweenInfo.new(0.32, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()
    end)

    makeDraggable(mainPanel)

    -- ── HEADER (Aphex Hub style, compact 62px) ─────────────────────────────
    local header = Instance.new("Frame", mainPanel)
    header.Size = UDim2.new(1, 0, 0, HDR)
    header.Position = UDim2.new(0, 0, 0, 0)
    header.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
    header.BorderSizePixel = 0
    header.ZIndex = 5
    Instance.new("UICorner", header).CornerRadius = UDim.new(0, 14)

    -- White top accent stripe (Aphex Hub style)
    local topBar = Instance.new("Frame", header)
    topBar.Size = UDim2.new(1, 0, 0, 2)
    topBar.Position = UDim2.new(0, 0, 0, 0)
    topBar.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    topBar.BorderSizePixel = 0
    topBar.ZIndex = 7
    Instance.new("UICorner", topBar).CornerRadius = UDim.new(0, 14)

    -- Left accent bar (Aphex Hub style)
    local hAccentBar = Instance.new("Frame", header)
    hAccentBar.Size = UDim2.new(0, 3, 0.6, 0)
    hAccentBar.Position = UDim2.new(0, 0, 0.2, 0)
    hAccentBar.BackgroundColor3 = Color3.fromRGB(220, 220, 220)
    hAccentBar.BorderSizePixel = 0
    hAccentBar.ZIndex = 7
    Instance.new("UICorner", hAccentBar).CornerRadius = UDim.new(0, 2)

    -- Avatar circle
    local avatarBg = Instance.new("Frame", header)
    avatarBg.Size = UDim2.new(0, 38, 0, 38)
    avatarBg.Position = UDim2.new(0, 12, 0.5, -19)
    avatarBg.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    avatarBg.BorderSizePixel = 0
    avatarBg.ZIndex = 6
    Instance.new("UICorner", avatarBg).CornerRadius = UDim.new(0, 19)
    local avStr = Instance.new("UIStroke", avatarBg)
    avStr.Color = Color3.fromRGB(200, 200, 200); avStr.Thickness = 1.5

    local avatarImg = Instance.new("ImageLabel", avatarBg)
    avatarImg.Size = UDim2.new(1, -4, 1, -4)
    avatarImg.Position = UDim2.new(0, 2, 0, 2)
    avatarImg.BackgroundTransparency = 1
    avatarImg.Image = ""
    avatarImg.ZIndex = 7
    Instance.new("UICorner", avatarImg).CornerRadius = UDim.new(0, 17)
    task.spawn(function()
        pcall(function()
            local img = Players:GetUserThumbnailAsync(LP.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100)
            avatarImg.Image = img
        end)
    end)

    -- Username + handle (left side)
    local usernameLbl = Instance.new("TextLabel", header)
    usernameLbl.Size = UDim2.new(0, 150, 0, 18)
    usernameLbl.Position = UDim2.new(0, 58, 0, 8)
    usernameLbl.BackgroundTransparency = 1
    usernameLbl.Text = LP.DisplayName
    usernameLbl.TextColor3 = Color3.fromRGB(230, 230, 230)
    usernameLbl.Font = Enum.Font.GothamBlack
    usernameLbl.TextSize = 13
    usernameLbl.TextXAlignment = Enum.TextXAlignment.Left
    usernameLbl.ZIndex = 6

    -- Fear Buyer badge
    local fearBuyerBadge = Instance.new("Frame", header)
    fearBuyerBadge.Size = UDim2.new(0, 72, 0, 14)
    fearBuyerBadge.Position = UDim2.new(0, 58, 0, 27)
    fearBuyerBadge.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    fearBuyerBadge.BorderSizePixel = 0
    fearBuyerBadge.ZIndex = 6
    Instance.new("UICorner", fearBuyerBadge).CornerRadius = UDim.new(0, 4)
    local badgeStroke = Instance.new("UIStroke", fearBuyerBadge)
    badgeStroke.Color = Color3.fromRGB(180, 180, 180)
    badgeStroke.Thickness = 1
    local fearBuyerLbl = Instance.new("TextLabel", fearBuyerBadge)
    fearBuyerLbl.Size = UDim2.new(1, 0, 1, 0)
    fearBuyerLbl.BackgroundTransparency = 1
    fearBuyerLbl.Text = "✦ Fear Buyer"
    fearBuyerLbl.TextColor3 = Color3.fromRGB(220, 220, 220)
    fearBuyerLbl.Font = Enum.Font.GothamBold
    fearBuyerLbl.TextSize = 8
    fearBuyerLbl.TextXAlignment = Enum.TextXAlignment.Center
    fearBuyerLbl.ZIndex = 7

    local handleLbl = Instance.new("TextLabel", header)
    handleLbl.Size = UDim2.new(0, 150, 0, 12)
    handleLbl.Position = UDim2.new(0, 136, 0, 29)
    handleLbl.BackgroundTransparency = 1
    handleLbl.Text = "@" .. LP.Name
    handleLbl.TextColor3 = Color3.fromRGB(85, 85, 85)
    handleLbl.Font = Enum.Font.Gotham
    handleLbl.TextSize = 10
    handleLbl.TextXAlignment = Enum.TextXAlignment.Left
    handleLbl.ZIndex = 6

    -- Title right-aligned
    local titleLbl = Instance.new("TextLabel", header)
    titleLbl.Size = UDim2.new(0, 180, 0, 20)
    titleLbl.Position = UDim2.new(1, -192, 0.5, -10)
    titleLbl.BackgroundTransparency = 1
    titleLbl.Text = "◆  FEAR DUELS"
    titleLbl.Font = Enum.Font.GothamBlack
    titleLbl.TextSize = 15
    titleLbl.TextXAlignment = Enum.TextXAlignment.Right
    titleLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLbl.ZIndex = 6

    -- Version badge
    local badgeBg = Instance.new("Frame", header)
    badgeBg.Size = UDim2.new(0, 44, 0, 14)
    badgeBg.Position = UDim2.new(1, -90, 0.5, 6)
    badgeBg.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    badgeBg.BorderSizePixel = 0
    badgeBg.ZIndex = 6
    Instance.new("UICorner", badgeBg).CornerRadius = UDim.new(0, 5)
    local badgeStr = Instance.new("UIStroke", badgeBg)
    badgeStr.Color = Color3.fromRGB(55, 55, 55); badgeStr.Thickness = 1
    local badgeLbl = Instance.new("TextLabel", badgeBg)
    badgeLbl.Size = UDim2.new(1, 0, 1, 0)
    badgeLbl.BackgroundTransparency = 1
    badgeLbl.Text = "v2.3"
    badgeLbl.TextColor3 = Color3.fromRGB(85, 85, 85)
    badgeLbl.Font = Enum.Font.GothamBold
    badgeLbl.TextSize = 8
    badgeLbl.ZIndex = 7

    -- Diamond minimize button
    local minWrap = Instance.new("Frame", header)
    minWrap.Size = UDim2.new(0, 30, 0, 30)
    minWrap.Position = UDim2.new(1, -38, 0.5, -15)
    minWrap.BackgroundTransparency = 1
    minWrap.BorderSizePixel = 0
    minWrap.ZIndex = 7

    local diamondOuter = Instance.new("Frame", minWrap)
    diamondOuter.Size = UDim2.new(0, 18, 0, 18)
    diamondOuter.Position = UDim2.new(0.5, -9, 0.5, -9)
    diamondOuter.Rotation = 45
    diamondOuter.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    diamondOuter.BorderSizePixel = 0
    diamondOuter.ZIndex = 7
    Instance.new("UICorner", diamondOuter).CornerRadius = UDim.new(0, 3)

    local diamondInner = Instance.new("Frame", diamondOuter)
    diamondInner.Size = UDim2.new(0, 7, 0, 7)
    diamondInner.Position = UDim2.new(0.5, -3, 0, 2)
    diamondInner.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
    diamondInner.BackgroundTransparency = 0
    diamondInner.BorderSizePixel = 0
    diamondInner.ZIndex = 8
    Instance.new("UICorner", diamondInner).CornerRadius = UDim.new(0, 2)

    local minBtn = Instance.new("TextButton", minWrap)
    minBtn.Size = UDim2.new(1, 0, 1, 0)
    minBtn.BackgroundTransparency = 1
    minBtn.Text = ""
    minBtn.ZIndex = 9
    minBtn.BorderSizePixel = 0
    minBtn.MouseEnter:Connect(function()
        TweenService:Create(diamondOuter, TweenInfo.new(0.15), {Rotation=90, Size=UDim2.new(0,20,0,20), Position=UDim2.new(0.5,-10,0.5,-10)}):Play()
    end)
    minBtn.MouseLeave:Connect(function()
        TweenService:Create(diamondOuter, TweenInfo.new(0.15), {Rotation=45, Size=UDim2.new(0,18,0,18), Position=UDim2.new(0.5,-9,0.5,-9)}):Play()
    end)

    -- Lock button
    local lockBtn = Instance.new("TextButton", header)
    lockBtn.Size = UDim2.new(0, 30, 0, 24)
    lockBtn.Position = UDim2.new(1, -68, 0.5, -12)
    lockBtn.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
    lockBtn.BorderSizePixel = 0
    lockBtn.Text = "🔓"
    lockBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    lockBtn.Font = Enum.Font.GothamBlack
    lockBtn.TextSize = 12
    lockBtn.ZIndex = 10
    Instance.new("UICorner", lockBtn).CornerRadius = UDim.new(0, 7)
    local lockStroke = Instance.new("UIStroke", lockBtn)
    lockStroke.Color = Color3.fromRGB(55, 55, 55); lockStroke.Thickness = 1

    -- Header divider
    local headerDiv = Instance.new("Frame", mainPanel)
    headerDiv.Size = UDim2.new(1, 0, 0, 1)
    headerDiv.Position = UDim2.new(0, 0, 0, HDR)
    headerDiv.BackgroundColor3 = C_BORDER
    headerDiv.BorderSizePixel = 0
    headerDiv.ZIndex = 3

    -- Lock overlay
    local lockOverlay = Instance.new("TextButton", mainPanel)
    lockOverlay.Name = "LockOverlay"
    lockOverlay.Size = UDim2.new(1, 0, 1, -HDR)
    lockOverlay.Position = UDim2.new(0, 0, 0, HDR)
    lockOverlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    lockOverlay.BackgroundTransparency = 0.82
    lockOverlay.BorderSizePixel = 0
    lockOverlay.Text = "🔒  LOCKED\nTap the lock icon to unlock"
    lockOverlay.TextColor3 = Color3.fromRGB(220, 100, 100)
    lockOverlay.Font = Enum.Font.GothamBold
    lockOverlay.TextSize = 13
    lockOverlay.ZIndex = 50
    lockOverlay.Visible = State.uiLocked
    Instance.new("UICorner", lockOverlay).CornerRadius = UDim.new(0, 20)

    lockBtn.Text = State.uiLocked and "🔒" or "🔓"
    lockBtn.TextColor3 = State.uiLocked and Color3.fromRGB(255,100,100) or C_ACCENT2
    lockBtn.BackgroundColor3 = State.uiLocked and Color3.fromRGB(60,40,40) or Color3.fromRGB(40, 40, 45)
    lockBtn.Activated:Connect(function()
        State.uiLocked = not State.uiLocked
        lockOverlay.Visible = State.uiLocked
        lockBtn.Text = State.uiLocked and "🔒" or "🔓"
        TweenService:Create(lockBtn, TweenInfo.new(0.15), {
            TextColor3 = State.uiLocked and Color3.fromRGB(255,100,100) or C_ACCENT2,
            BackgroundColor3 = State.uiLocked and Color3.fromRGB(60,40,40) or Color3.fromRGB(40, 40, 45)
        }):Play()
        if setLockUI then setLockUI(State.uiLocked) end
        autoSave()
    end)
    
    local miniBar = Instance.new("Frame", gui)
    miniBar.Size = UDim2.new(0, 36, 0, 72)
    miniBar.AnchorPoint = Vector2.new(0, 0.5)
    miniBar.Position = UDim2.new(0, 8, 0.5, 0)
    miniBar.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
    miniBar.BorderSizePixel = 0
    miniBar.Active = true
    miniBar.Visible = false
    Instance.new("UICorner", miniBar).CornerRadius = UDim.new(0, 10)
    local miniStr = Instance.new("UIStroke", miniBar)
    miniStr.Color = Color3.fromRGB(70, 70, 70); miniStr.Thickness = 1.5
    makeDraggable(miniBar)
    local miniTxt = Instance.new("TextLabel", miniBar)
    miniTxt.Size = UDim2.new(1, 0, 1, -8)
    miniTxt.Position = UDim2.new(0, 0, 0, 4)
    miniTxt.BackgroundTransparency = 1
    miniTxt.Text = "◆"
    miniTxt.TextColor3 = C_ACCENT
    miniTxt.Font = Enum.Font.GothamBold
    miniTxt.TextSize = 14
    miniTxt.TextXAlignment = Enum.TextXAlignment.Center
    miniTxt.ZIndex = 5
    local miniClickBtn = Instance.new("TextButton", miniBar)
    miniClickBtn.Size = UDim2.new(1, 0, 1, 0)
    miniClickBtn.BackgroundTransparency = 1
    miniClickBtn.Text = ""
    miniClickBtn.ZIndex = 6
    local function showGUI()
        mainPanel.Visible = true
        miniBar.Visible = false
        State.guiVisible = true
        -- Animate in: scale up from 0.82 + fade in
        local sc = mainPanel:FindFirstChildOfClass("UIScale")
        if sc then sc.Scale = 0.82 end
        mainPanel.BackgroundTransparency = 1
        local ms = mainPanel:FindFirstChildOfClass("UIStroke")
        if ms then ms.Transparency = 1 end
        TweenService:Create(mainPanel, TweenInfo.new(0.28, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {BackgroundTransparency = 0}):Play()
        if sc then TweenService:Create(sc, TweenInfo.new(0.28, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play() end
        if ms then TweenService:Create(ms, TweenInfo.new(0.28), {Transparency = 0}):Play() end
    end
    local function hideGUI()
        miniBar.AnchorPoint = Vector2.new(0, 0.5)
        miniBar.Position = UDim2.new(0, 8, 0.5, 0)
        State.guiVisible = false
        -- Animate out: scale down + fade out, then hide
        local sc = mainPanel:FindFirstChildOfClass("UIScale")
        local ms = mainPanel:FindFirstChildOfClass("UIStroke")
        TweenService:Create(mainPanel, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {BackgroundTransparency = 1}):Play()
        if sc then TweenService:Create(sc, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Scale = 0.82}):Play() end
        if ms then TweenService:Create(ms, TweenInfo.new(0.2), {Transparency = 1}):Play() end
        task.delay(0.22, function()
            if not State.guiVisible then
                mainPanel.Visible = false
                miniBar.Visible = true
            end
        end)
    end
    minBtn.Activated:Connect(hideGUI)
    miniClickBtn.Activated:Connect(showGUI)
    
    -- ── LEFT SIDEBAR (Shit Script style) ──────────────────────────────────
    local sidebar = Instance.new("Frame", mainPanel)
    sidebar.Name = "Sidebar"
    sidebar.Size = UDim2.new(0, SIDE_W, 1, -HDR)
    sidebar.Position = UDim2.new(0, 0, 0, HDR)
    sidebar.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
    sidebar.BackgroundTransparency = 0
    sidebar.BorderSizePixel = 0
    sidebar.ZIndex = 8
    sidebar.ClipsDescendants = true

    local sdivider = Instance.new("Frame", mainPanel)
    sdivider.Size = UDim2.new(0, 1, 1, -HDR)
    sdivider.Position = UDim2.new(0, SIDE_W, 0, HDR)
    sdivider.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    sdivider.BorderSizePixel = 0
    sdivider.ZIndex = 9

    local sideLL = Instance.new("UIListLayout", sidebar)
    sideLL.SortOrder = Enum.SortOrder.LayoutOrder
    sideLL.Padding = UDim.new(0, 2)
    local sidePad = Instance.new("UIPadding", sidebar)
    sidePad.PaddingTop = UDim.new(0, 10)
    sidePad.PaddingLeft = UDim.new(0, 6)
    sidePad.PaddingRight = UDim.new(0, 6)

    -- Content area (right of sidebar)
    local contentArea = Instance.new("Frame", mainPanel)
    contentArea.Name = "ContentArea"
    contentArea.Size = UDim2.new(1, -(SIDE_W + 1), 1, -HDR)
    contentArea.Position = UDim2.new(0, SIDE_W + 1, 0, HDR)
    contentArea.BackgroundTransparency = 1
    contentArea.BorderSizePixel = 0
    contentArea.ClipsDescendants = true
    contentArea.ZIndex = 5

    -- Search bar inside content area
    local searchBar = Instance.new("Frame", contentArea)
    searchBar.Name = "SearchBar"
    searchBar.Size = UDim2.new(1, -12, 0, SBH - 4)
    searchBar.Position = UDim2.new(0, 6, 0, 4)
    searchBar.BackgroundColor3 = Color3.fromRGB(14, 14, 14)
    searchBar.BorderSizePixel = 0
    searchBar.ZIndex = 6
    Instance.new("UICorner", searchBar).CornerRadius = UDim.new(0, 8)
    local sbStroke = Instance.new("UIStroke", searchBar)
    sbStroke.Color = Color3.fromRGB(45, 45, 45); sbStroke.Thickness = 1

    local sbIcon = Instance.new("TextLabel", searchBar)
    sbIcon.Size = UDim2.new(0, 22, 1, 0)
    sbIcon.Position = UDim2.new(0, 6, 0, 0)
    sbIcon.BackgroundTransparency = 1
    sbIcon.Text = "🔍"
    sbIcon.TextSize = 12
    sbIcon.Font = Enum.Font.Gotham
    sbIcon.ZIndex = 7

    local sbBox = Instance.new("TextBox", searchBar)
    sbBox.Size = UDim2.new(1, -50, 1, 0)
    sbBox.Position = UDim2.new(0, 26, 0, 0)
    sbBox.BackgroundTransparency = 1
    sbBox.PlaceholderText = "Search..."
    sbBox.PlaceholderColor3 = Color3.fromRGB(90, 90, 105)
    sbBox.Text = ""
    sbBox.TextColor3 = Color3.fromRGB(210, 210, 230)
    sbBox.Font = Enum.Font.Gotham
    sbBox.TextSize = 11
    sbBox.TextXAlignment = Enum.TextXAlignment.Left
    sbBox.ClearTextOnFocus = false
    sbBox.ZIndex = 7

    local sbClear = Instance.new("TextButton", searchBar)
    sbClear.Size = UDim2.new(0, 20, 0, 20)
    sbClear.Position = UDim2.new(1, -24, 0.5, -10)
    sbClear.BackgroundColor3 = Color3.fromRGB(55, 55, 65)
    sbClear.BorderSizePixel = 0
    sbClear.Text = "✕"
    sbClear.TextColor3 = Color3.fromRGB(160, 160, 175)
    sbClear.Font = Enum.Font.GothamBold
    sbClear.TextSize = 9
    sbClear.Visible = false
    sbClear.ZIndex = 8
    Instance.new("UICorner", sbClear).CornerRadius = UDim.new(1, 0)

    local TAB_NAMES = {"Steal", "Movement", "Extras", "Bypass", "Binds", "Display", "Config"}
    local tabPages, tabBtns = {}, {}
    local activeTab = nil

    local function createTabPage(name)
        local sf = Instance.new("ScrollingFrame", contentArea)
        sf.Size = UDim2.new(1, 0, 1, -CONT_Y)
        sf.Position = UDim2.new(0, 0, 0, CONT_Y)
        sf.BackgroundTransparency = 1
        sf.BorderSizePixel = 0
        sf.ScrollBarThickness = 3
        sf.ScrollBarImageColor3 = C_BORDER2
        sf.CanvasSize = UDim2.new(0, 0, 0, 0)
        sf.AutomaticCanvasSize = Enum.AutomaticSize.Y
        sf.Visible = false
        sf.ZIndex = 3
        local ll = Instance.new("UIListLayout", sf)
        ll.SortOrder = Enum.SortOrder.LayoutOrder
        ll.Padding = UDim.new(0, 6)
        local pd = Instance.new("UIPadding", sf)
        pd.PaddingLeft = UDim.new(0, 8); pd.PaddingRight = UDim.new(0, 8)
        pd.PaddingTop = UDim.new(0, 10); pd.PaddingBottom = UDim.new(0, 10)
        tabPages[name] = sf
    end

    local function switchTab(name)
        if activeTab == name then return end
        if activeTab then
            tabPages[activeTab].Visible = false
            local oldBtn = tabBtns[activeTab]
            TweenService:Create(oldBtn, TweenInfo.new(0.12), {
                TextColor3 = Color3.fromRGB(155, 155, 155),
                BackgroundColor3 = Color3.fromRGB(5, 5, 5),
                BackgroundTransparency = 0.3
            }):Play()
            local oldAcc = oldBtn:FindFirstChild("Accent")
            if oldAcc then oldAcc:Destroy() end
        end
        activeTab = name
        tabPages[name].Visible = true
        local newBtn = tabBtns[name]
        TweenService:Create(newBtn, TweenInfo.new(0.12), {
            TextColor3 = Color3.fromRGB(230, 230, 230),
            BackgroundColor3 = Color3.fromRGB(40, 40, 40),
            BackgroundTransparency = 0
        }):Play()
        if not newBtn:FindFirstChild("Accent") then
            local acc = Instance.new("Frame", newBtn)
            acc.Name = "Accent"
            acc.Size = UDim2.new(0, 3, 0.55, 0)
            acc.Position = UDim2.new(0, 0, 0.225, 0)
            acc.BackgroundColor3 = C_ACCENT
            acc.BorderSizePixel = 0
            acc.ZIndex = 10
            Instance.new("UICorner", acc).CornerRadius = UDim.new(0, 2)
        end
    end

    for i, name in ipairs(TAB_NAMES) do
        local btn = Instance.new("TextButton", sidebar)
        btn.Size = UDim2.new(1, 0, 0, 34)
        btn.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
        btn.BackgroundTransparency = 0.3
        btn.BorderSizePixel = 0
        btn.Text = name
        btn.TextColor3 = Color3.fromRGB(155, 155, 155)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 10
        btn.LayoutOrder = i
        btn.ZIndex = 9
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
        btn.MouseEnter:Connect(function()
            if name ~= activeTab then
                TweenService:Create(btn, TweenInfo.new(0.12), {TextColor3 = Color3.fromRGB(230,230,230), BackgroundColor3 = Color3.fromRGB(20,20,20), BackgroundTransparency = 0}):Play()
            end
        end)
        btn.MouseLeave:Connect(function()
            if name ~= activeTab then
                TweenService:Create(btn, TweenInfo.new(0.12), {TextColor3 = Color3.fromRGB(155,155,155), BackgroundColor3 = Color3.fromRGB(5,5,5), BackgroundTransparency = 0.3}):Play()
            end
        end)
        btn.Activated:Connect(function() switchTab(name) end)
        tabBtns[name] = btn
        createTabPage(name)
    end

    -- Search logic
    _searchRows = {}

    local function doSearch(query)
        query = query:lower():gsub("^%s*(.-)%s*$", "%1")
        local isSearching = query ~= ""
        sbClear.Visible = isSearching
        TweenService:Create(sbStroke, TweenInfo.new(0.15), {
            Color = isSearching and Color3.fromRGB(140, 140, 160) or Color3.fromRGB(45, 45, 45)
        }):Play()
        if not isSearching then
            for _, entry in ipairs(_searchRows) do entry.row.Visible = true end
            switchTab(TAB_NAMES[1])
            return
        end
        local tabHasMatch = {}
        for _, entry in ipairs(_searchRows) do
            local matches = entry.label:lower():find(query, 1, true) ~= nil
            entry.row.Visible = matches
            if matches then tabHasMatch[entry.tab] = true end
        end
        for _, n in ipairs(TAB_NAMES) do
            if tabHasMatch[n] then if activeTab ~= n then switchTab(n) end; break end
        end
    end

    sbBox:GetPropertyChangedSignal("Text"):Connect(function() doSearch(sbBox.Text) end)
    sbClear.Activated:Connect(function() sbBox.Text = ""; doSearch("") end)
    sbBox.Focused:Connect(function() _G._fearGuiClickActive = true end)
    sbBox.FocusLost:Connect(function() task.delay(0.1, function() _G._fearGuiClickActive = false end) end)

    -- ── ROW ORDER & HELPERS ──────────────────────────────────────────────────
    local rowOrder = {}
    for _, n in ipairs(TAB_NAMES) do rowOrder[n] = 0 end
    local function LO(n) rowOrder[n] = rowOrder[n] + 1; return rowOrder[n] end
    local function makeGap(n, px) local f = Instance.new("Frame", tabPages[n]); f.Size = UDim2.new(1,0,0,px or 3); f.BackgroundTransparency = 1; f.BorderSizePixel = 0; f.LayoutOrder = LO(n) end
    local function makeSectionLbl(n, text)
        local row = Instance.new("Frame", tabPages[n]); row.Size = UDim2.new(1,0,0,20); row.BackgroundTransparency = 1; row.BorderSizePixel = 0; row.LayoutOrder = LO(n); row.ZIndex = 3
        local lbl = Instance.new("TextLabel", row); lbl.Size = UDim2.new(1,0,1,0); lbl.BackgroundTransparency = 1; lbl.Text = text:upper(); lbl.TextColor3 = Color3.fromRGB(140, 140, 160); lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 10; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
    end
    local function makeToggleRow(n, label, defOn, onToggle, hotkey)
        local row = Instance.new("Frame", tabPages[n])
        row.Size = UDim2.new(1,0,0,36); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0; row.LayoutOrder = LO(n); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7); Instance.new("UIStroke", row).Color = C_BORDER
        -- Register for search
        table.insert(_searchRows, { row = row, label = (hotkey and (label.." ["..hotkey.."]") or label), tab = n })
        
        local displayText = label
        if hotkey then
            displayText = label .. " [" .. hotkey .. "]"
        end
        
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(0.65,0,1,0); lbl.Position = UDim2.new(0,10,0,0)
        lbl.BackgroundTransparency = 1; lbl.Text = displayText; lbl.TextColor3 = C_ACCENT
        lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        
        local PW2, PH2, DW = 36, 18, 12; local DON_X, DOFF_X = -(DW+2), 2
        local pill = Instance.new("Frame", row)
        pill.Size = UDim2.new(0,PW2,0,PH2); pill.Position = UDim2.new(1,-(PW2+6),0.5,-PH2/2)
        pill.BackgroundColor3 = defOn and C_ON_BG or C_OFF_BG; pill.BorderSizePixel = 0; pill.ZIndex = 4
        Instance.new("UICorner", pill).CornerRadius = UDim.new(1,0)
        
        local ps = Instance.new("UIStroke", pill)
        ps.Color = defOn and C_ACCENT2 or C_BORDER; ps.Thickness = 1
        
        local dot = Instance.new("Frame", pill)
        dot.Size = UDim2.new(0,DW,0,DW)
        dot.Position = defOn and UDim2.new(1,DON_X,0.5,-DW/2) or UDim2.new(0,DOFF_X,0.5,-DW/2)
        dot.BackgroundColor3 = defOn and C_WHITE or C_DIM; dot.BorderSizePixel = 0; dot.ZIndex = 5
        Instance.new("UICorner", dot).CornerRadius = UDim.new(1,0)
        
        local isOn = defOn or false
        local function setV(on)
            isOn = on
            TweenService:Create(pill, TweenInfo.new(0.18,Enum.EasingStyle.Quad), {BackgroundColor3 = on and C_ON_BG or C_OFF_BG}):Play()
            TweenService:Create(ps, TweenInfo.new(0.18), {Color = on and C_ACCENT2 or C_BORDER}):Play()
            TweenService:Create(dot, TweenInfo.new(0.18,Enum.EasingStyle.Back), {
                Position = on and UDim2.new(1,DON_X,0.5,-DW/2) or UDim2.new(0,DOFF_X,0.5,-DW/2),
                BackgroundColor3 = on and C_WHITE or C_DIM
            }):Play()
        end
        
        local clk = Instance.new("TextButton", row)
        clk.Size = UDim2.new(1,0,1,0); clk.BackgroundTransparency = 1; clk.Text = ""; clk.ZIndex = 6
        local _togDebounce = false
        clk.Activated:Connect(function()
            if _togDebounce then return end
            _togDebounce = true
            _G._fearGuiClickActive = true
            task.delay(0.3, function() _togDebounce = false end)
            task.delay(0.1, function() _G._fearGuiClickActive = false end)
            isOn = not isOn; setV(isOn)
            if onToggle then pcall(onToggle, isOn) end
        end)
        return setV
    end
    
    local function makeInputRow(n, label, default, onChange)
        local row = Instance.new("Frame", tabPages[n]); row.Size = UDim2.new(1,0,0,56); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0; row.LayoutOrder = LO(n); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7); Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row); lbl.Size = UDim2.new(1,-14,0,20); lbl.Position = UDim2.new(0,10,0,4); lbl.BackgroundTransparency = 1; lbl.Text = label; lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        local curVal = default; local step = 1
        local BTN_W2, BOX_W, BTN_H2 = 32, 64, 26
        local totalW = BTN_W2 + 4 + BOX_W + 4 + BTN_W2
        local startX = math.floor((CW - 16 - totalW) / 2)
        local ROW_Y = 27
        local minusBtn = Instance.new("TextButton", row); minusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); minusBtn.Position = UDim2.new(0,startX,0,ROW_Y); minusBtn.BackgroundColor3 = C_KEY_BG; minusBtn.BorderSizePixel = 0; minusBtn.Text = "-"; minusBtn.TextColor3 = C_ACCENT2; minusBtn.Font = Enum.Font.GothamBlack; minusBtn.TextSize = 16; minusBtn.ZIndex = 5
        Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0,6); Instance.new("UIStroke", minusBtn).Color = C_BORDER2
        local box = Instance.new("TextBox", row); box.Size = UDim2.new(0,BOX_W,0,BTN_H2); box.Position = UDim2.new(0,startX+BTN_W2+4,0,ROW_Y); box.BackgroundColor3 = C_KEY_BG; box.BorderSizePixel = 0; box.Text = tostring(curVal); box.PlaceholderText = "0"; box.TextColor3 = C_ACCENT2; box.Font = Enum.Font.GothamBlack; box.TextSize = 12; box.TextXAlignment = Enum.TextXAlignment.Center; box.ClearTextOnFocus = false; box.ZIndex = 5
        Instance.new("UICorner", box).CornerRadius = UDim.new(0,6)
        local bs = Instance.new("UIStroke", box); bs.Color = C_BORDER; bs.Thickness = 1
        local plusBtn = Instance.new("TextButton", row); plusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); plusBtn.Position = UDim2.new(0,startX+BTN_W2+4+BOX_W+4,0,ROW_Y); plusBtn.BackgroundColor3 = C_KEY_BG; plusBtn.BorderSizePixel = 0; plusBtn.Text = "+"; plusBtn.TextColor3 = C_ACCENT2; plusBtn.Font = Enum.Font.GothamBlack; plusBtn.TextSize = 16; plusBtn.ZIndex = 5
        Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0,6); Instance.new("UIStroke", plusBtn).Color = C_BORDER2
        local function updateVal(newV)
            newV = math.max(0, math.floor(newV * 100 + 0.5) / 100)
            curVal = newV
            box.Text = math.floor(curVal) == curVal and tostring(math.floor(curVal)) or tostring(curVal)
            if onChange then onChange(curVal) end
        end
        box:GetPropertyChangedSignal("Text"):Connect(function()
            local current = box.Text
            local filtered = current:gsub("[^%d%.]", "")
            local firstDot = filtered:find("%.")
            if firstDot then
                local before = filtered:sub(1,firstDot)
                local after = filtered:sub(firstDot+1):gsub("%.", "")
                filtered = before .. after
            end
            if current ~= filtered then box.Text = filtered end
        end)
        box.Focused:Connect(function() TweenService:Create(bs, TweenInfo.new(0.15), {Color = C_ACCENT2}):Play() end)
        box.FocusLost:Connect(function()
            TweenService:Create(bs, TweenInfo.new(0.15), {Color = C_BORDER2}):Play()
            local v = tonumber(box.Text)
            if v then updateVal(v) else box.Text = tostring(curVal) end
        end)
        minusBtn.Activated:Connect(function()
            TweenService:Create(minusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_BORDER2}):Play()
            task.delay(0.12, function() TweenService:Create(minusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_KEY_BG}):Play() end)
            updateVal(curVal - step)
        end)
        plusBtn.Activated:Connect(function()
            TweenService:Create(plusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_BORDER2}):Play()
            task.delay(0.12, function() TweenService:Create(plusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_KEY_BG}):Play() end)
            updateVal(curVal + step)
        end)
        local function startRepeat(btn, delta)
            local held = true
            btn.InputEnded:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then held = false end end)
            task.spawn(function()
                task.wait(0.45)
                while held do updateVal(curVal + delta); task.wait(0.07) end
            end)
        end
        minusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(minusBtn, -step) end end)
        plusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(plusBtn, step) end end)
        return box, updateVal
    end
    -- Combat Tab
    makeSectionLbl("Steal", "AUTO STEAL")
    setInstaGrab = makeToggleRow("Steal", "Auto Steal", Steal.AutoStealEnabled, function(on)
        Steal.AutoStealEnabled = on
        if on then
            if not pcall(startAutoSteal) then Steal.AutoStealEnabled = false; setInstaGrab(false) end
        else
            stopAutoSteal()
        end
        autoSave()
    end)
    makeSectionLbl("Steal", "STEAL SETTINGS")
    local boxGrabRadius
    boxGrabRadius, setGrabRadius = makeInputRow("Steal", "Steal Radius", Steal.StealRadius, function(v) if v >= 5 and v <= 300 then Steal.StealRadius = math.floor(v); Steal.cachedPrompts = {}; Steal.promptCacheTime = 0 end; autoSave() end)
    local boxStealDuration
    boxStealDuration, setStealDuration = makeInputRow("Steal", "Steal Duration (s)", Steal.StealDuration, function(v) if v >= 0.05 and v <= 2 then Steal.StealDuration = v end; autoSave() end)
    -- PANELS (Bypass + Insta Reset)
    do
        local function addCombatPanelBtn(labelText, callback)
            local row = Instance.new("Frame", tabPages["Bypass"])
            row.Size = UDim2.new(1, 0, 0, 40)
            row.BackgroundColor3 = C_ROW
            row.BorderSizePixel = 0
            row.LayoutOrder = LO("Bypass")
            row.ZIndex = 3
            Instance.new("UICorner", row).CornerRadius = UDim.new(0, 7)
            Instance.new("UIStroke", row).Color = C_BORDER

            local lbl = Instance.new("TextLabel", row)
            lbl.Size = UDim2.new(0.55, 0, 1, 0)
            lbl.Position = UDim2.new(0, 10, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Text = labelText
            lbl.TextColor3 = C_ACCENT
            lbl.Font = Enum.Font.GothamBold
            lbl.TextSize = 11
            lbl.TextXAlignment = Enum.TextXAlignment.Left
            lbl.ZIndex = 4

            local btn = Instance.new("TextButton", row)
            btn.Size = UDim2.new(0, 58, 0, 26)
            btn.Position = UDim2.new(1, -68, 0.5, -13)
            btn.BackgroundColor3 = C_OFF_BG
            btn.BorderSizePixel = 0
            btn.Text = "OPEN"
            btn.TextColor3 = C_ACCENT2
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 10
            btn.ZIndex = 5
            Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
            Instance.new("UIStroke", btn).Color = C_BORDER2

            btn.Activated:Connect(function()
                _G._fearGuiClickActive = true
                task.delay(0.1, function() _G._fearGuiClickActive = false end)
                callback()
            end)
        end

        makeGap("Bypass", 5)
        makeSectionLbl("Bypass", "FEAR BYPASS")
        -- ── FEAR BYPASS BUTTON ──────────────────────────────────────────────
        addCombatPanelBtn("Fear Bypass", function()
            local panelName = "fearBypassPanel"
            local existing = gui:FindFirstChild(panelName)
            if existing then existing.Visible = not existing.Visible; return end

            local bypassActivated = false
            local bypassKeybind = Enum.KeyCode.E
            local bypassWaitingForKey = false
            local bypassPower = 10000
            local bypassLagAmount = 0.12
            local bypassLagConn = nil

            local function bypassApplyPower(val)
                bypassPower = math.clamp(val, 10000, 500000)
                local t2 = (bypassPower - 10000) / 490000
                bypassLagAmount = t2 * 0.2
            end
            bypassApplyPower(bypassPower)

            local function bypassStartLag()
                if bypassLagConn then bypassLagConn:Disconnect() end
                bypassLagConn = RunService.RenderStepped:Connect(function()
                    if not bypassActivated then return end
                    if bypassLagAmount > 0 then
                        local t2 = tick()
                        while tick() - t2 < bypassLagAmount do end
                    end
                end)
            end

            local function bypassStopLag()
                bypassActivated = false
                if bypassLagConn then bypassLagConn:Disconnect(); bypassLagConn = nil end
            end

            local bp = Instance.new("Frame", gui)
            bp.Name = panelName
            bp.Size = UDim2.new(0, 220, 0, 320)
            bp.Position = UDim2.new(0.5, -110, 0.5, -160)
            bp.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
            bp.BackgroundTransparency = 0.2
            bp.BorderSizePixel = 0
            bp.Active = true
            bp.ZIndex = 150
            local bpCorner = Instance.new("UICorner", bp); bpCorner.CornerRadius = UDim.new(0, 12)
            local bpStroke = Instance.new("UIStroke", bp); bpStroke.Color = Color3.fromRGB(70,70,70); bpStroke.Thickness = 2
            makeDraggable(bp)

            local bpClose = Instance.new("TextButton", bp)
            bpClose.Size = UDim2.new(0, 22, 0, 22)
            bpClose.Position = UDim2.new(1, -30, 0, 6)
            bpClose.BackgroundColor3 = Color3.fromRGB(180, 60, 80)
            bpClose.BorderSizePixel = 0
            bpClose.Text = "×"
            bpClose.TextColor3 = Color3.new(1,1,1)
            bpClose.Font = Enum.Font.GothamBold
            bpClose.TextSize = 16
            bpClose.AutoButtonColor = false
            bpClose.ZIndex = 153
            Instance.new("UICorner", bpClose).CornerRadius = UDim.new(1, 0)
            bpClose.MouseButton1Click:Connect(function() bp.Visible = false end)

            local bpTitle = Instance.new("TextLabel", bp)
            bpTitle.Size = UDim2.new(1, 0, 0, 50)
            bpTitle.BackgroundTransparency = 1
            bpTitle.Text = "FEAR SPEED BYPASS"
            bpTitle.TextColor3 = Color3.new(1,1,1)
            bpTitle.Font = Enum.Font.GothamBold
            bpTitle.TextSize = 16
            bpTitle.ZIndex = 152

            local bpPowerLbl = Instance.new("TextLabel", bp)
            bpPowerLbl.Size = UDim2.new(0.8, 0, 0, 20)
            bpPowerLbl.Position = UDim2.new(0.1, 0, 0, 60)
            bpPowerLbl.BackgroundTransparency = 1
            bpPowerLbl.Text = "SET POWER (10k - 500k):"
            bpPowerLbl.TextColor3 = Color3.fromRGB(150,150,150)
            bpPowerLbl.Font = Enum.Font.GothamMedium
            bpPowerLbl.TextSize = 11
            bpPowerLbl.TextXAlignment = Enum.TextXAlignment.Left
            bpPowerLbl.ZIndex = 152

            local bpPowerBox = Instance.new("TextBox", bp)
            bpPowerBox.Size = UDim2.new(0.8, 0, 0, 40)
            bpPowerBox.Position = UDim2.new(0.1, 0, 0, 85)
            bpPowerBox.BackgroundColor3 = Color3.fromRGB(40,40,40)
            bpPowerBox.Text = tostring(bypassPower)
            bpPowerBox.TextColor3 = Color3.new(1,1,1)
            bpPowerBox.Font = Enum.Font.GothamBold
            bpPowerBox.TextSize = 16
            bpPowerBox.BorderSizePixel = 0
            bpPowerBox.ClearTextOnFocus = false
            bpPowerBox.ZIndex = 152
            Instance.new("UICorner", bpPowerBox).CornerRadius = UDim.new(0, 8)
            bpPowerBox.FocusLost:Connect(function()
                local val = tonumber(bpPowerBox.Text)
                if val then bypassApplyPower(val); bpPowerBox.Text = tostring(bypassPower)
                else bpPowerBox.Text = tostring(bypassPower) end
            end)

            local bpKbLbl = Instance.new("TextLabel", bp)
            bpKbLbl.Size = UDim2.new(0.8, 0, 0, 20)
            bpKbLbl.Position = UDim2.new(0.1, 0, 0, 135)
            bpKbLbl.BackgroundTransparency = 1
            bpKbLbl.Text = "TOGGLE KEY:"
            bpKbLbl.TextColor3 = Color3.fromRGB(150,150,150)
            bpKbLbl.Font = Enum.Font.GothamMedium
            bpKbLbl.TextSize = 11
            bpKbLbl.TextXAlignment = Enum.TextXAlignment.Left
            bpKbLbl.ZIndex = 152

            local bpKbBtn = Instance.new("TextButton", bp)
            bpKbBtn.Size = UDim2.new(0.8, 0, 0, 40)
            bpKbBtn.Position = UDim2.new(0.1, 0, 0, 160)
            bpKbBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
            bpKbBtn.Text = "E"
            bpKbBtn.TextColor3 = Color3.fromRGB(0,170,255)
            bpKbBtn.Font = Enum.Font.GothamBold
            bpKbBtn.TextSize = 16
            bpKbBtn.BorderSizePixel = 0
            bpKbBtn.AutoButtonColor = false
            bpKbBtn.ZIndex = 152
            Instance.new("UICorner", bpKbBtn).CornerRadius = UDim.new(0, 8)
            bpKbBtn.MouseButton1Click:Connect(function()
                bypassWaitingForKey = true
                bpKbBtn.Text = "..."
                bpKbBtn.TextColor3 = Color3.fromRGB(255,100,100)
            end)

            local bpToggleBtn = Instance.new("TextButton", bp)
            bpToggleBtn.Size = UDim2.new(0.8, 0, 0, 50)
            bpToggleBtn.Position = UDim2.new(0.1, 0, 0, 230)
            bpToggleBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
            bpToggleBtn.Text = "DEACTIVATED"
            bpToggleBtn.TextColor3 = Color3.new(1,1,1)
            bpToggleBtn.Font = Enum.Font.GothamBold
            bpToggleBtn.TextSize = 16
            bpToggleBtn.BorderSizePixel = 0
            bpToggleBtn.AutoButtonColor = false
            bpToggleBtn.ZIndex = 152
            Instance.new("UICorner", bpToggleBtn).CornerRadius = UDim.new(0, 8)

            local function bypassToggle()
                if not bypassActivated then
                    bypassActivated = true
                    bpToggleBtn.Text = "ACTIVATED"
                    bpToggleBtn.BackgroundColor3 = Color3.fromRGB(0,150,70)
                    bypassStartLag()
                else
                    bypassStopLag()
                    bpToggleBtn.Text = "DEACTIVATED"
                    bpToggleBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
                end
            end
            bpToggleBtn.MouseButton1Click:Connect(bypassToggle)

            UIS.InputBegan:Connect(function(input, gpe)
                if gpe then return end
                if bypassWaitingForKey then
                    if input.UserInputType == Enum.UserInputType.Keyboard then
                        bypassKeybind = input.KeyCode
                        bpKbBtn.Text = bypassKeybind.Name
                        bpKbBtn.TextColor3 = Color3.fromRGB(0,170,255)
                        bypassWaitingForKey = false
                    end
                    return
                end
                if input.KeyCode == bypassKeybind and bp.Visible then
                    bypassToggle()
                end
            end)

            LP.CharacterAdded:Connect(function()
                task.wait(1)
                if bypassActivated then
                    bypassStopLag(); bypassActivated = true; bypassStartLag()
                end
            end)
        end)

        makeGap("Bypass", 5)
        makeSectionLbl("Bypass", "INSTA RESET")
        -- ── INSTA RESET BUTTON ──────────────────────────────────────────────
        addCombatPanelBtn("Insta Reset", function()
            local panelName = "fearduelsInstaResetPanel"
            local existing = gui:FindFirstChild(panelName)
            if existing then existing.Visible = not existing.Visible; return end

            local irTpPos = CFrame.new(1000003.56, 999999.69, 8.17)
            local irDebounce = false
            local irCon = nil

            local function irRestoreCamera()
                pcall(function()
                    local ch = LP.Character
                    local hm = ch and ch:FindFirstChildOfClass("Humanoid")
                    workspace.CurrentCamera.CameraType = Enum.CameraType.Custom
                    if hm then workspace.CurrentCamera.CameraSubject = hm end
                end)
            end

            local function doReset(onDone)
                if irDebounce then return end
                local c = LP.Character
                if not c then return end
                local hrp2 = c:FindFirstChild("HumanoidRootPart")
                local hum2 = c:FindFirstChildOfClass("Humanoid")
                if not hrp2 or not hum2 then return end
                irDebounce = true
                if irCon then irCon:Disconnect(); irCon = nil end

                -- Zero velocity
                pcall(function() hum2.WalkSpeed = 16 end)
                pcall(function() hrp2.AssemblyLinearVelocity = Vector3.new(0,0,0) end)

                -- Destroy flying carpet if present
                local carpet = c:FindFirstChild("Flying Carpet")
                if carpet then pcall(function() carpet:Destroy() end) end

                -- Lock camera
                pcall(function()
                    local cam = workspace.CurrentCamera
                    cam.CameraType = Enum.CameraType.Scriptable
                    cam.CFrame = cam.CFrame
                end)

                -- Disable collisions so we don't get stuck
                pcall(function()
                    for _, part in ipairs(c:GetDescendants()) do
                        if part:IsA("BasePart") then part.CanCollide = false end
                    end
                end)

                -- Temporarily stop anti-die from blocking the kill
                local savedConns = _antiDieConns
                _antiDieConns = {}
                for _, conn in ipairs(savedConns) do pcall(function() conn:Disconnect() end) end
                pcall(function() hum2:SetStateEnabled(Enum.HumanoidStateType.Dead, true) end)

                -- Force kill: health=0, TP to void, break joints
                pcall(function() hum2.Health = 0 end)
                pcall(function()
                    hrp2.CFrame = irTpPos
                end)
                pcall(function() c:BreakJoints() end)

                LP.CharacterAdded:Once(function(newChar)
                    task.wait(0.1)
                    irRestoreCamera()
                    irDebounce = false
                    if onDone then onDone() end
                end)
                task.delay(5, function() if irDebounce then irDebounce = false; irRestoreCamera(); if onDone then onDone() end end end)
            end

            local panel = Instance.new("Frame", gui)
            panel.Name = panelName
            panel.Size = UDim2.new(0, 240, 0, 84)
            panel.Position = UDim2.new(0.5, -120, 0.5, -42)
            panel.BackgroundColor3 = Color3.fromRGB(0,0,0)
            panel.BorderSizePixel = 0
            panel.Active = true
            panel.ZIndex = 100
            Instance.new("UICorner", panel).CornerRadius = UDim.new(0, 12)
            local pStroke = Instance.new("UIStroke", panel); pStroke.Color = Color3.fromRGB(40,40,40); pStroke.Thickness = 1
            makeDraggable(panel)

            local ptitle = Instance.new("TextLabel", panel)
            ptitle.Size = UDim2.new(1, -50, 0, 36)
            ptitle.Position = UDim2.new(0, 14, 0, 0)
            ptitle.BackgroundTransparency = 1
            ptitle.Text = "fearduels insta reset"
            ptitle.TextColor3 = Color3.fromRGB(230,230,230)
            ptitle.Font = Enum.Font.GothamBlack
            ptitle.TextSize = 14
            ptitle.TextXAlignment = Enum.TextXAlignment.Left
            ptitle.ZIndex = 101

            local pclose = Instance.new("TextButton", panel)
            pclose.Size = UDim2.new(0, 24, 0, 24)
            pclose.Position = UDim2.new(1, -34, 0, 6)
            pclose.BackgroundColor3 = Color3.fromRGB(20,20,20)
            pclose.BorderSizePixel = 0
            pclose.Text = "-"
            pclose.TextColor3 = Color3.fromRGB(200,200,200)
            pclose.Font = Enum.Font.GothamBold
            pclose.TextSize = 14
            pclose.ZIndex = 102
            Instance.new("UICorner", pclose).CornerRadius = UDim.new(0, 5)
            local pcStroke = Instance.new("UIStroke", pclose); pcStroke.Color = Color3.fromRGB(50,50,50); pcStroke.Thickness = 1
            pclose.MouseButton1Click:Connect(function() panel.Visible = false end)

            local pdiv = Instance.new("Frame", panel)
            pdiv.Size = UDim2.new(1, 0, 0, 1)
            pdiv.Position = UDim2.new(0, 0, 0, 36)
            pdiv.BackgroundColor3 = Color3.fromRGB(25,25,25)
            pdiv.BorderSizePixel = 0
            pdiv.ZIndex = 101

            local pResetBtn = Instance.new("TextButton", panel)
            pResetBtn.Size = UDim2.new(1, -20, 0, 34)
            pResetBtn.Position = UDim2.new(0, 10, 0, 43)
            pResetBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
            pResetBtn.BorderSizePixel = 0
            pResetBtn.Text = "INSTA RESET"
            pResetBtn.TextColor3 = Color3.fromRGB(255,255,255)
            pResetBtn.Font = Enum.Font.GothamBlack
            pResetBtn.TextSize = 14
            pResetBtn.ZIndex = 102
            Instance.new("UICorner", pResetBtn).CornerRadius = UDim.new(0, 8)
            local prStroke = Instance.new("UIStroke", pResetBtn); prStroke.Color = Color3.fromRGB(80,80,80); prStroke.Thickness = 1
            pResetBtn.MouseButton1Click:Connect(function()
                if irDebounce then return end
                pResetBtn.Text = "RESETTING..."
                pResetBtn.BackgroundColor3 = Color3.fromRGB(30,30,30)
                doReset(function()
                    if pResetBtn and pResetBtn.Parent then
                        pResetBtn.Text = "INSTA RESET"
                        pResetBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
                    end
                end)
            end)
        end)

        -- ── OPEN ANTI BUTTON ────────────────────────────────────────────────
        makeGap("Bypass", 5)
        makeSectionLbl("Bypass", "ANTI PANEL")
        addCombatPanelBtn("Open Anti", function()
            local panelName = "fearAntiPanel"
            local existing = gui:FindFirstChild(panelName)
            if existing then existing.Visible = not existing.Visible; return end

            -- ── Inf Jump state ──
            local infJumpActive = false
            local infJumpSession2 = 0
            local lastBurstTime2 = 0
            local HOLD_INTERVAL2 = 0.055
            local infJumpRenderConn2 = nil
            local touchJumpHeld2 = false

            local infJumpRay2 = RaycastParams.new()
            infJumpRay2.FilterType = Enum.RaycastFilterType.Exclude

            local function infJumpFloorStandY2(r, hum, char)
                infJumpRay2.FilterDescendantsInstances = { char }
                local hit = workspace:Raycast(r.Position + Vector3.new(0, 2.25, 0), Vector3.new(0, -200, 0), infJumpRay2)
                if not hit then return nil, nil end
                local floorY = hit.Position.Y
                local standY = floorY + hum.HipHeight + r.Size.Y * 0.5 + 0.12
                return standY, floorY
            end

            local function infJumpCorrectSink2(r, hum, _char, floorY, standY)
                if not floorY or not standY then return end
                if r.Position.Y >= floorY + 0.35 then return end
                r.CFrame = CFrame.new(r.Position.X, standY, r.Position.Z) * (r.CFrame - r.CFrame.Position)
                local v = r.AssemblyLinearVelocity
                r.AssemblyLinearVelocity = Vector3.new(v.X, math.max(0, v.Y), v.Z)
            end

            local function isJumpHeld2()
                if UIS:IsKeyDown(Enum.KeyCode.Space) then return true end
                local ok, down = pcall(function() return UIS:IsGamepadButtonDown(Enum.UserInputType.Gamepad1, Enum.KeyCode.ButtonA) end)
                if ok and down then return true end
                return touchJumpHeld2
            end

            local function doInfJumpBurst2()
                if not infJumpActive then return end
                local char = LP.Character
                if not char then return end
                local hrp2 = char:FindFirstChild("HumanoidRootPart")
                local hum2 = char:FindFirstChildOfClass("Humanoid")
                if not hrp2 or not hum2 or hum2.Health <= 0 then return end
                infJumpSession2 = infJumpSession2 + 1
                local session = infJumpSession2
                local jy = 50
                if hum2.UseJumpPower and hum2.JumpPower > 0 then
                    jy = hum2.JumpPower
                else
                    local jh = hum2.JumpHeight
                    if type(jh) == "number" and jh > 0 then
                        jy = math.sqrt(math.max(0, 2 * workspace.Gravity * jh))
                    end
                end
                jy = math.clamp(jy, 35, 95)
                local standY, floorY = infJumpFloorStandY2(hrp2, hum2, char)
                infJumpCorrectSink2(hrp2, hum2, char, floorY, standY)
                local v0 = hrp2.AssemblyLinearVelocity
                hrp2.AssemblyLinearVelocity = Vector3.new(v0.X, math.max(v0.Y * 0.15, jy), v0.Z)
                local postSim = RunService.PostSimulation
                local waitPhys = (postSim and function() postSim:Wait() end) or function() RunService.Heartbeat:Wait() end
                task.spawn(function()
                    local c = char; local boost = jy
                    for i = 1, 10 do
                        waitPhys()
                        if session ~= infJumpSession2 or LP.Character ~= c or not infJumpActive then break end
                        local r2 = c:FindFirstChild("HumanoidRootPart")
                        local h2 = c:FindFirstChildOfClass("Humanoid")
                        if not r2 or not h2 or h2.Health <= 0 then break end
                        local sy, fy = infJumpFloorStandY2(r2, h2, c)
                        infJumpCorrectSink2(r2, h2, c, fy, sy)
                        if i > 2 and h2.FloorMaterial ~= Enum.Material.Air then break end
                        local vel = r2.AssemblyLinearVelocity
                        if vel.Y < boost * 0.72 then
                            r2.AssemblyLinearVelocity = Vector3.new(vel.X, math.min(boost, math.max(vel.Y + boost * 0.42, boost * 0.72)), vel.Z)
                        end
                    end
                end)
            end

            local function startInfJumpHandler2()
                if infJumpRenderConn2 then return end
                infJumpRenderConn2 = RunService.RenderStepped:Connect(function()
                    if not infJumpActive then return end
                    if not isJumpHeld2() then return end
                    local now = tick()
                    if now - lastBurstTime2 < HOLD_INTERVAL2 then return end
                    lastBurstTime2 = now
                    doInfJumpBurst2()
                end)
            end

            local function stopInfJumpHandler2()
                if infJumpRenderConn2 then infJumpRenderConn2:Disconnect(); infJumpRenderConn2 = nil end
                infJumpSession2 = infJumpSession2 + 1
                lastBurstTime2 = 0; touchJumpHeld2 = false
            end

            -- Jump button hook for mobile
            task.defer(function()
                local pg = LP:FindFirstChild("PlayerGui")
                if not pg then return end
                local function hookJumpBtn(btn)
                    if not btn:IsA("GuiButton") or btn.Name ~= "JumpButton" or btn:GetAttribute("AntiPanelJumpHook") then return end
                    btn:SetAttribute("AntiPanelJumpHook", true)
                    btn.MouseButton1Down:Connect(function() touchJumpHeld2 = true end)
                    btn.MouseButton1Up:Connect(function() touchJumpHeld2 = false end)
                    btn.MouseLeave:Connect(function() touchJumpHeld2 = false end)
                end
                for _, d in ipairs(pg:GetDescendants()) do hookJumpBtn(d) end
                pg.DescendantAdded:Connect(hookJumpBtn)
            end)

            UIS.JumpRequest:Connect(function()
                if infJumpActive then lastBurstTime2 = tick(); doInfJumpBurst2() end
            end)

            -- ── Colors ──
            local AC_GREEN = Color3.fromRGB(0, 255, 100)
            local AC_PINK  = Color3.fromRGB(255, 45, 85)
            local C_CARD   = Color3.fromRGB(18, 18, 22)
            local C_BORDER_ANTI = Color3.fromRGB(32, 32, 38)
            local C_PILL_OFF = Color3.fromRGB(28, 28, 32)
            local C_DOT_OFF  = Color3.fromRGB(80, 80, 85)

            -- ── Panel frame ──
            local ap = Instance.new("Frame", gui)
            ap.Name = panelName
            ap.Size = UDim2.new(0, 220, 0, 100)
            ap.Position = UDim2.new(0.5, -110, 0.5, -80)
            ap.BackgroundColor3 = Color3.fromRGB(11, 11, 14)
            ap.BorderSizePixel = 0
            ap.Active = true
            ap.ZIndex = 150
            Instance.new("UICorner", ap).CornerRadius = UDim.new(0, 10)
            makeDraggable(ap)

            -- Gradient stroke
            local apStroke = Instance.new("UIStroke", ap)
            apStroke.Thickness = 1.5
            apStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            local apGrad = Instance.new("UIGradient", apStroke)
            apGrad.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0, AC_GREEN),
                ColorSequenceKeypoint.new(0.5, AC_PINK),
                ColorSequenceKeypoint.new(1, AC_GREEN),
            })
            apGrad.Rotation = 45

            -- Close button
            local apClose = Instance.new("TextButton", ap)
            apClose.Size = UDim2.new(0, 20, 0, 20)
            apClose.Position = UDim2.new(1, -26, 0, 5)
            apClose.BackgroundColor3 = Color3.fromRGB(180, 60, 80)
            apClose.BorderSizePixel = 0
            apClose.Text = "×"
            apClose.TextColor3 = Color3.new(1,1,1)
            apClose.Font = Enum.Font.GothamBold
            apClose.TextSize = 14
            apClose.AutoButtonColor = false
            apClose.ZIndex = 155
            Instance.new("UICorner", apClose).CornerRadius = UDim.new(1, 0)
            apClose.MouseButton1Click:Connect(function() ap.Visible = false end)

            -- Title
            local apTitle = Instance.new("TextLabel", ap)
            apTitle.Size = UDim2.new(1, -30, 0, 30)
            apTitle.Position = UDim2.new(0, 10, 0, 0)
            apTitle.BackgroundTransparency = 1
            apTitle.Text = "FEARAPHEX  .gg/fearduels"
            apTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
            apTitle.Font = Enum.Font.GothamBlack
            apTitle.TextSize = 11
            apTitle.TextXAlignment = Enum.TextXAlignment.Left
            apTitle.ZIndex = 152

            -- Status glow dot
            local apGlowDot = Instance.new("Frame", ap)
            apGlowDot.Size = UDim2.new(0, 6, 0, 6)
            apGlowDot.Position = UDim2.new(1, -14, 0, 12)
            apGlowDot.BackgroundColor3 = AC_GREEN
            apGlowDot.BorderSizePixel = 0
            apGlowDot.ZIndex = 153
            Instance.new("UICorner", apGlowDot).CornerRadius = UDim.new(0, 3)

            -- Helper to create a toggle row inside the panel
            local function makeAntiRow(parent, yPos, labelText, accentColor, cardActive)
                local row = Instance.new("Frame", parent)
                row.Size = UDim2.new(1, -20, 0, 52)
                row.Position = UDim2.new(0, 10, 0, yPos)
                row.BackgroundColor3 = C_CARD
                row.BorderSizePixel = 0
                row.ZIndex = 152
                Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)
                local rowStroke = Instance.new("UIStroke", row)
                rowStroke.Color = C_BORDER_ANTI
                rowStroke.Thickness = 1
                rowStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

                local lbl = Instance.new("TextLabel", row)
                lbl.Size = UDim2.new(1, -70, 0, 18)
                lbl.Position = UDim2.new(0, 10, 0, 9)
                lbl.BackgroundTransparency = 1
                lbl.Text = labelText
                lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
                lbl.Font = Enum.Font.GothamBold
                lbl.TextSize = 12
                lbl.TextXAlignment = Enum.TextXAlignment.Left
                lbl.ZIndex = 153

                local statusLbl = Instance.new("TextLabel", row)
                statusLbl.Size = UDim2.new(1, -70, 0, 14)
                statusLbl.Position = UDim2.new(0, 10, 0, 28)
                statusLbl.BackgroundTransparency = 1
                statusLbl.Text = "STATUS: DISABLED"
                statusLbl.TextColor3 = Color3.fromRGB(150, 150, 160)
                statusLbl.Font = Enum.Font.GothamSemibold
                statusLbl.TextSize = 8
                statusLbl.TextXAlignment = Enum.TextXAlignment.Left
                statusLbl.ZIndex = 153

                local pill = Instance.new("Frame", row)
                pill.Size = UDim2.new(0, 34, 0, 18)
                pill.Position = UDim2.new(1, -44, 0.5, -9)
                pill.BackgroundColor3 = C_PILL_OFF
                pill.ZIndex = 153
                Instance.new("UICorner", pill).CornerRadius = UDim.new(0, 10)
                local pillStroke = Instance.new("UIStroke", pill)
                pillStroke.Color = C_BORDER_ANTI; pillStroke.Thickness = 1

                local ball = Instance.new("Frame", pill)
                ball.Size = UDim2.new(0, 10, 0, 10)
                ball.Position = UDim2.new(0, 4, 0.5, -5)
                ball.BackgroundColor3 = C_DOT_OFF
                ball.ZIndex = 154
                Instance.new("UICorner", ball).CornerRadius = UDim.new(0, 5)

                local hitBtn = Instance.new("TextButton", row)
                hitBtn.Size = UDim2.new(1, 0, 1, 0)
                hitBtn.BackgroundTransparency = 1
                hitBtn.Text = ""
                hitBtn.ZIndex = 155

                local isOn = false
                local function setOn(on)
                    isOn = on
                    statusLbl.Text = on and "STATUS: ACTIVE" or "STATUS: DISABLED"
                    statusLbl.TextColor3 = on and accentColor or Color3.fromRGB(150, 150, 160)
                    TweenService:Create(row, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
                        BackgroundColor3 = on and cardActive or C_CARD
                    }):Play()
                    TweenService:Create(rowStroke, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
                        Color = on and accentColor or C_BORDER_ANTI
                    }):Play()
                    TweenService:Create(ball, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                        Position = on and UDim2.new(1, -14, 0.5, -5) or UDim2.new(0, 4, 0.5, -5),
                        BackgroundColor3 = on and Color3.fromRGB(255, 255, 255) or C_DOT_OFF,
                    }):Play()
                end

                return hitBtn, setOn
            end

            -- Inf Jump row
            local ijHit, setInfJumpOn = makeAntiRow(ap, 34, "Inf Jump", AC_PINK, Color3.fromRGB(45, 25, 35))
            ijHit.MouseButton1Click:Connect(function()
                infJumpActive = not infJumpActive
                setInfJumpOn(infJumpActive)
                if infJumpActive then
                    startInfJumpHandler2()
                else
                    stopInfJumpHandler2()
                end
            end)

            -- Cleanup on panel destroy
            ap.AncestryChanged:Connect(function()
                if not ap.Parent then
                    stopInfJumpHandler2()
                end
            end)
        end)
    end
    -- Movement Tab
    makeSectionLbl("Movement", "Speed")
    local boxNormalSpeed
    boxNormalSpeed, setNormalSpeed = makeInputRow("Movement", "Normal Speed", State.normalSpeed, function(v) if v > 0 and v <= 500 then State.normalSpeed = v end; autoSave() end)
    local boxCarrySpeed
    boxCarrySpeed, setCarrySpeed = makeInputRow("Movement", "Carry Speed", State.carrySpeed, function(v) if v > 0 and v <= 500 then State.carrySpeed = v end; autoSave() end)
    local boxLaggerSpeed
    boxLaggerSpeed, setLaggerSpeed = makeInputRow("Movement", "Lagger Speed", State.laggerSpeed, function(v) if v > 0 and v <= 500 then State.laggerSpeed = v end; autoSave() end)
    makeInputRow("Movement", "Jump Velocity", State.jumpVelocity, function(v) if v > 0 and v <= 500 then State.jumpVelocity = v end; autoSave() end)
    makeInputRow("Movement", "OXG Speed", State.oxgSpeed, function(v) if v > 0 and v <= 500 then State.oxgSpeed = v end; autoSave() end)
    setOxgMode = makeToggleRow("Movement", "OXG Mode", State.oxgModeEnabled, function(on) State.oxgModeEnabled = on; autoSave() end, "K2")
    makeSectionLbl("Movement", "Auto Movement")
    setAutoLeft = makeToggleRow("Movement", "Auto Left", State.autoLeftEnabled, function(on) if on then startAutoLeft() else stopAutoLeft() end; autoSave() end, "V")
    setAutoRight = makeToggleRow("Movement", "Auto Right", State.autoRightEnabled, function(on) if on then startAutoRight() else stopAutoRight() end; autoSave() end, "B")
    -- DROP / TP DOWN CATEGORY
    makeSectionLbl("Movement", "DROP / TP DOWN")
    
    setTpDown = makeToggleRow("Movement", "TP DOWN", State.tpDownEnabled, function(on)
        if on then
            -- Fire once immediately then shut itself off
            tpDownNow()
            State.tpDownEnabled = false
            task.defer(function()
                if setTpDown then setTpDown(false) end
            end)
        else
            State.tpDownEnabled = false
            stopTpDown()
        end
        autoSave()
    end, "Z")
    setDropBR = makeToggleRow("Movement", "DROP BR", State.dropBREnabled, function(on)
        if on then
            -- Fire once immediately then shut itself off
            State.dropBREnabled = true
            startDropBR()
            dropBrainrotNow()
            State.dropBREnabled = false
            stopDropBR()
            task.defer(function()
                if setDropBR then setDropBR(false) end
            end)
        else
            State.dropBREnabled = false
            stopDropBR()
        end
        autoSave()
    end, "X")
    
    -- AUTO TP CATEGORY (New)
    makeSectionLbl("Movement", "AUTO TP")
    setAutoTP = makeToggleRow("Movement", "Auto TP (fall TP)", State.autoTPEnabled, function(on)
        State.autoTPEnabled = on
        if on then
            startAutoTP()
        else
            stopAutoTP()
        end
        autoSave()
    end)
    -- Auto TP Height input row
    do
        local row = Instance.new("Frame", tabPages["Movement"])
        row.Size = UDim2.new(1,0,0,56); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0; row.LayoutOrder = LO("Movement"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7); Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(1,-14,0,20); lbl.Position = UDim2.new(0,10,0,4)
        lbl.BackgroundTransparency = 1; lbl.Text = "Auto TP Height"
        lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold
        lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        
        local BTN_W2, BOX_W, BTN_H2 = 32, 64, 26
        local totalW = BTN_W2 + 4 + BOX_W + 4 + BTN_W2
        local startX = math.floor((CW - 16 - totalW) / 2)
        local ROW_Y = 27
        
        local minusBtn = Instance.new("TextButton", row)
        minusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); minusBtn.Position = UDim2.new(0,startX,0,ROW_Y)
        minusBtn.BackgroundColor3 = C_KEY_BG; minusBtn.BorderSizePixel = 0
        minusBtn.Text = "-"; minusBtn.TextColor3 = C_ACCENT2
        minusBtn.Font = Enum.Font.GothamBlack; minusBtn.TextSize = 16; minusBtn.ZIndex = 5
        Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0,6)
        Instance.new("UIStroke", minusBtn).Color = C_BORDER2
        
        local heightBox = Instance.new("TextBox", row)
        heightBox.Size = UDim2.new(0,BOX_W,0,BTN_H2); heightBox.Position = UDim2.new(0,startX+BTN_W2+4,0,ROW_Y)
        heightBox.BackgroundColor3 = C_KEY_BG; heightBox.BorderSizePixel = 0
        heightBox.Text = tostring(State.autoTPHeight)
        heightBox.TextColor3 = C_ACCENT2; heightBox.Font = Enum.Font.GothamBlack
        heightBox.TextSize = 12; heightBox.TextXAlignment = Enum.TextXAlignment.Center
        heightBox.ClearTextOnFocus = false; heightBox.ZIndex = 5
        Instance.new("UICorner", heightBox).CornerRadius = UDim.new(0,6)
        local bs = Instance.new("UIStroke", heightBox); bs.Color = C_BORDER; bs.Thickness = 1
        
        local plusBtn = Instance.new("TextButton", row)
        plusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); plusBtn.Position = UDim2.new(0,startX+BTN_W2+4+BOX_W+4,0,ROW_Y)
        plusBtn.BackgroundColor3 = C_KEY_BG; plusBtn.BorderSizePixel = 0
        plusBtn.Text = "+"; plusBtn.TextColor3 = C_ACCENT2
        plusBtn.Font = Enum.Font.GothamBlack; plusBtn.TextSize = 16; plusBtn.ZIndex = 5
        Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0,6)
        Instance.new("UIStroke", plusBtn).Color = C_BORDER2
        
        local function updateHeight(newH)
            newH = math.max(1, math.floor(newH))
            State.autoTPHeight = newH
            heightBox.Text = tostring(newH)
            autoSave()
        end
        
        heightBox:GetPropertyChangedSignal("Text"):Connect(function()
            local current = heightBox.Text
            local filtered = current:gsub("[^%d]", "")
            if current ~= filtered then heightBox.Text = filtered end
        end)
        heightBox.Focused:Connect(function() TweenService:Create(bs, TweenInfo.new(0.15), {Color = C_ACCENT2}):Play() end)
        heightBox.FocusLost:Connect(function()
            TweenService:Create(bs, TweenInfo.new(0.15), {Color = C_BORDER2}):Play()
            local v = tonumber(heightBox.Text)
            if v and v > 0 then updateHeight(v) else heightBox.Text = tostring(State.autoTPHeight) end
        end)
        minusBtn.Activated:Connect(function()
            TweenService:Create(minusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_BORDER2}):Play()
            task.delay(0.12, function() TweenService:Create(minusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_KEY_BG}):Play() end)
            updateHeight(State.autoTPHeight - 1)
        end)
        plusBtn.Activated:Connect(function()
            TweenService:Create(plusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_BORDER2}):Play()
            task.delay(0.12, function() TweenService:Create(plusBtn, TweenInfo.new(0.08), {BackgroundColor3 = C_KEY_BG}):Play() end)
            updateHeight(State.autoTPHeight + 1)
        end)
        local function startRepeat(btn, delta)
            local held = true
            btn.InputEnded:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then held = false end end)
            task.spawn(function()
                task.wait(0.45)
                while held do updateHeight(State.autoTPHeight + delta); task.wait(0.07) end
            end)
        end
        minusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(minusBtn, -1) end end)
        plusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(plusBtn, 1) end end)
        setAutoTPHeight = updateHeight
    end
    
    -- Mechanics Tab (merged into Combat)
    makeSectionLbl("Extras", "EXTRAS")
    -- 2-column chip helper
    local function make2ColRow(n, labelA, stateA, cbA, labelB, stateB, cbB)
        local row = Instance.new("Frame", tabPages[n])
        row.Size = UDim2.new(1, 0, 0, 36)
        row.BackgroundTransparency = 1
        row.BorderSizePixel = 0
        row.LayoutOrder = LO(n)
        row.ZIndex = 3
        local function makeChip(label, defOn, cb, xSide)
            local chip = Instance.new("Frame", row)
            chip.Size = UDim2.new(0.48, 0, 1, 0)
            chip.Position = xSide == 0 and UDim2.new(0, 0, 0, 0) or UDim2.new(0.52, 0, 0, 0)
            chip.BackgroundColor3 = defOn and C_ON_BG or C_OFF_BG
            chip.BorderSizePixel = 0
            chip.ZIndex = 3
            Instance.new("UICorner", chip).CornerRadius = UDim.new(0, 7)
            local chipStroke = Instance.new("UIStroke", chip)
            chipStroke.Color = defOn and C_ACCENT or C_BORDER
            chipStroke.Thickness = 1
            local lbl = Instance.new("TextLabel", chip)
            lbl.Size = UDim2.new(1, -6, 1, 0)
            lbl.Position = UDim2.new(0, 6, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Text = label
            lbl.TextColor3 = defOn and C_ACCENT2 or C_DIM
            lbl.Font = Enum.Font.GothamBold
            lbl.TextSize = 10
            lbl.TextXAlignment = Enum.TextXAlignment.Left
            lbl.ZIndex = 4
            local isOn = defOn or false
            local function setV(on)
                isOn = on
                TweenService:Create(chip, TweenInfo.new(0.15), {BackgroundColor3 = on and C_ON_BG or C_OFF_BG}):Play()
                TweenService:Create(chipStroke, TweenInfo.new(0.15), {Color = on and C_ACCENT or C_BORDER}):Play()
                TweenService:Create(lbl, TweenInfo.new(0.15), {TextColor3 = on and C_ACCENT2 or C_DIM}):Play()
            end
            local clk = Instance.new("TextButton", chip)
            clk.Size = UDim2.new(1, 0, 1, 0); clk.BackgroundTransparency = 1; clk.Text = ""; clk.ZIndex = 5
            local _db = false
            clk.Activated:Connect(function()
                if _db then return end; _db = true
                _G._fearGuiClickActive = true
                task.delay(0.3, function() _db = false end)
                task.delay(0.1, function() _G._fearGuiClickActive = false end)
                isOn = not isOn; setV(isOn)
                if cb then pcall(cb, isOn) end
            end)
            return setV
        end
        local setA = makeChip(labelA, stateA, cbA, 0)
        local setB = makeChip(labelB, stateB, cbB, 1)
        return setA, setB
    end

    do
        local a, b = make2ColRow("Extras",
            "Inf Jump", State.infJumpEnabled, function(on) State.infJumpEnabled = on; autoSave() end,
            "Anti Rag", State.antiRagdollEnabled, function(on) State.antiRagdollEnabled = on; if on then startAntiRagdoll() else stopAntiRagdoll() end; autoSave() end)
        setInfJump = a; setAntiRag = b
    end

    do
        -- Manual / Hold mode selector for Inf Jump
        local modeRow = Instance.new("Frame", tabPages["Extras"])
        modeRow.Size = UDim2.new(1, 0, 0, 30)
        modeRow.BackgroundTransparency = 1
        modeRow.BorderSizePixel = 0
        modeRow.LayoutOrder = LO("Extras")

        local jumpModeLbl = Instance.new("TextLabel", modeRow)
        jumpModeLbl.Size = UDim2.new(0.36, 0, 1, 0)
        jumpModeLbl.Position = UDim2.new(0, 0, 0, 0)
        jumpModeLbl.BackgroundTransparency = 1
        jumpModeLbl.Text = "Inf Jump"
        jumpModeLbl.TextColor3 = C_ACCENT
        jumpModeLbl.Font = Enum.Font.GothamBold
        jumpModeLbl.TextSize = 10
        jumpModeLbl.TextXAlignment = Enum.TextXAlignment.Left
        jumpModeLbl.ZIndex = 4

        local function makeJumpModeBtn(label, xSide, isDefault)
            local chip = Instance.new("Frame", modeRow)
            chip.Size = UDim2.new(0.30, 0, 1, 0)
            chip.Position = xSide == 0 and UDim2.new(0.37, 0, 0, 0) or UDim2.new(0.69, 0, 0, 0)
            chip.BackgroundColor3 = isDefault and C_ON_BG or C_OFF_BG
            chip.BorderSizePixel = 0
            chip.ZIndex = 3
            Instance.new("UICorner", chip).CornerRadius = UDim.new(0, 7)
            local chipStroke = Instance.new("UIStroke", chip)
            chipStroke.Color = isDefault and C_ACCENT or C_BORDER
            chipStroke.Thickness = 1
            local lbl = Instance.new("TextLabel", chip)
            lbl.Size = UDim2.new(1, -6, 1, 0)
            lbl.Position = UDim2.new(0, 6, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Text = label
            lbl.TextColor3 = isDefault and C_ACCENT2 or C_DIM
            lbl.Font = Enum.Font.GothamBold
            lbl.TextSize = 10
            lbl.TextXAlignment = Enum.TextXAlignment.Center
            lbl.ZIndex = 4
            local function setActive(on)
                TweenService:Create(chip, TweenInfo.new(0.15), {BackgroundColor3 = on and C_ON_BG or C_OFF_BG}):Play()
                TweenService:Create(chipStroke, TweenInfo.new(0.15), {Color = on and C_ACCENT or C_BORDER}):Play()
                TweenService:Create(lbl, TweenInfo.new(0.15), {TextColor3 = on and C_ACCENT2 or C_DIM}):Play()
            end
            return chip, setActive
        end

        local manChip, setManActive = makeJumpModeBtn("Manual", 0, true)
        local holdChip, setHoldActive = makeJumpModeBtn("Hold", 1, false)

        local manBtn = Instance.new("TextButton", manChip)
        manBtn.Size = UDim2.new(1,0,1,0); manBtn.BackgroundTransparency = 1; manBtn.Text = ""; manBtn.ZIndex = 5
        manBtn.Activated:Connect(function()
            State.infJumpMode = "manual"
            setManActive(true); setHoldActive(false)
            autoSave()
        end)

        local holdBtn = Instance.new("TextButton", holdChip)
        holdBtn.Size = UDim2.new(1,0,1,0); holdBtn.BackgroundTransparency = 1; holdBtn.Text = ""; holdBtn.ZIndex = 5
        holdBtn.Activated:Connect(function()
            State.infJumpMode = "hold"
            setManActive(false); setHoldActive(true)
            autoSave()
        end)

        -- Restore from save
        if State.infJumpMode == "hold" then
            setManActive(false); setHoldActive(true)
        end

        -- expose so loadConfig can sync chips
        syncInfJumpChips = function()
            if State.infJumpMode == "hold" then
                setManActive(false); setHoldActive(true)
            else
                setManActive(true); setHoldActive(false)
            end
        end
    end

    do
        local a, b = make2ColRow("Extras",
            "FPS Boost", State.fpsBoostEnabled, function(on) State.fpsBoostEnabled = on; if on then pcall(applyFPSBoost) end; autoSave() end,
            "Tryhard Anim", State.animEnabled, function(on) State.animEnabled = on; if on then startAnimToggle() else stopAnimToggle() end; autoSave() end)
        setFps = a; setAnimToggle = b
    end

    do
        local a, b = make2ColRow("Extras",
            "Medusa Ctr", State.medusaCounterEnabled, function(on) State.medusaCounterEnabled = on; if on then if LP.Character then setupMedusa(LP.Character) end else stopMedusaCounter() end; autoSave() end,
            "Unwalk", State.unwalkEnabled, function(on) State.unwalkEnabled = on; if on then startUnwalk() else stopUnwalk() end; autoSave() end)
        setMedusaCounter = a
    end
    -- Bat Counter always-on indicator
    do
        local row = Instance.new("Frame", tabPages["Extras"])
        row.Size = UDim2.new(1, 0, 0, 32)
        row.BackgroundColor3 = Color3.fromRGB(18, 38, 18)
        row.BorderSizePixel = 0
        row.LayoutOrder = LO("Extras")
        row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0, 7)
        local stroke = Instance.new("UIStroke", row)
        stroke.Color = Color3.fromRGB(40, 120, 40); stroke.Thickness = 1
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(1, -12, 1, 0)
        lbl.Position = UDim2.new(0, 10, 0, 0)
        lbl.BackgroundTransparency = 1
        lbl.Text = "✓  Bat Counter — Always On"
        lbl.TextColor3 = Color3.fromRGB(80, 210, 80)
        lbl.Font = Enum.Font.GothamBold
        lbl.TextSize = 11
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.ZIndex = 4
    end

    make2ColRow("Extras",
        "Rm Accs", false, function(on)
            if on then
                for _, p in pairs(Players:GetPlayers()) do
                    if p.Character then
                        for _, obj in ipairs(p.Character:GetDescendants()) do
                            if obj:IsA("Accessory") or obj:IsA("Hat") then pcall(function() obj:Destroy() end) end
                        end
                    end
                end
            end
        end,
        "Dark Mode", false, function(on)
            if on then enableDarkMode() else disableDarkMode() end
        end)
    -- ── LAGGER TAB ────────────────────────────────────────────────────────
    do
        makeGap("Bypass", 5)
        makeSectionLbl("Bypass", "LAGGER")
        local openBtn = Instance.new("TextButton", tabPages["Bypass"])
        openBtn.Size = UDim2.new(1, 0, 0, 40)
        openBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        openBtn.BorderSizePixel = 0
        openBtn.Text = "Open Lagger"
        openBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
        openBtn.Font = Enum.Font.GothamBold
        openBtn.TextSize = 13
        openBtn.LayoutOrder = LO("Bypass")
        openBtn.ZIndex = 4
        Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 7)
        local openBtnStroke = Instance.new("UIStroke", openBtn)
        openBtnStroke.Color = Color3.fromRGB(0, 0, 0)
        openBtnStroke.Thickness = 1
        openBtnStroke.Transparency = 0.15

        openBtn.Activated:Connect(function()
            -- destroy any existing instance
            local oldSG = game.CoreGui:FindFirstChild("CursedLagger")
            if oldSG then oldSG:Destroy() end

            local UserInputService2 = game:GetService("UserInputService")
            local TweenService2 = game:GetService("TweenService")
            local RunService2 = game:GetService("RunService")

            local LAGGER_CONFIG = { TableIncrease = 25, Tries = 1, LoopWaitTime = 1.2 }
            local CUSTOM_REMOTE_PATH = "RobloxReplicatedStorage.SetPlayerBlockList"

            local function resolveRemote(path)
                if not path or path == "" then return nil end
                local obj = game
                local cleaned = path:gsub("^game%.", "")
                for segment in cleaned:gmatch("[^%.]+") do
                    if obj then obj = obj[segment] else return nil end
                end
                return obj
            end

            local function getmaxvalue(val)
                local mainvalueifonetable = 499999
                if type(val) ~= "number" then return nil end
                return mainvalueifonetable / (val + 2)
            end

            local function buildBombTable(tableincrease)
                local maintable = {}
                local spammedtable = {}
                table.insert(spammedtable, {})
                local z = spammedtable[1]
                for i = 1, tableincrease do
                    local tableins = {}
                    table.insert(z, tableins)
                    z = tableins
                end
                local maximum = getmaxvalue(tableincrease) or 9999999
                for i = 1, maximum do
                    table.insert(maintable, spammedtable)
                    if i % 5000 == 0 then task.wait() end
                end
                return maintable
            end

            local preBuiltTable = nil
            local remoteInstance = nil
            local wlaggerEnabled = false
            local wlaggerThread = nil

            local function startLaggerLoop()
                if not preBuiltTable then
                    preBuiltTable = buildBombTable(LAGGER_CONFIG.TableIncrease)
                    remoteInstance = resolveRemote(CUSTOM_REMOTE_PATH)
                end
                while wlaggerEnabled do
                    game:GetService("NetworkClient"):SetOutgoingKBPSLimit(math.huge)
                    task.spawn(function()
                        if remoteInstance then
                            pcall(function()
                                if remoteInstance:IsA("RemoteEvent") or remoteInstance:IsA("UnreliableRemoteEvent") then
                                    remoteInstance:FireServer(preBuiltTable)
                                elseif remoteInstance:IsA("RemoteFunction") then
                                    remoteInstance:InvokeServer(preBuiltTable)
                                end
                            end)
                        end
                    end)
                    task.wait(math.max(LAGGER_CONFIG.LoopWaitTime, 0.15))
                end
            end

            local function stopLaggerLoop()
                wlaggerEnabled = false
                if wlaggerThread then coroutine.close(wlaggerThread); wlaggerThread = nil end
            end

            local function startLagger()
                if wlaggerThread then return end
                wlaggerEnabled = true
                wlaggerThread = coroutine.create(startLaggerLoop)
                coroutine.resume(wlaggerThread)
            end

            -- Build the white GUI


            local ScreenGui2 = Instance.new("ScreenGui")
            ScreenGui2.Name = "CursedLagger"
            ScreenGui2.ResetOnSpawn = false
            ScreenGui2.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
            ScreenGui2.DisplayOrder = 999
            ScreenGui2.Parent = game.CoreGui

            local Main2 = Instance.new("Frame")
            Main2.Name = "Main"
            Main2.Size = UDim2.new(0, 200, 0, 82)
            Main2.Position = UDim2.new(0, 10, 0, 10)
            Main2.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
            Main2.BorderSizePixel = 0
            Main2.Active = true
            Main2.Draggable = true
            Main2.Parent = ScreenGui2
            Instance.new("UICorner", Main2).CornerRadius = UDim.new(0, 12)
            local Outline2 = Instance.new("UIStroke", Main2)
            Outline2.Thickness = 1; Outline2.Color = Color3.fromRGB(60,60,70); Outline2.Transparency = 0

            local Effects2 = Instance.new("Frame")
            Effects2.Size = UDim2.new(0,300,0,300)
            Effects2.Position = Main2.Position - UDim2.new(0,50,0,90)
            Effects2.BackgroundTransparency = 1
            Effects2.ClipsDescendants = false
            Effects2.ZIndex = Main2.ZIndex - 1
            Effects2.Parent = ScreenGui2
            local lastMainPos2 = Main2.Position
            Main2:GetPropertyChangedSignal("Position"):Connect(function()
                local delta = Main2.Position - lastMainPos2
                Effects2.Position = Effects2.Position + delta
                lastMainPos2 = Main2.Position
            end)

            local function createSmoke2()
                local smoke = Instance.new("ImageLabel")
                smoke.Image = "rbxassetid://673623088"
                smoke.BackgroundTransparency = 1
                smoke.Size = UDim2.new(0,35,0,35)
                smoke.Position = UDim2.new(math.random(),0,math.random(),0)
                smoke.ImageTransparency = 0.8
                smoke.ImageColor3 = Color3.fromRGB(100,100,100)
                smoke.Rotation = math.random(0,360)
                smoke.Parent = Effects2
                task.spawn(function()
                    while smoke and smoke.Parent do
                        local tw = TweenInfo.new(2+math.random(), Enum.EasingStyle.Linear, Enum.EasingDirection.InOut)
                        TweenService2:Create(smoke, tw, {Position=UDim2.new(math.random(),0,math.random(),0), Rotation=smoke.Rotation+math.random(180,720), ImageTransparency=0.95}):Play()
                        task.wait(2+math.random())
                        TweenService2:Create(smoke, TweenInfo.new(2, Enum.EasingStyle.Linear), {Position=UDim2.new(math.random(),0,math.random(),0), Rotation=smoke.Rotation+math.random(180,360), ImageTransparency=0.8}):Play()
                        task.wait(2)
                    end
                end)
            end
            for _ = 1, 4 do createSmoke2() end

            local function createLaser2()
                local laser = Instance.new("Frame")
                laser.BackgroundColor3 = Color3.fromRGB(128,0,255)
                laser.BorderSizePixel = 0
                laser.Size = UDim2.new(0,1,0,120)
                laser.AnchorPoint = Vector2.new(0.5,0.5)
                laser.Position = UDim2.new(0.5,0,0.5,0)
                laser.Rotation = math.random(0,360)
                laser.BackgroundTransparency = 0.6
                local grad = Instance.new("UIGradient", laser)
                grad.Color = ColorSequence.new({ColorSequenceKeypoint.new(0,Color3.fromRGB(200,100,255)),ColorSequenceKeypoint.new(0.5,Color3.fromRGB(128,0,255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(200,100,255))})
                grad.Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0,0.9),NumberSequenceKeypoint.new(0.5,0.4),NumberSequenceKeypoint.new(1,0.9)})
                laser.Parent = Effects2
                task.spawn(function()
                    while laser and laser.Parent do
                        TweenService2:Create(laser, TweenInfo.new(1.5, Enum.EasingStyle.Linear), {Rotation=laser.Rotation+180}):Play()
                        task.wait(1.5)
                        laser.Rotation = laser.Rotation % 360
                    end
                end)
            end
            for _ = 1, 2 do createLaser2() end

            -- Title
            local Title2 = Instance.new("TextLabel", Main2)
            Title2.Size = UDim2.new(1,-30,0,24); Title2.Position = UDim2.new(0,8,0,0)
            Title2.BackgroundTransparency = 1; Title2.Text = "fear duel lagger"
            Title2.TextColor3 = Color3.fromRGB(255,255,255); Title2.TextSize = 10
            Title2.Font = Enum.Font.GothamBold; Title2.TextXAlignment = Enum.TextXAlignment.Left

            local Close2 = Instance.new("TextButton", Main2)
            Close2.Size = UDim2.new(0,20,0,20); Close2.Position = UDim2.new(1,-24,0,2)
            Close2.BackgroundTransparency = 1; Close2.Text = "✕"
            Close2.TextColor3 = Color3.fromRGB(255,255,255); Close2.TextSize = 14
            Close2.Font = Enum.Font.GothamBold; Close2.BorderSizePixel = 0
            Close2.MouseButton1Click:Connect(function() stopLaggerLoop(); ScreenGui2:Destroy(); Effects2:Destroy() end)

            local Div2 = Instance.new("Frame", Main2)
            Div2.Size = UDim2.new(1,-16,0,1); Div2.Position = UDim2.new(0,8,0,26)
            Div2.BackgroundColor3 = Color3.fromRGB(50,50,55); Div2.BorderSizePixel = 0

            local LagSec2 = Instance.new("TextLabel", Main2)
            LagSec2.Size = UDim2.new(1,-16,0,14); LagSec2.Position = UDim2.new(0,8,0,32)
            LagSec2.BackgroundTransparency = 1; LagSec2.Text = "LARP"
            LagSec2.TextColor3 = Color3.fromRGB(128,0,255); LagSec2.TextSize = 8
            LagSec2.Font = Enum.Font.GothamBold; LagSec2.TextXAlignment = Enum.TextXAlignment.Left
            LagSec2.TextTransparency = 0.2

            local EnableRow2 = Instance.new("Frame", Main2)
            EnableRow2.Size = UDim2.new(1,-16,0,28); EnableRow2.Position = UDim2.new(0,8,0,48)
            EnableRow2.BackgroundColor3 = Color3.fromRGB(30,30,36); EnableRow2.BorderSizePixel = 0
            Instance.new("UICorner", EnableRow2).CornerRadius = UDim.new(0,6)

            local EnableText2 = Instance.new("TextLabel", EnableRow2)
            EnableText2.Size = UDim2.new(1,-50,1,0); EnableText2.Position = UDim2.new(0,8,0,0)
            EnableText2.BackgroundTransparency = 1; EnableText2.Text = "Release"
            EnableText2.TextColor3 = Color3.fromRGB(255,255,255); EnableText2.TextSize = 11
            EnableText2.Font = Enum.Font.GothamSemibold; EnableText2.TextXAlignment = Enum.TextXAlignment.Left

            local ToggleBg2 = Instance.new("Frame", EnableRow2)
            ToggleBg2.Size = UDim2.new(0,32,0,16); ToggleBg2.Position = UDim2.new(1,-40,0.5,-8)
            ToggleBg2.BackgroundColor3 = Color3.fromRGB(180,180,190); ToggleBg2.BorderSizePixel = 0
            Instance.new("UICorner", ToggleBg2).CornerRadius = UDim.new(1,0)

            local ToggleDot2 = Instance.new("Frame", ToggleBg2)
            ToggleDot2.Size = UDim2.new(0,12,0,12); ToggleDot2.Position = UDim2.new(0,2,0.5,-6)
            ToggleDot2.BackgroundColor3 = Color3.fromRGB(255,255,255); ToggleDot2.BorderSizePixel = 0
            Instance.new("UICorner", ToggleDot2).CornerRadius = UDim.new(1,0)

            local ToggleHit2 = Instance.new("TextButton", ToggleBg2)
            ToggleHit2.Size = UDim2.new(1,0,1,0); ToggleHit2.BackgroundTransparency = 1; ToggleHit2.Text = ""

            local function setLagger2(state)
                wlaggerEnabled = state
                local tw = TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut)
                if wlaggerEnabled then
                    TweenService2:Create(ToggleBg2, tw, {BackgroundColor3=Color3.fromRGB(128,0,255)}):Play()
                    TweenService2:Create(ToggleDot2, tw, {Position=UDim2.new(0,18,0.5,-6)}):Play()
                    startLagger()
                else
                    TweenService2:Create(ToggleBg2, tw, {BackgroundColor3=Color3.fromRGB(180,180,190)}):Play()
                    TweenService2:Create(ToggleDot2, tw, {Position=UDim2.new(0,2,0.5,-6)}):Play()
                    stopLaggerLoop()
                end
            end
            ToggleHit2.MouseButton1Click:Connect(function() setLagger2(not wlaggerEnabled) end)

            -- Link to the duelLagger bind in Settings
            State._toggleDuelLaggerFn = function() setLagger2(not wlaggerEnabled) end

            ScreenGui2.AncestryChanged:Connect(function()
                if not ScreenGui2.Parent then Effects2:Destroy() end
            end)
        end)
    end

    -- Display Tab
    makeGap("Display",5)
    makeSectionLbl("Display", "GUI SIZE")

    -- ── GUI SIZE PICKER ────────────────────────────────────────────────────
    do
        local row = Instance.new("Frame", tabPages["Display"])
        row.Size = UDim2.new(1,0,0,52); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Display"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(1,-10,0,18); lbl.Position = UDim2.new(0,10,0,4)
        lbl.BackgroundTransparency = 1; lbl.Text = "Main GUI Size"
        lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11
        lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        local sizes = { {label="Small", scale=0.75}, {label="Medium", scale=1.0}, {label="Large", scale=1.2} }
        local chipW, chipH, chipG = 72, 22, 6
        local totalW = #sizes * chipW + (#sizes-1) * chipG
        local startX = math.floor((CW - 16 - totalW) / 2)
        local chips = {}
        local function refreshGuiChips()
            for i, chip in ipairs(chips) do
                local active = math.abs(sizes[i].scale - State.guiScale) < 0.01
                TweenService:Create(chip, TweenInfo.new(0.12), {
                    BackgroundColor3 = active and Color3.fromRGB(80,80,95) or Color3.fromRGB(30,30,36),
                }):Play()
                chip.TextColor3 = active and C_WHITE or C_DIM
            end
        end
        for i, sz in ipairs(sizes) do
            local chip = Instance.new("TextButton", row)
            chip.Size = UDim2.new(0,chipW,0,chipH)
            chip.Position = UDim2.new(0, startX + (i-1)*(chipW+chipG), 0, 26)
            chip.BackgroundColor3 = Color3.fromRGB(30,30,36)
            chip.BorderSizePixel = 0; chip.Text = sz.label
            chip.TextColor3 = C_DIM; chip.Font = Enum.Font.GothamBold; chip.TextSize = 10
            chip.ZIndex = 5
            Instance.new("UICorner", chip).CornerRadius = UDim.new(0,6)
            Instance.new("UIStroke", chip).Color = C_BORDER2
            chips[i] = chip
            chip.Activated:Connect(function()
                State.guiScale = sz.scale
                refreshGuiChips()
                task.delay(0.15, function() rebuildUI(); rebuildQuickButtons(); applyGuiFromConfig() end)
            end)
        end
        refreshGuiChips()
    end
    makeGap("Display", 4)

    makeGap("Display", 4)

    -- ── MOVEABLE BUTTONS PICKER ────────────────────────────────────────────
    do
        local row = Instance.new("Frame", tabPages["Display"])
        row.Size = UDim2.new(1,0,0,52); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Display"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(1,-10,0,18); lbl.Position = UDim2.new(0,10,0,4)
        lbl.BackgroundTransparency = 1; lbl.Text = "Right Buttons Moveable"
        lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11
        lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        local opts = { {label="Off", val=false}, {label="On", val=true} }
        local chipW, chipH, chipG = 110, 22, 6
        local totalW = #opts * chipW + (#opts-1) * chipG
        local startX = math.floor((CW - 16 - totalW) / 2)
        local mChips = {}
        local function refreshMoveChips()
            for i, chip in ipairs(mChips) do
                local active = opts[i].val == State.moveableMode
                TweenService:Create(chip, TweenInfo.new(0.12), {
                    BackgroundColor3 = active and Color3.fromRGB(80,80,95) or Color3.fromRGB(30,30,36),
                }):Play()
                chip.TextColor3 = active and C_WHITE or C_DIM
            end
        end
        for i, opt in ipairs(opts) do
            local chip = Instance.new("TextButton", row)
            chip.Size = UDim2.new(0,chipW,0,chipH)
            chip.Position = UDim2.new(0, startX + (i-1)*(chipW+chipG), 0, 26)
            chip.BackgroundColor3 = Color3.fromRGB(30,30,36)
            chip.BorderSizePixel = 0; chip.Text = opt.label
            chip.TextColor3 = C_DIM; chip.Font = Enum.Font.GothamBold; chip.TextSize = 10
            chip.ZIndex = 5
            Instance.new("UICorner", chip).CornerRadius = UDim.new(0,6)
            Instance.new("UIStroke", chip).Color = C_BORDER2
            mChips[i] = chip
            chip.Activated:Connect(function()
                State.moveableMode = opt.val
                refreshMoveChips()
                autoSave()
                task.delay(0.15, rebuildQuickButtons)
            end)
        end
        refreshMoveChips()
    end
    makeGap("Display", 4)
    -- ── BUTTON SIZE PICKER ────────────────────────────────────────────────
    do
        local row = Instance.new("Frame", tabPages["Display"])
        row.Size = UDim2.new(1,0,0,56); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Display"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(1,-14,0,20); lbl.Position = UDim2.new(0,10,0,4)
        lbl.BackgroundTransparency = 1; lbl.Text = "Button Size"
        lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11
        lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4
        local curSize = State.quickBtnSize
        local BTN_W2, BOX_W, BTN_H2 = 32, 64, 26
        local totalW = BTN_W2 + 4 + BOX_W + 4 + BTN_W2
        local startX = math.floor((CW - 16 - totalW) / 2)
        local ROW_Y = 27
        local minusBtn = Instance.new("TextButton", row)
        minusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); minusBtn.Position = UDim2.new(0,startX,0,ROW_Y)
        minusBtn.BackgroundColor3 = C_KEY_BG; minusBtn.BorderSizePixel = 0
        minusBtn.Text = "-"; minusBtn.TextColor3 = C_ACCENT2
        minusBtn.Font = Enum.Font.GothamBlack; minusBtn.TextSize = 16; minusBtn.ZIndex = 5
        Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0,6)
        Instance.new("UIStroke", minusBtn).Color = C_BORDER2
        local sizeBox = Instance.new("TextButton", row)
        sizeBox.Size = UDim2.new(0,BOX_W,0,BTN_H2); sizeBox.Position = UDim2.new(0,startX+BTN_W2+4,0,ROW_Y)
        sizeBox.BackgroundColor3 = C_KEY_BG; sizeBox.BorderSizePixel = 0
        sizeBox.Text = tostring(curSize); sizeBox.TextColor3 = C_ACCENT2
        sizeBox.Font = Enum.Font.GothamBlack; sizeBox.TextSize = 12
        sizeBox.TextXAlignment = Enum.TextXAlignment.Center; sizeBox.ZIndex = 5
        Instance.new("UICorner", sizeBox).CornerRadius = UDim.new(0,6)
        Instance.new("UIStroke", sizeBox).Color = C_BORDER
        local plusBtn = Instance.new("TextButton", row)
        plusBtn.Size = UDim2.new(0,BTN_W2,0,BTN_H2); plusBtn.Position = UDim2.new(0,startX+BTN_W2+4+BOX_W+4,0,ROW_Y)
        plusBtn.BackgroundColor3 = C_KEY_BG; plusBtn.BorderSizePixel = 0
        plusBtn.Text = "+"; plusBtn.TextColor3 = C_ACCENT2
        plusBtn.Font = Enum.Font.GothamBlack; plusBtn.TextSize = 16; plusBtn.ZIndex = 5
        Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0,6)
        Instance.new("UIStroke", plusBtn).Color = C_BORDER2
        local function applySize(v)
            curSize = math.clamp(math.floor(v), 30, 150)
            State.quickBtnSize = curSize
            sizeBox.Text = tostring(curSize)
            task.delay(0.05, rebuildQuickButtons)
            autoSave()
        end
        local function flashBtn(b)
            TweenService:Create(b, TweenInfo.new(0.08), {BackgroundColor3 = C_BORDER2}):Play()
            task.delay(0.12, function() TweenService:Create(b, TweenInfo.new(0.08), {BackgroundColor3 = C_KEY_BG}):Play() end)
        end
        minusBtn.Activated:Connect(function() flashBtn(minusBtn); applySize(curSize - 5) end)
        plusBtn.Activated:Connect(function() flashBtn(plusBtn); applySize(curSize + 5) end)
        local function startRepeat(btn, delta)
            local held = true
            btn.InputEnded:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then held = false end end)
            task.spawn(function()
                task.wait(0.45)
                while held do applySize(curSize + delta); task.wait(0.1) end
            end)
        end
        minusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(minusBtn, -5) end end)
        plusBtn.InputBegan:Connect(function(inp) if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then startRepeat(plusBtn, 5) end end)
    end
    makeGap("Display", 4)
    do
        local row = Instance.new("Frame", tabPages["Display"])
        row.Size = UDim2.new(1,0,0,36); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Display"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER

        local lockPosBtn = Instance.new("TextButton", row)
        lockPosBtn.Size = UDim2.new(1,-20,0,24)
        lockPosBtn.Position = UDim2.new(0,10,0.5,-12)
        lockPosBtn.BorderSizePixel = 0
        lockPosBtn.Font = Enum.Font.GothamBold
        lockPosBtn.TextSize = 11
        lockPosBtn.ZIndex = 5
        Instance.new("UICorner", lockPosBtn).CornerRadius = UDim.new(0,6)
        local _lpStroke = Instance.new("UIStroke", lockPosBtn)

        local function refreshLockPosBtn()
            lockPosBtn.Text = State.quickLocked and "🔒  Positions Locked — Tap to Unlock" or "🔓  Lock Button Positions"
            lockPosBtn.TextColor3 = State.quickLocked and Color3.fromRGB(255,100,100) or Color3.fromRGB(100,220,120)
            lockPosBtn.BackgroundColor3 = State.quickLocked and Color3.fromRGB(55,25,25) or Color3.fromRGB(25,55,30)
            _lpStroke.Color = State.quickLocked and Color3.fromRGB(140,50,50) or Color3.fromRGB(50,140,70)
        end
        refreshLockPosBtn()

        lockPosBtn.Activated:Connect(function()
            State.quickLocked = not State.quickLocked
            refreshLockPosBtn()
            autoSave()
        end)
    end
    makeGap("Display", 4)
    -- ── RESET BUTTON POSITIONS ────────────────────────────────────────────
    do
        local row = Instance.new("Frame", tabPages["Display"])
        row.Size = UDim2.new(1,0,0,36); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Display"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER

        local resetBtn = Instance.new("TextButton", row)
        resetBtn.Size = UDim2.new(1,-20,0,24)
        resetBtn.Position = UDim2.new(0,10,0.5,-12)
        resetBtn.BackgroundColor3 = Color3.fromRGB(45,30,30)
        resetBtn.BorderSizePixel = 0
        resetBtn.Text = "Reset Button Positions"
        resetBtn.TextColor3 = Color3.fromRGB(220,100,100)
        resetBtn.Font = Enum.Font.GothamBold
        resetBtn.TextSize = 11
        resetBtn.ZIndex = 5
        Instance.new("UICorner", resetBtn).CornerRadius = UDim.new(0,6)
        local rbStroke = Instance.new("UIStroke", resetBtn)
        rbStroke.Color = Color3.fromRGB(140,50,50); rbStroke.Thickness = 1

        resetBtn.Activated:Connect(function()
            -- Clear per-button positions AND container so everything recalculates from scratch
            for i = 1, 8 do
                _btnSavedPos[i] = nil
            end
            _btnSavedPos.containerPos = nil
            autoSave()
            -- Flash feedback
            TweenService:Create(resetBtn, TweenInfo.new(0.08), {BackgroundColor3 = Color3.fromRGB(120,40,40)}):Play()
            task.delay(0.15, function()
                TweenService:Create(resetBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(45,30,30)}):Play()
            end)
            task.delay(0.15, rebuildQuickButtons)
        end)
    end
    makeGap("Config", 4)
    do
        local row = Instance.new("Frame", tabPages["Config"]); row.Size = UDim2.new(1,0,0,36); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0; row.LayoutOrder = LO("Config"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7); Instance.new("UIStroke", row).Color = C_BORDER
        local lbl = Instance.new("TextLabel", row); lbl.Size = UDim2.new(1,-16,1,0); lbl.Position = UDim2.new(0,10,0,0); lbl.BackgroundTransparency = 1; lbl.Text = "Auto Save Config Enabled"; lbl.TextColor3 = C_ACCENT2; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 6
    end
    -- FOV Slider
    makeGap("Config", 4)
    do
        local FOV_MIN, FOV_MAX = 10, 180
        local row = Instance.new("Frame", tabPages["Config"])
        row.Size = UDim2.new(1,0,0,54); row.BackgroundColor3 = C_ROW; row.BorderSizePixel = 0
        row.LayoutOrder = LO("Config"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER

        local lbl = Instance.new("TextLabel", row)
        lbl.Size = UDim2.new(0.55,0,0,22); lbl.Position = UDim2.new(0,10,0,6)
        lbl.BackgroundTransparency = 1; lbl.Text = "Camera FOV"
        lbl.TextColor3 = C_ACCENT; lbl.Font = Enum.Font.GothamBold
        lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.ZIndex = 4

        local valBox = Instance.new("TextBox", row)
        valBox.Size = UDim2.new(0,44,0,20); valBox.Position = UDim2.new(1,-52,0,8)
        valBox.BackgroundColor3 = Color3.fromRGB(22,22,22)
        valBox.Text = tostring(State.fovValue)
        valBox.TextColor3 = C_ACCENT2; valBox.Font = Enum.Font.GothamBold
        valBox.TextSize = 11; valBox.ClearTextOnFocus = false; valBox.ZIndex = 5
        Instance.new("UICorner", valBox).CornerRadius = UDim.new(0,5)
        Instance.new("UIStroke", valBox).Color = C_BORDER

        local trackBg = Instance.new("Frame", row)
        trackBg.Size = UDim2.new(1,-20,0,6); trackBg.Position = UDim2.new(0,10,0,40)
        trackBg.BackgroundColor3 = Color3.fromRGB(35,35,35); trackBg.BorderSizePixel = 0; trackBg.ZIndex = 4
        Instance.new("UICorner", trackBg).CornerRadius = UDim.new(1,0)

        local pct0 = math.clamp((State.fovValue - FOV_MIN)/(FOV_MAX - FOV_MIN),0,1)
        local fill = Instance.new("Frame", trackBg)
        fill.Size = UDim2.new(pct0,0,1,0); fill.BackgroundColor3 = C_ACCENT
        fill.BorderSizePixel = 0; fill.ZIndex = 4
        Instance.new("UICorner", fill).CornerRadius = UDim.new(1,0)

        local thumb = Instance.new("Frame", trackBg)
        thumb.Size = UDim2.new(0,14,0,14); thumb.AnchorPoint = Vector2.new(0.5,0.5)
        thumb.Position = UDim2.new(pct0,0,0.5,0)
        thumb.BackgroundColor3 = Color3.new(1,1,1); thumb.BorderSizePixel = 0; thumb.ZIndex = 5
        Instance.new("UICorner", thumb).CornerRadius = UDim.new(1,0)

        local dragBtn = Instance.new("TextButton", trackBg)
        dragBtn.Size = UDim2.new(1,0,4,0); dragBtn.Position = UDim2.new(0,0,-1.5,0)
        dragBtn.BackgroundTransparency = 1; dragBtn.Text = ""; dragBtn.ZIndex = 6

        local function setFovVisual(v)
            v = math.clamp(math.floor(v*10+0.5)/10, FOV_MIN, FOV_MAX)
            State.fovValue = v
            local p = (v - FOV_MIN)/(FOV_MAX - FOV_MIN)
            fill.Size = UDim2.new(p,0,1,0)
            thumb.Position = UDim2.new(p,0,0.5,0)
            valBox.Text = tostring(v)
            applyFOV()
            autoSave()
        end
        setFovSlider = setFovVisual

        local fovDragging = false
        dragBtn.MouseButton1Down:Connect(function() fovDragging = true end)
        UIS.InputEnded:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
                fovDragging = false
            end
        end)
        UIS.InputChanged:Connect(function(i)
            if fovDragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
                local rel = (i.Position.X - trackBg.AbsolutePosition.X) / trackBg.AbsoluteSize.X
                local v = FOV_MIN + (FOV_MAX - FOV_MIN) * math.clamp(rel,0,1)
                setFovVisual(v)
            end
        end)
        -- Touch drag support
        dragBtn.InputBegan:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.Touch then fovDragging = true end
        end)
        dragBtn.InputEnded:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.Touch then fovDragging = false end
        end)
        valBox.FocusLost:Connect(function()
            local n = tonumber(valBox.Text)
            if n then setFovVisual(n)
            else valBox.Text = tostring(State.fovValue) end
        end)
    end
    setLockUI = makeToggleRow("Config", "Lock UI", State.uiLocked, function(on) State.uiLocked = on; autoSave() end)

    -- ===== COLOUR THEME PICKER =====
    makeGap("Config", 6)
    do
        -- Section header
        local hdr = Instance.new("Frame", tabPages["Config"])
        hdr.Size = UDim2.new(1,0,0,20); hdr.BackgroundTransparency = 1; hdr.LayoutOrder = LO("Config"); hdr.BorderSizePixel = 0
        local htxt = Instance.new("TextLabel", hdr)
        htxt.Size = UDim2.new(1,0,1,0); htxt.BackgroundTransparency = 1
        htxt.Text = "COLOUR THEME"; htxt.TextColor3 = C_ACCENT
        htxt.Font = Enum.Font.GothamBlack; htxt.TextSize = 10; htxt.TextXAlignment = Enum.TextXAlignment.Left
    end
    makeGap("Config", 3)
    do
        -- Two rows of 4 colour swatches
        local ROW_H = 58
        local SWATCH_W, SWATCH_H, SWATCH_G = 48, 40, 5
        local SWATCHES_PER_ROW = 4
        local totalSwW = SWATCHES_PER_ROW * SWATCH_W + (SWATCHES_PER_ROW-1) * SWATCH_G
        local startSwX = math.floor((CW - 16 - totalSwW) / 2)

        local numRows = math.ceil(#THEMES / SWATCHES_PER_ROW)
        local outerRow = Instance.new("Frame", tabPages["Config"])
        outerRow.Size = UDim2.new(1,0,0, numRows * (SWATCH_H + 6) + 6)
        outerRow.BackgroundColor3 = C_ROW; outerRow.BorderSizePixel = 0
        outerRow.LayoutOrder = LO("Config"); outerRow.ZIndex = 3
        Instance.new("UICorner", outerRow).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", outerRow).Color = C_BORDER

        local swatchBtns = {}

        local function refreshSwatches()
            for _, sw in ipairs(swatchBtns) do
                local isActive = sw._theme.name == State.accentTheme
                TweenService:Create(sw.frame, TweenInfo.new(0.12), {
                    BackgroundColor3 = sw._theme.glow,
                    BackgroundTransparency = isActive and 0 or 0.35,
                }):Play()
                sw.lbl.TextColor3 = isActive and Color3.fromRGB(255,255,255) or Color3.fromRGB(180,180,180)
                if sw.selDot then
                    sw.selDot.Visible = isActive
                end
            end
        end

        for i, theme in ipairs(THEMES) do
            local col = (i-1) % SWATCHES_PER_ROW
            local row = math.floor((i-1) / SWATCHES_PER_ROW)
            local sx = startSwX + col * (SWATCH_W + SWATCH_G)
            local sy = 6 + row * (SWATCH_H + 6)

            local sf = Instance.new("Frame", outerRow)
            sf.Size = UDim2.new(0, SWATCH_W, 0, SWATCH_H)
            sf.Position = UDim2.new(0, sx, 0, sy)
            sf.BackgroundColor3 = theme.glow
            sf.BackgroundTransparency = (theme.name == State.accentTheme) and 0 or 0.35
            sf.BorderSizePixel = 0; sf.ZIndex = 4
            Instance.new("UICorner", sf).CornerRadius = UDim.new(0,8)
            local sfStroke = Instance.new("UIStroke", sf); sfStroke.Color = theme.accent2; sfStroke.Thickness = 1.5

            -- Colour dot at top
            local dot = Instance.new("Frame", sf)
            dot.Size = UDim2.new(0,14,0,14); dot.Position = UDim2.new(0.5,-7,0,4)
            dot.BackgroundColor3 = theme.accent2; dot.BorderSizePixel = 0; dot.ZIndex = 5
            Instance.new("UICorner", dot).CornerRadius = UDim.new(1,0)

            -- Selected tick
            local selDot = Instance.new("TextLabel", sf)
            selDot.Size = UDim2.new(0,12,0,12); selDot.Position = UDim2.new(1,-14,0,2)
            selDot.BackgroundTransparency = 1; selDot.Text = "✓"
            selDot.TextColor3 = Color3.fromRGB(255,255,255); selDot.Font = Enum.Font.GothamBold
            selDot.TextSize = 9; selDot.ZIndex = 6
            selDot.Visible = (theme.name == State.accentTheme)

            local lbl = Instance.new("TextLabel", sf)
            lbl.Size = UDim2.new(1,0,0,14); lbl.Position = UDim2.new(0,0,1,-16)
            lbl.BackgroundTransparency = 1; lbl.Text = theme.name
            lbl.TextColor3 = (theme.name == State.accentTheme) and Color3.fromRGB(255,255,255) or Color3.fromRGB(180,180,180)
            lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 8; lbl.ZIndex = 5

            local clk = Instance.new("TextButton", sf)
            clk.Size = UDim2.new(1,0,1,0); clk.BackgroundTransparency = 1; clk.Text = ""; clk.ZIndex = 7
            table.insert(swatchBtns, { frame = sf, lbl = lbl, selDot = selDot, _theme = theme })

            clk.Activated:Connect(function()
                _G._fearGuiClickActive = true
                task.delay(0.1, function() _G._fearGuiClickActive = false end)
                applyTheme(theme)
                refreshSwatches()
                -- Rebuild GUIs with new theme so everything recolours
                task.delay(0.15, function()
                    rebuildUI()
                    rebuildQuickButtons()
                    applyGuiFromConfig()
                end)
                autoSave()
            end)
        end

        refreshSwatches()
    end
    makeGap("Config", 6)

    -- ===== BINDS EDITOR =====
    makeGap("Binds", 8)
    do
        local hdr = Instance.new("Frame", tabPages["Binds"])
        hdr.Size = UDim2.new(1,0,0,20); hdr.BackgroundTransparency = 1; hdr.LayoutOrder = LO("Binds"); hdr.BorderSizePixel = 0
        local htxt = Instance.new("TextLabel", hdr)
        htxt.Size = UDim2.new(1,0,1,0); htxt.BackgroundTransparency = 1
        htxt.Text = "BINDS  (KB / Controller)"; htxt.TextColor3 = C_ACCENT
        htxt.Font = Enum.Font.GothamBlack; htxt.TextSize = 10; htxt.TextXAlignment = Enum.TextXAlignment.Left
    end

    bindRowRefs = {}   -- action -> label ref for live update
    local listeningFor = nil -- { action, inputType }

    -- Overlay that captures the next input
    local overlayGui = Instance.new("ScreenGui")
    overlayGui.Name = "FearBindOverlay"; overlayGui.ResetOnSpawn = false
    overlayGui.DisplayOrder = 999; overlayGui.IgnoreGuiInset = true
    overlayGui.Parent = LP:WaitForChild("PlayerGui")
    local overlayBg = Instance.new("Frame", overlayGui)
    overlayBg.Size = UDim2.new(1,0,1,0); overlayBg.BackgroundColor3 = Color3.fromRGB(0,0,0)
    overlayBg.BackgroundTransparency = 0.55; overlayBg.BorderSizePixel = 0; overlayBg.Visible = false; overlayBg.ZIndex = 50
    local overlayLbl = Instance.new("TextLabel", overlayBg)
    overlayLbl.Size = UDim2.new(1,0,1,0); overlayLbl.BackgroundTransparency = 1
    overlayLbl.Text = "Press any key or button...\n(Backspace to clear KB bind  /  DPad Up to clear Controller bind)"
    overlayLbl.TextColor3 = C_ACCENT2
    overlayLbl.Font = Enum.Font.GothamBold; overlayLbl.TextSize = 18; overlayLbl.ZIndex = 51
    local overlaySub = Instance.new("TextLabel", overlayBg)
    overlaySub.Size = UDim2.new(1,0,0,24); overlaySub.Position = UDim2.new(0,0,0.6,0)
    overlaySub.BackgroundTransparency = 1; overlaySub.Text = ""
    overlaySub.TextColor3 = C_ACCENT; overlaySub.Font = Enum.Font.GothamBlack; overlaySub.TextSize = 14; overlaySub.ZIndex = 51

    local function stopListening()
        listeningFor = nil
        overlayBg.Visible = false
        task.delay(0.05, function()
            _G._fearBindOverlayActive = false
        end)
    end
    local function startListening(action, inputType, actionLabel, typeLbl)
        listeningFor = { action = action, inputType = inputType }
        overlaySub.Text = "Binding: " .. actionLabel .. "  [" .. typeLbl .. "]"
        overlayBg.Visible = true
        _G._fearBindOverlayActive = true
    end

    -- Capture input when overlay is active
    UIS.InputBegan:Connect(function(input, gameProcessed)
        if not listeningFor then return end
        -- Allow gamepad inputs even if gameProcessed (Roblox marks controller buttons as processed)
        local isGamepad = (input.UserInputType == Enum.UserInputType.Gamepad1)
        if gameProcessed and not isGamepad then return end
        local action = listeningFor.action
        local inputType = listeningFor.inputType
        local kc = input.KeyCode

        -- Clear bind: Backspace = clear keyboard bind, DPadUp = clear controller bind
        -- DPadDown is now usable as a normal bind
        local isClearKey = (inputType == "key" and kc == Enum.KeyCode.Backspace)
            or (inputType == "pad" and kc == Enum.KeyCode.DPadUp)
        if isClearKey then
            if inputType == "key" then Binds[action].key = nil
            else Binds[action].pad = nil end
        else
            -- Filter: only accept real keys for keyboard, gamepad codes for controller
            local isGamepad = (input.UserInputType == Enum.UserInputType.Gamepad1)
            if inputType == "key" and isGamepad then return end
            if inputType == "pad" and not isGamepad then return end
            -- Ignore modifier-only keypresses
            local ignoreKeys = { LeftShift=true, RightShift=true, LeftControl=true, RightControl=true,
                             LeftAlt=true, RightAlt=true, LeftMeta=true, RightMeta=true, Unknown=true }
            if ignoreKeys[kc.Name] then return end
            -- Block reserved gamepad buttons that Roblox uses for movement/jump/menu
            -- ButtonA = X (jump), ButtonB = Circle (roll/cancel), ButtonSelect = touchpad/options
            local reservedPad = {
                ButtonA=true,       -- X on PS4 = jump, DO NOT bind
                ButtonSelect=true,  -- Options/touchpad = menu
                ButtonStart=true,   -- reserved
                ButtonL3=false,      -- left stick click (allowed)
                ButtonR3=true,      -- right stick click
            }
            if isGamepad and reservedPad[kc.Name] then
                -- Show warning in overlay instead of binding
                overlaySub.Text = "⚠ " .. kc.Name .. " is reserved by Roblox! Pick another button."
                task.delay(1.5, function()
                    if listeningFor then
                        overlaySub.Text = "Binding: " .. BIND_LABELS[action] .. "  [Controller]"
                    end
                end)
                return
            end
            if inputType == "key" then Binds[action].key = kc.Name
            else Binds[action].pad = kc.Name end
        end

        -- Update display label
        if bindRowRefs[action] then
            bindRowRefs[action].Text = getBindDisplay(action)
        end
        autoSave()
        stopListening()
    end)

    -- Build one row per action
    for _, action in ipairs(BIND_ORDER) do
        local label = BIND_LABELS[action]
        local row = Instance.new("Frame", tabPages["Binds"])
        row.Size = UDim2.new(1,0,0,38); row.BackgroundColor3 = C_ROW
        row.BorderSizePixel = 0; row.LayoutOrder = LO("Binds"); row.ZIndex = 3
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,7)
        Instance.new("UIStroke", row).Color = C_BORDER

        -- Action name
        local nameLbl = Instance.new("TextLabel", row)
        nameLbl.Size = UDim2.new(0,90,1,0); nameLbl.Position = UDim2.new(0,8,0,0)
        nameLbl.BackgroundTransparency = 1; nameLbl.Text = label
        nameLbl.TextColor3 = C_ACCENT2; nameLbl.Font = Enum.Font.GothamBold
        nameLbl.TextSize = 10; nameLbl.TextXAlignment = Enum.TextXAlignment.Left; nameLbl.ZIndex = 4

        -- Current bind display (KB / Pad)
        local bindLbl = Instance.new("TextLabel", row)
        bindLbl.Size = UDim2.new(0,90,1,0); bindLbl.Position = UDim2.new(0,98,0,0)
        bindLbl.BackgroundTransparency = 1; bindLbl.Text = getBindDisplay(action)
        bindLbl.TextColor3 = C_DIM; bindLbl.Font = Enum.Font.Gotham
        bindLbl.TextSize = 9; bindLbl.TextXAlignment = Enum.TextXAlignment.Left; bindLbl.ZIndex = 4
        bindRowRefs[action] = bindLbl

        -- KB button
        local kbBtn = Instance.new("TextButton", row)
        kbBtn.Size = UDim2.new(0,28,0,22); kbBtn.Position = UDim2.new(1,-62,0.5,-11)
        kbBtn.BackgroundColor3 = C_KEY_BG; kbBtn.BorderSizePixel = 0
        kbBtn.Text = "KB"; kbBtn.TextColor3 = C_ACCENT; kbBtn.Font = Enum.Font.GothamBold
        kbBtn.TextSize = 9; kbBtn.ZIndex = 3
        Instance.new("UICorner", kbBtn).CornerRadius = UDim.new(0,5)
        Instance.new("UIStroke", kbBtn).Color = C_BORDER2

        -- Pad button
        local padBtn = Instance.new("TextButton", row)
        padBtn.Size = UDim2.new(0,28,0,22); padBtn.Position = UDim2.new(1,-30,0.5,-11)
        padBtn.BackgroundColor3 = C_KEY_BG; padBtn.BorderSizePixel = 0
        padBtn.Text = "PAD"; padBtn.TextColor3 = C_ACCENT; padBtn.Font = Enum.Font.GothamBold
        padBtn.TextSize = 8; padBtn.ZIndex = 3
        Instance.new("UICorner", padBtn).CornerRadius = UDim.new(0,5)
        Instance.new("UIStroke", padBtn).Color = C_BORDER2

        local act = action  -- capture for closure
        kbBtn.Activated:Connect(function()
            startListening(act, "key", BIND_LABELS[act], "Keyboard")
        end)
        padBtn.Activated:Connect(function()
            startListening(act, "pad", BIND_LABELS[act], "Controller")
        end)
    end

    makeGap("Binds",5)
    do
        local ft = Instance.new("TextLabel", tabPages["Binds"]); ft.Size = UDim2.new(1,0,0,16); ft.BackgroundTransparency = 1; ft.LayoutOrder = LO("Binds"); ft.Text = "fearduels.cc · v2.3"; ft.TextColor3 = Color3.fromRGB(60, 60, 75); ft.Font = Enum.Font.Gotham; ft.TextSize = 8; ft.TextXAlignment = Enum.TextXAlignment.Center
    end
    switchTab("Steal")
end
local function applyGuiFromConfig()
    local cfg = _loadedCfg
    if not cfg then return end
    if setNormalSpeed and cfg.normalSpeed then setNormalSpeed(cfg.normalSpeed) end
    if setCarrySpeed and cfg.carrySpeed then setCarrySpeed(cfg.carrySpeed) end
    if setLaggerSpeed and cfg.laggerSpeed then setLaggerSpeed(cfg.laggerSpeed) end
    if setGrabRadius and cfg.grabRadius then setGrabRadius(cfg.grabRadius) end
    if setStealDuration and cfg.stealDuration then setStealDuration(math.clamp(cfg.stealDuration, 0.05, 5)) end
    if cfg.autoStealEnabled then Steal.AutoStealEnabled = true; pcall(startAutoSteal); if setInstaGrab then setInstaGrab(true) end end
    if cfg.infJump and setInfJump then setInfJump(true) end
    if cfg.antiRagdoll and setAntiRag then setAntiRag(true); startAntiRagdoll() end
    if cfg.fpsBoost and setFps then setFps(true); pcall(applyFPSBoost) end
    if cfg.brainrotReturnEnabled then State.brainrotReturnEnabled = true end
    if cfg.animEnabled and setAnimToggle then setAnimToggle(true); task.spawn(function() task.wait(0.5); startAnimToggle() end) end
    if cfg.carrySpeedActive then State.carrySpeedActive = true end
    if cfg.laggerModeEnabled then State.laggerModeEnabled = true end
    if cfg.oxgModeEnabled then State.oxgModeEnabled = true end
    if cfg.uiLocked ~= nil then State.uiLocked = cfg.uiLocked; if setLockUI then setLockUI(State.uiLocked) end end
    if cfg.desyncEnabled then State.desyncEnabled = true; applyDesync(true) end
    if cfg.medusaCounterEnabled and setMedusaCounter then setMedusaCounter(true); if LP.Character then setupMedusa(LP.Character) end end
    if cfg.batAimbotEnabled and not State.batAimbotEnabled then
        State.batAimbotEnabled = true
        startBatAimbot()
    end
    if cfg.tpDownEnabled and setTpDown then setTpDown(true) end
    if cfg.dropBREnabled and setDropBR then setDropBR(true) end
    if cfg.autoLeftEnabled and setAutoLeft then setAutoLeft(true); startAutoLeft() end
    if cfg.autoRightEnabled and setAutoRight then setAutoRight(true); startAutoRight() end
    if cfg.fovValue and setFovSlider then setFovSlider(math.clamp(cfg.fovValue, 10, 180)) end
    -- Apply Auto TP settings
    if cfg.autoTPEnabled ~= nil and setAutoTP then
        State.autoTPEnabled = cfg.autoTPEnabled
        setAutoTP(State.autoTPEnabled)
        if State.autoTPEnabled then startAutoTP() else stopAutoTP() end
    end
    if cfg.autoTPHeight and setAutoTPHeight then
        State.autoTPHeight = cfg.autoTPHeight
        setAutoTPHeight(State.autoTPHeight)
    end
    -- Refresh bind labels in the Settings UI so loaded binds are visible
    for action, lbl in pairs(bindRowRefs) do
        lbl.Text = getBindDisplay(action)
    end
end
-- ========== INTRO SEQUENCE (Blue Bands) ==========
local INTRO_MUSIC_ID = "rbxassetid://119414415681261"
local INTRO_DURATION = 11

local introGui = Instance.new("ScreenGui")
introGui.Name = "FearDuelIntro"
introGui.ResetOnSpawn = false
introGui.DisplayOrder = 99999
introGui.IgnoreGuiInset = true
introGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
introGui.Parent = LP:WaitForChild("PlayerGui")

local introOverlay = Instance.new("Frame", introGui)
introOverlay.Size = UDim2.new(1, 0, 1, 0)
introOverlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
introOverlay.BackgroundTransparency = 1
introOverlay.BorderSizePixel = 0
introOverlay.ZIndex = 1

-- Music
local introSound = Instance.new("Sound", introOverlay)
introSound.SoundId = INTRO_MUSIC_ID
introSound.Volume = 0.85
introSound.Looped = false
task.spawn(function()
    task.wait(0.15)
    pcall(function()
        introSound.TimePosition = 45
        introSound:Play()
    end)
end)

-- Title split into two halves for slide-in effect
local introFear = Instance.new("TextLabel", introOverlay)
introFear.Size = UDim2.new(0.5, -10, 0, 60)
introFear.Position = UDim2.new(-1, 0, 0.5, -50)
introFear.BackgroundTransparency = 1
introFear.Text = "FEAR"
introFear.TextColor3 = Color3.fromRGB(255, 255, 255)
introFear.Font = Enum.Font.GothamBlack
introFear.TextSize = 32
introFear.TextXAlignment = Enum.TextXAlignment.Right
introFear.TextTransparency = 0
introFear.ZIndex = 2
introFear.TextStrokeTransparency = 0.6
introFear.TextStrokeColor3 = Color3.fromRGB(180, 180, 180)

local introDuels = Instance.new("TextLabel", introOverlay)
introDuels.Size = UDim2.new(0.5, -10, 0, 60)
introDuels.Position = UDim2.new(2, 0, 0.5, -50)
introDuels.BackgroundTransparency = 1
introDuels.Text = "DUELS"
introDuels.TextColor3 = Color3.fromRGB(255, 255, 255)
introDuels.Font = Enum.Font.GothamBlack
introDuels.TextSize = 32
introDuels.TextXAlignment = Enum.TextXAlignment.Left
introDuels.TextTransparency = 0
introDuels.ZIndex = 2
introDuels.TextStrokeTransparency = 0.6
introDuels.TextStrokeColor3 = Color3.fromRGB(180, 180, 180)

-- Subtitle label
local introSub = Instance.new("TextLabel", introOverlay)
introSub.Size = UDim2.new(1, 0, 0, 24)
introSub.Position = UDim2.new(0, 0, 0.5, 32)
introSub.BackgroundTransparency = 1
introSub.Text = "/fearduels"
introSub.TextColor3 = Color3.fromRGB(200, 200, 200)
introSub.Font = Enum.Font.GothamBold
introSub.TextSize = 15
introSub.TextXAlignment = Enum.TextXAlignment.Center
introSub.TextTransparency = 1
introSub.ZIndex = 2

-- Underline bar
local introLine = Instance.new("Frame", introOverlay)
introLine.Size = UDim2.new(0, 0, 0, 2)
introLine.Position = UDim2.new(0.5, 0, 0.5, 26)
introLine.BackgroundColor3 = Color3.fromRGB(220, 220, 220)
introLine.BorderSizePixel = 0
introLine.ZIndex = 2

-- Animate intro in a separate thread; Initialize runs immediately after
task.spawn(function()
    -- Slide FEAR in from left + zoom out from small
    introFear.TextSize = 8
    TweenService:Create(introFear, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0, 0, 0.5, -50),
        TextSize = 32
    }):Play()
    -- Slide DUELS in from right + zoom out from small
    introDuels.TextSize = 8
    TweenService:Create(introDuels, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 10, 0.5, -50),
        TextSize = 32
    }):Play()
    task.wait(0.5)

    -- Expand underline
    local lineW = 300
    TweenService:Create(introLine, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, lineW, 0, 2),
        Position = UDim2.new(0.5, -lineW / 2, 0.5, 26),
    }):Play()

    -- Fade in subtitle
    TweenService:Create(introSub, TweenInfo.new(0.45), {TextTransparency = 0}):Play()

    -- Hold for the rest of the duration
    local elapsed = 0.5 + 0.45
    local remaining = INTRO_DURATION - elapsed
    if remaining > 0 then task.wait(remaining) end

    -- Slide back out opposite sides
    TweenService:Create(introFear, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        Position = UDim2.new(-1, 0, 0.5, -50),
        TextSize = 8
    }):Play()
    TweenService:Create(introDuels, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        Position = UDim2.new(2, 0, 0.5, -50),
        TextSize = 8
    }):Play()
    TweenService:Create(introSub, TweenInfo.new(0.4), {TextTransparency = 1}):Play()
    TweenService:Create(introLine, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        Size = UDim2.new(0, 0, 0, 2),
        Position = UDim2.new(0.5, 0, 0.5, 26),
    }):Play()
    task.wait(0.45)
    pcall(function() introSound:Stop() end)
    introGui:Destroy()
end)

-- Initialize (runs right away, behind the intro overlay)
rebuildUI()
loadConfigData()
rebuildQuickButtons()
applyGuiFromConfig()
setupHotkeys()

-- Auto-enable grab on start
task.defer(function()
    Steal.AutoStealEnabled = true
    pcall(startAutoSteal)
    if setInstaGrab then setInstaGrab(true) end
end)

-- ========== AUTO STEAL PROGRESS WINDOW (fearduels_4 style) ==========
local stealProgressGui = (function()
    local SPG = {}
    local _oldSPG = LP.PlayerGui:FindFirstChild("StealProgressScreenGui")
    if _oldSPG then _oldSPG:Destroy() end
    local spScreenGui = Instance.new("ScreenGui")
    spScreenGui.Name = "StealProgressScreenGui"
    spScreenGui.ResetOnSpawn = false
    spScreenGui.DisplayOrder = 200
    spScreenGui.IgnoreGuiInset = true
    spScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    spScreenGui.Parent = LP.PlayerGui

    -- Main container — taller to fit FPS/ping row
    local spFrame = Instance.new("Frame", spScreenGui)
    spFrame.Name = "StealProgressGui"
    spFrame.Size = UDim2.new(0, 200, 0, 46)
    spFrame.Position = UDim2.new(0.5, -100, 1, -60)
    spFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
    spFrame.BackgroundTransparency = 0.05
    spFrame.BorderSizePixel = 0
    spFrame.Active = true
    spFrame.Visible = true
    spFrame.ZIndex = 300
    spFrame.ClipsDescendants = true

    local function mkC(p,r) local c=Instance.new("UICorner",p); c.CornerRadius=UDim.new(0,r or 8); return c end
    local function mkS(p,col,th) local s=Instance.new("UIStroke",p); s.Color=col; s.Thickness=th or 1; s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; return s end
    mkC(spFrame, 0); mkS(spFrame, Color3.fromRGB(60, 60, 70), 1.5)

    -- Background image
    local spBgImg = Instance.new("ImageLabel", spFrame)
    spBgImg.Size = UDim2.new(1, 0, 1, 0)
    spBgImg.Position = UDim2.new(0, 0, 0, 0)
    spBgImg.BackgroundTransparency = 1
    spBgImg.Image = "rbxassetid://112977078041259"
    spBgImg.ScaleType = Enum.ScaleType.Crop
    spBgImg.ImageTransparency = 0.45
    spBgImg.ZIndex = 299
    mkC(spBgImg, 0)

    -- Draggable
    local spDragging, spDragStart, spStartPos = false, nil, nil
    spFrame.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            spDragging = true; spDragStart = inp.Position; spStartPos = spFrame.Position
            inp.Changed:Connect(function() if inp.UserInputState == Enum.UserInputState.End then spDragging = false end end)
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not spDragging then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - spDragStart
            spFrame.Position = UDim2.new(spStartPos.X.Scale, spStartPos.X.Offset + d.X, spStartPos.Y.Scale, spStartPos.Y.Offset + d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then spDragging = false end
    end)

    local spPct = Instance.new("TextLabel", spFrame)
    spPct.Size = UDim2.new(0.4, 0, 0, 22)
    spPct.Position = UDim2.new(0, 12, 0, 5)
    spPct.BackgroundTransparency = 1
    spPct.Text = "0%"
    spPct.TextColor3 = Color3.fromRGB(255, 255, 255)
    spPct.Font = Enum.Font.GothamBlack
    spPct.TextSize = 14
    spPct.TextXAlignment = Enum.TextXAlignment.Left
    spPct.ZIndex = 302

    local spRadiusLbl = Instance.new("TextLabel", spFrame)
    spRadiusLbl.Size = UDim2.new(0.58, 0, 0, 22)
    spRadiusLbl.Position = UDim2.new(0.42, 0, 0, 5)
    spRadiusLbl.BackgroundTransparency = 1
    spRadiusLbl.Text = "Radius: " .. tostring(Steal.StealRadius)
    spRadiusLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    spRadiusLbl.Font = Enum.Font.GothamBold
    spRadiusLbl.TextSize = 12
    spRadiusLbl.TextXAlignment = Enum.TextXAlignment.Right
    spRadiusLbl.ZIndex = 302

    -- FPS / Ping row
    local spFpsLbl = Instance.new("TextLabel", spFrame)
    spFpsLbl.Size = UDim2.new(0.5, 0, 0, 14)
    spFpsLbl.Position = UDim2.new(0, 12, 0, 24)
    spFpsLbl.BackgroundTransparency = 1
    spFpsLbl.Text = "FPS: --"
    spFpsLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    spFpsLbl.Font = Enum.Font.GothamBold
    spFpsLbl.TextSize = 11
    spFpsLbl.TextXAlignment = Enum.TextXAlignment.Left
    spFpsLbl.ZIndex = 302

    local spPingLbl = Instance.new("TextLabel", spFrame)
    spPingLbl.Size = UDim2.new(0.5, -14, 0, 14)
    spPingLbl.Position = UDim2.new(0.5, 0, 0, 24)
    spPingLbl.BackgroundTransparency = 1
    spPingLbl.Text = "PING: --"
    spPingLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    spPingLbl.Font = Enum.Font.GothamBold
    spPingLbl.TextSize = 11
    spPingLbl.TextXAlignment = Enum.TextXAlignment.Right
    spPingLbl.ZIndex = 302

    -- High ping warning overlay
    local pingWarnGui = Instance.new("ScreenGui", LP.PlayerGui)
    pingWarnGui.Name = "PingWarnOverlay"
    pingWarnGui.ResetOnSpawn = false
    pingWarnGui.DisplayOrder = 9999
    pingWarnGui.IgnoreGuiInset = true

    local pingWarnFrame = Instance.new("Frame", pingWarnGui)
    pingWarnFrame.Size = UDim2.new(1, 0, 1, 0)
    pingWarnFrame.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
    pingWarnFrame.BackgroundTransparency = 1
    pingWarnFrame.BorderSizePixel = 0
    pingWarnFrame.ZIndex = 9999
    pingWarnFrame.Visible = false

    local pingWarnLbl = Instance.new("TextLabel", pingWarnFrame)
    pingWarnLbl.Size = UDim2.new(1, 0, 0, 60)
    pingWarnLbl.Position = UDim2.new(0, 0, 0.5, -30)
    pingWarnLbl.BackgroundTransparency = 1
    pingWarnLbl.Text = "⚠ WARNING HIGH PING ⚠"
    pingWarnLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    pingWarnLbl.Font = Enum.Font.GothamBlack
    pingWarnLbl.TextSize = 36
    pingWarnLbl.TextStrokeTransparency = 0
    pingWarnLbl.TextStrokeColor3 = Color3.fromRGB(180, 0, 0)
    pingWarnLbl.ZIndex = 10000

    local _pingWarnDone = false
    local function triggerPingWarning()
        if _pingWarnDone then return end
        _pingWarnDone = true
        pingWarnFrame.Visible = true
        task.spawn(function()
            local endTime = tick() + 3
            while tick() < endTime do
                TweenService:Create(pingWarnFrame, TweenInfo.new(0.25), {BackgroundTransparency = 0.6}):Play()
                task.wait(0.25)
                TweenService:Create(pingWarnFrame, TweenInfo.new(0.25), {BackgroundTransparency = 0.85}):Play()
                task.wait(0.25)
            end
            pingWarnFrame.Visible = false
            pingWarnFrame.BackgroundTransparency = 1
        end)
    end

    -- FPS/Ping update loop
    do
        local _spFpsTick = tick(); local _spFc = 0
        RunService.RenderStepped:Connect(function()
            _spFc = _spFc + 1
            local now = tick()
            if now - _spFpsTick >= 0.5 then
                local fps = math.round(_spFc / (now - _spFpsTick))
                _spFc = 0; _spFpsTick = now
                local ping = math.round(LP:GetNetworkPing() * 1000)
                spFpsLbl.Text = "FPS: " .. fps
                spPingLbl.Text = "PING: " .. ping .. "ms"
                if ping >= 100 then
                    triggerPingWarning()
                end
            end
        end)
    end

    -- Progress bar track (very thin: 3px)
    local spTrackBg = Instance.new("Frame", spFrame)
    spTrackBg.Size = UDim2.new(1, -20, 0, 2)
    spTrackBg.Position = UDim2.new(0, 10, 1, -8)
    spTrackBg.BackgroundColor3 = Color3.fromRGB(50, 50, 58)
    spTrackBg.BorderSizePixel = 0
    spTrackBg.ZIndex = 301
    mkC(spTrackBg, 2)

    -- Fill bar
    local spFill = Instance.new("Frame", spTrackBg)
    spFill.Size = UDim2.new(0, 0, 1, 0)
    spFill.BackgroundColor3 = Color3.fromRGB(210, 210, 225)
    spFill.BorderSizePixel = 0
    spFill.ZIndex = 302
    mkC(spFill, 2)

    local _lastPct = 0
    function SPG.update()
        if not spFrame.Visible then return end
        local pct = 0
        if State.isStealing and State.stealStartTime then
            local elapsed = tick() - State.stealStartTime
            pct = math.clamp(elapsed / math.max(Steal.StealDuration, 0.01), 0, 1)
        elseif Steal.AutoStealEnabled then
            local char = LP.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if root and findNearestPrompt() then pct = 1 else pct = 0 end
        end
        _lastPct = _lastPct + (pct - _lastPct) * 0.2
        local display = math.floor(_lastPct * 100)
        spFill.Size = UDim2.new(math.clamp(_lastPct, 0, 1), 0, 1, 0)
        spPct.Text = display .. "%"
        -- Keep radius label live so it reflects any slider changes
        spRadiusLbl.Text = "Radius: " .. tostring(Steal.StealRadius)
    end
    function SPG.setVisible(v) spFrame.Visible = v end
    function SPG.isVisible() return spFrame.Visible end
    return SPG
end)()
-- Character setup
local function setupChar(char)
    task.wait(0.1); resetBaseSide(); originalAnims = nil
    detectBaseSideAsync(function(side)
        if State.brainrotReturnEnabled then State.brainrotReturnSide = (side == "right") and "left" or "right" end
    end)
    h = char:WaitForChild("Humanoid",5)
    hrp = char:WaitForChild("HumanoidRootPart",5)
    if not h or not hrp then return end
    State.lastKnownHealth = h.Health
    local head = char:FindFirstChild("Head")
    if head then
        local old = head:FindFirstChild("SpeedBillboard")
        if old then old:Destroy() end
        local bb = Instance.new("BillboardGui", head)
        bb.Name = "SpeedBillboard"; bb.Size = UDim2.new(0,140,0,25)
        bb.StudsOffset = Vector3.new(0,3,0); bb.AlwaysOnTop = true
        bb.ResetOnSpawn = false
        speedLbl = Instance.new("TextLabel", bb)
        speedLbl.Size = UDim2.new(1,0,0,25); speedLbl.BackgroundTransparency = 1
        speedLbl.TextColor3 = C_ACCENT2; speedLbl.Font = Enum.Font.GothamBold
        speedLbl.TextScaled = true; speedLbl.TextStrokeTransparency = 0
    end
    if State.antiRagdollEnabled and not Conns.antiRag then task.wait(0.5); startAntiRagdoll() end
    if State.animEnabled then task.wait(0.3); saveOriginalAnims(char); applyAnimPack(char) end
    if State.medusaCounterEnabled then setupMedusa(char) end
    if State.desyncEnabled then task.wait(0.2); applyDesync(true) end
    if State.autoLeftEnabled then task.wait(0.5); startAutoLeft() end
    if State.autoRightEnabled then task.wait(0.5); startAutoRight() end
    if State.batAimbotEnabled then
        task.wait(0.5)
        startBatAimbot()
    end
    if State.tpDownEnabled then
        task.wait(0.5)
        startTpDown()
        if setTpDown then setTpDown(true) end
    end
    if State.dropBREnabled then
        task.wait(0.5)
        startDropBR()
        if setDropBR then setDropBR(true) end
    end
    if State.unwalkEnabled then
        if unwalkConn then unwalkConn:Disconnect(); unwalkConn = nil end
        _unwalkAnimations = {}
        task.wait(0.3)
        startUnwalk()
    end
    -- Auto TP will be handled by the heartbeat connection, just ensure it's running
    if State.autoTPEnabled then
        if autoTPConn then stopAutoTP() end
        startAutoTP()
    end
end
LP.CharacterAdded:Connect(setupChar)
if LP.Character then task.spawn(function() setupChar(LP.Character) end) end

-- ========== ENEMY SPEED TRACKER ==========
local enemySpeedBBs = {} -- [player] = {bb, lbl, lastHRP, smooth}

local function getOrCreateEnemyBB(player)
    local char = player.Character
    if not char then return nil end
    local head = char:FindFirstChild("Head")
    local hrp2 = char:FindFirstChild("HumanoidRootPart")
    if not head or not hrp2 then return nil end

    local existing = enemySpeedBBs[player]
    if existing and existing.lastHRP ~= hrp2 then
        pcall(function() existing.bb:Destroy() end)
        enemySpeedBBs[player] = nil
        existing = nil
    end

    if not existing then
        local old = head:FindFirstChild("EnemySpeedBillboard")
        if old then old:Destroy() end
        local bb = Instance.new("BillboardGui", head)
        bb.Name = "EnemySpeedBillboard"
        bb.Size = UDim2.new(0, 140, 0, 25)
        bb.StudsOffset = Vector3.new(0, 3.6, 0)
        bb.AlwaysOnTop = true
        bb.ResetOnSpawn = false
        local lbl = Instance.new("TextLabel", bb)
        lbl.Size = UDim2.new(1, 0, 1, 0)
        lbl.BackgroundTransparency = 1
        lbl.TextColor3 = Color3.fromRGB(0, 0, 0)
        lbl.Font = Enum.Font.GothamBold
        lbl.TextScaled = true
        lbl.TextStrokeTransparency = 0
        enemySpeedBBs[player] = {bb = bb, lbl = lbl, lastHRP = hrp2, smooth = 0}
        existing = enemySpeedBBs[player]
    end
    return existing
end

local function cleanupEnemyBB(player)
    local d = enemySpeedBBs[player]
    if d then pcall(function() d.bb:Destroy() end) end
    enemySpeedBBs[player] = nil
end

Players.PlayerRemoving:Connect(cleanupEnemyBB)

RunService.Heartbeat:Connect(function()
    for _, p in ipairs(Players:GetPlayers()) do
        if p == LP then continue end
        local char = p.Character
        local hrp2 = char and char:FindFirstChild("HumanoidRootPart")
        if hrp2 then
            local d = getOrCreateEnemyBB(p)
            if d and d.lbl then
                local raw = Vector3.new(hrp2.AssemblyLinearVelocity.X, 0, hrp2.AssemblyLinearVelocity.Z).Magnitude
                d.smooth = (d.smooth or raw) * 0.6 + raw * 0.4
                d.lbl.Text = string.format("%.1f", d.smooth)
            end
        else
            cleanupEnemyBB(p)
        end
    end
end)
-- ========== FEAR USER TAG ==========
-- Mark this client so other Fear users can detect it
_G._fearUserTag = true

local FEAR_TAG_NAME = "FearUserTag"
local FEAR_TAG_COLOR = _activeTheme and _activeTheme.accent2 or Color3.fromRGB(180, 180, 200)

local function applyFearTag(char)
    if not char then return end
    local head = char:WaitForChild("Head", 5)
    if not head then return end
    local old = head:FindFirstChild(FEAR_TAG_NAME)
    if old then old:Destroy() end

    local bb = Instance.new("BillboardGui", head)
    bb.Name = FEAR_TAG_NAME
    bb.Size = UDim2.new(0, 240, 0, 44)
    bb.StudsOffset = Vector3.new(0, 2.6, 0)
    bb.AlwaysOnTop = false
    bb.ResetOnSpawn = false

    local lbl = Instance.new("TextLabel", bb)
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = "✦ /fearduels ✦"
    lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    lbl.Font = Enum.Font.GothamBlack
    lbl.TextSize = 20
    lbl.TextStrokeTransparency = 0
    lbl.TextStrokeColor3 = Color3.fromRGB(180, 180, 180)
    lbl.TextScaled = false

    -- Shiny white pulse effect
    task.spawn(function()
        local t = 0
        while lbl and lbl.Parent do
            t = t + 0.08
            local shine = 0.85 + math.sin(t * 3) * 0.15
            local v = math.floor(255 * shine)
            lbl.TextColor3 = Color3.fromRGB(v, v, v)
            lbl.TextStrokeColor3 = Color3.fromRGB(
                math.floor(160 * shine),
                math.floor(160 * shine),
                math.floor(160 * shine)
            )
            task.wait(0.03)
        end
    end)
end

local function removeFearTag(char)
    if not char then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local tag = head:FindFirstChild(FEAR_TAG_NAME)
    if tag then tag:Destroy() end
end

local function checkAndTagPlayer(player)
    -- Poll _G._fearUserTag on the player's client via a shared _G check
    -- Since _G is per-VM this only works for players in the same executor context;
    -- we use an Attribute as a cross-client signal instead
    task.spawn(function()
        local char = player.Character or player.CharacterAdded:Wait()
        -- Give their script a moment to set the attribute
        task.wait(1.5)
        if player:GetAttribute("FearUser") then
            applyFearTag(char)
        end
        -- Also watch for it appearing later
        player:GetAttributeChangedSignal("FearUser"):Connect(function()
            if player:GetAttribute("FearUser") then
                applyFearTag(player.Character)
            else
                removeFearTag(player.Character)
            end
        end)
        -- Re-apply on respawn
        player.CharacterAdded:Connect(function(newChar)
            task.wait(1.5)
            if player:GetAttribute("FearUser") then
                applyFearTag(newChar)
            end
        end)
    end)
end

-- Broadcast that this local player is a Fear user via a replicated Attribute
LP:SetAttribute("FearUser", true)
-- Apply tag to own character too
LP.CharacterAdded:Connect(function(char)
    task.spawn(function()
        task.wait(0.5)
        applyFearTag(char)
    end)
end)
-- Self-apply with retry loop in case character isnt ready yet (Luarmor timing)
task.spawn(function()
    for _ = 1, 10 do
        local char = LP.Character
        if char then
            local head = char:FindFirstChild("Head")
            if head and not head:FindFirstChild(FEAR_TAG_NAME) then
                applyFearTag(char)
                break
            end
        end
        task.wait(0.5)
    end
end)

-- Check all current and future players
for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LP then checkAndTagPlayer(p) end
end
Players.PlayerAdded:Connect(function(p)
    if p ~= LP then checkAndTagPlayer(p) end
end)
-- ========== FPS & PING GUI (removed) ==========
-- Infinite Jump
UIS.JumpRequest:Connect(function()
    if not State.infJumpEnabled then return end
    if State.infJumpMode ~= "manual" then return end
    local char = LP.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then
        local hum = char:FindFirstChildOfClass("Humanoid")
        local heldTool = nil
        for _, v in ipairs(char:GetChildren()) do
            if v:IsA("Tool") then heldTool = v break end
        end
        root.Velocity = Vector3.new(root.Velocity.X, State.jumpVelocity, root.Velocity.Z)
        if heldTool and hum then
            task.defer(function()
                if heldTool and heldTool.Parent ~= char then
                    hum:EquipTool(heldTool)
                end
            end)
        end
    end
end)
-- Movement (RenderStepped for smooth visuals)
local lastSpeed = 0
local _speedLblTick = 0
local _speedSmooth = 0
RunService.RenderStepped:Connect(function()
    if not (h and hrp) then return end
    if State._tplnProgress then return end
    if not State.autoLeftEnabled and not State.autoRightEnabled and not State.batAimbotEnabled then
        local md = h.MoveDirection
        local targetSpd
        if State.laggerModeEnabled then targetSpd = State.laggerSpeed
        elseif State.oxgModeEnabled then targetSpd = State.oxgSpeed
        elseif State.carrySpeedActive then targetSpd = State.carrySpeed
        else local useCarry = getMobileCarryState(); targetSpd = useCarry and State.carrySpeed or State.normalSpeed end
        local smoothFactor = 0.85
        lastSpeed = lastSpeed * smoothFactor + targetSpd * (1 - smoothFactor)
        local spd = lastSpeed
        if md.Magnitude > 0 then
            State.lastMoveDir = md
            hrp.Velocity = Vector3.new(md.X * spd, hrp.Velocity.Y, md.Z * spd)
        end
    end
    -- Speed label: rolling average of last 3 frames for stable but accurate reading
    if speedLbl then
        local vel = hrp.AssemblyLinearVelocity
        local raw = Vector3.new(vel.X, 0, vel.Z).Magnitude
        _speedSmooth = (_speedSmooth or raw) * 0.6 + raw * 0.4
        speedLbl.Text = string.format("%.1f", _speedSmooth)
    end
    -- Steal progress floating window update
    pcall(function() stealProgressGui.update() end)
end)
-- ========== MASTER HEARTBEAT (consolidated, replaces 3 separate always-on connections) ==========
RunService.Heartbeat:Connect(function()
    -- Inf jump
    if State.infJumpEnabled and State.infJumpMode == "hold" then
        local char = LP.Character
        if char then
            local root = char:FindFirstChild("HumanoidRootPart")
            local hum2 = char:FindFirstChildOfClass("Humanoid")
            if root and hum2 then
                local jumpHeld = UIS:IsKeyDown(Enum.KeyCode.Space)
                    or UIS:IsKeyDown(Enum.KeyCode.ButtonA)
                    or (hum2 and hum2.Jump)
                -- only boost when falling or near ground, not while still rising
                if jumpHeld and root.Velocity.Y < 38 then
                    root.Velocity = Vector3.new(root.Velocity.X, 38, root.Velocity.Z)
                end
            end
        end
    end
    -- Brainrot return
    if State.brainrotReturnEnabled and not State.brainrotReturnCooldown then
        local char = LP.Character
        if char then
            local hum2 = char:FindFirstChildOfClass("Humanoid")
            if hum2 then
                local cur = hum2.Health
                local wasHit = cur < State.lastKnownHealth - 5
                local st = hum2:GetState()
                local isRag = (st == Enum.HumanoidStateType.Physics or st == Enum.HumanoidStateType.Ragdoll or st == Enum.HumanoidStateType.FallingDown)
                State.lastKnownHealth = cur
                if wasHit or isRag then
                    if State.brainrotReturnSide == "left" then doReturnTeleport(POS.RETURN_L)
                    elseif State.brainrotReturnSide == "right" then doReturnTeleport(POS.RETURN_R)
                    elseif not State._sideDetecting then
                        State._sideDetecting = true
                        detectBaseSideAsync(function(s)
                            State.brainrotReturnSide = (s == "right") and "left" or "right"
                            State._sideDetecting = false
                            doReturnTeleport(State.brainrotReturnSide == "left" and POS.RETURN_L or POS.RETURN_R)
                        end)
                    end
                end
            end
        end
    end
end)
detectBaseSideAsync(function(side)
    if State.brainrotReturnEnabled then State.brainrotReturnSide = (side == "right") and "left" or "right" end
end)


-- Print hotkey guide
print("========================================")
print("[Fear Duel] LOADED - CUSTOM BINDS ACTIVE")
print("========================================")
print("Go to Settings tab to assign keyboard")
print("and controller binds for each action.")
print("Binds save automatically.")
print("========================================")
