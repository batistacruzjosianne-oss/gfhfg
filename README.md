-- AraDuels (BLUE THEME)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
local UIS = game:GetService("UserInputService")

-- ----------------------------------------------------------------
--  Colors & State
-- ----------------------------------------------------------------
local State = {
    antiBatActive = false,
    infJumpActive = false,
    antiBatThread = nil,
    infJumpThread = nil,
    guiVisible = true,
    movementBlocked = false
}

local C = {
    bg = Color3.fromRGB(15, 25, 45),
    card = Color3.fromRGB(18, 18, 22),
    cardActive1 = Color3.fromRGB(20, 35, 55),
    cardActive2 = Color3.fromRGB(45, 25, 35),
    border = Color3.fromRGB(32, 32, 38),
    text = Color3.fromRGB(255, 255, 255),
    textSub = Color3.fromRGB(150, 150, 160),
    accent1 = Color3.fromRGB(100, 200, 255),
    accent2 = Color3.fromRGB(255, 45, 85),
    pillOff = Color3.fromRGB(28, 28, 32),
    dotOff = Color3.fromRGB(80, 80, 85),
    watermark = Color3.fromRGB(120, 200, 255)
}

-- ----------------------------------------------------------------
--  Keybinds
-- ----------------------------------------------------------------
local antiBatBind = { type = "keyboard", value = Enum.KeyCode.Z }  -- MUDOU PARA Z
local infJumpBind = { type = "keyboard", value = Enum.KeyCode.I }
local guiToggleBind = { type = "keyboard", value = Enum.KeyCode.LeftControl }

-- ----------------------------------------------------------------
--  Config file
-- ----------------------------------------------------------------
local CONFIG_FILE = "AraDuels_config.json"

local shortToFullGamepad = {
    A = "ButtonA", B = "ButtonB", X = "ButtonX", Y = "ButtonY",
    LB = "ButtonL1", RB = "ButtonR1", LT = "ButtonL2", RT = "ButtonR2",
    START = "ButtonStart", SELECT = "ButtonSelect",
    L3 = "LeftThumbstick", R3 = "RightThumbstick",
    ["DPAD UP"] = "DirectionalPadUp",
    ["DPAD DOWN"] = "DirectionalPadDown",
    ["DPAD LEFT"] = "DirectionalPadLeft",
    ["DPAD RIGHT"] = "DirectionalPadRight"
}

local function getKeyName(keyCode)
    local name = tostring(keyCode):gsub("Enum.KeyCode.", "")
    if name == "LeftControl" then return "LCTRL"
    elseif name == "RightControl" then return "RCTRL"
    elseif name == "LeftShift" then return "LSHIFT"
    elseif name == "RightShift" then return "RSHIFT"
    elseif name == "LeftAlt" then return "LALT"
    elseif name == "RightAlt" then return "RALT"
    elseif name == "ButtonA" then return "A"
    elseif name == "ButtonB" then return "B"
    elseif name == "ButtonX" then return "X"
    elseif name == "ButtonY" then return "Y"
    elseif name == "ButtonL1" then return "LB"
    elseif name == "ButtonR1" then return "RB"
    elseif name == "ButtonL2" then return "LT"
    elseif name == "ButtonR2" then return "RT"
    elseif name == "ButtonStart" then return "START"
    elseif name == "ButtonSelect" then return "SELECT"
    elseif name == "LeftThumbstick" then return "L3"
    elseif name == "RightThumbstick" then return "R3"
    elseif name == "DirectionalPadUp" then return "DPAD UP"
    elseif name == "DirectionalPadDown" then return "DPAD DOWN"
    elseif name == "DirectionalPadLeft" then return "DPAD LEFT"
    elseif name == "DirectionalPadRight" then return "DPAD RIGHT"
    end
    if #name > 6 then name = name:sub(1,6) end
    return name:upper()
end

local function saveConfig()
    local data = {
        antiBatType = antiBatBind.type,
        antiBatValue = tostring(antiBatBind.value):gsub("Enum.KeyCode.", ""),
        infJumpType = infJumpBind.type,
        infJumpValue = tostring(infJumpBind.value):gsub("Enum.KeyCode.", ""),
        guiToggleType = guiToggleBind.type,
        guiToggleValue = tostring(guiToggleBind.value):gsub("Enum.KeyCode.", "")
    }
    for attempt = 1, 3 do
        local ok = pcall(function()
            if writefile then
                writefile(CONFIG_FILE, HttpService:JSONEncode(data))
                task.wait(0.1)
                local verify = readfile(CONFIG_FILE)
                if verify and verify ~= "" then
                    return true
                end
            end
            return false
        end)
        if ok then break end
        task.wait(0.2)
    end
end

local function loadConfig()
    local raw, err = pcall(function() return readfile and readfile(CONFIG_FILE) end)
    if not raw or not err or err == "" then return end
    local ok, cfg = pcall(HttpService.JSONDecode, HttpService, err)
    if not ok or not cfg then
        pcall(function() if deletefile then deletefile(CONFIG_FILE) end end)
        return
    end

    local function getEnum(valueStr, typ)
        if typ == "controller" then
            local full = Enum.KeyCode[valueStr]
            if full and full ~= Enum.KeyCode.Unknown then
                return full
            end
            local mapped = shortToFullGamepad[valueStr]
            if mapped then
                return Enum.KeyCode[mapped]
            end
        else
            return Enum.KeyCode[valueStr]
        end
        return nil
    end

    local ab = getEnum(cfg.antiBatValue, cfg.antiBatType)
    if ab then
        antiBatBind.type = cfg.antiBatType
        antiBatBind.value = ab
    end
    local ij = getEnum(cfg.infJumpValue, cfg.infJumpType)
    if ij then
        infJumpBind.type = cfg.infJumpType
        infJumpBind.value = ij
    end
    local gt = getEnum(cfg.guiToggleValue, cfg.guiToggleType)
    if gt then
        guiToggleBind.type = cfg.guiToggleType
        guiToggleBind.value = gt
    end
end
pcall(loadConfig)

-- ----------------------------------------------------------------
--  BLOQUEIO DE MOVIMENTO (Mobile + PC)
-- ----------------------------------------------------------------
local movementConnections = {}
local mobileButtons = {}

-- Bloqueia movimento no PC (WASD)
local function blockPCMovement(blocked)
    if blocked then
        -- Cria um contexto de entrada falso para bloquear
        local context = Instance.new("ContextActionService")
        -- Não precisa de contexto, só bloquear as teclas
    end
end

-- Bloqueia movimento no Mobile (botões virtuais)
local function blockMobileMovement(blocked)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return end
    
    for _, btn in ipairs(pg:GetDescendants()) do
        if btn:IsA("GuiButton") and btn.Name and string.find(btn.Name, "Move") then
            if blocked then
                btn.Active = false
                btn.Visible = false
            else
                btn.Active = true
                btn.Visible = true
            end
        end
    end
end

-- Bloqueia TODAS as entradas de movimento
local function blockAllMovement(blocked)
    State.movementBlocked = blocked
    
    if blocked then
        -- Bloqueia teclas WASD no PC
        local keys = {Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D}
        for _, key in ipairs(keys) do
            local conn = UIS.InputBegan:Connect(function(inp, gpe)
                if gpe then return end
                if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == key then
                    -- Cancela o input
                    inp:Consume()
                end
            end)
            table.insert(movementConnections, conn)
        end
        
        -- Bloqueia joystick virtual no Mobile
        blockMobileMovement(true)
        
        -- Bloqueia movimento do teclado (método alternativo)
        blockPCMovement(true)
        
    else
        -- Remove bloqueios do PC
        for _, conn in ipairs(movementConnections) do
            conn:Disconnect()
        end
        movementConnections = {}
        
        -- Libera Mobile
        blockMobileMovement(false)
        blockPCMovement(false)
    end
end

-- ----------------------------------------------------------------
--  Anti Bat - COM BLOQUEIO DE MOVIMENTO
-- ----------------------------------------------------------------
local antiBatStatus, antiBatSwitchBall, antiBatRow, antiBatRowStroke
local antiBatKeyBtn, antiBatKeyBtnStroke

local function toggleAntiBat()
    State.antiBatActive = not State.antiBatActive
    
    -- BLOQUEIA OU LIBERA O MOVIMENTO
    blockAllMovement(State.antiBatActive)
    
    if antiBatStatus then
        antiBatStatus.Text = State.antiBatActive and "STATUS: ACTIVE" or "STATUS: DISABLED"
        antiBatStatus.TextColor3 = State.antiBatActive and C.accent1 or C.textSub
    end
    if antiBatRow and antiBatRowStroke then
        TweenService:Create(antiBatRow, TweenInfo.new(0.2), {
            BackgroundColor3 = State.antiBatActive and C.cardActive1 or C.card
        }):Play()
        TweenService:Create(antiBatRowStroke, TweenInfo.new(0.2), {
            Color = State.antiBatActive and C.accent1 or C.border
        }):Play()
    end
    if antiBatSwitchBall then
        TweenService:Create(antiBatSwitchBall, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Position = State.antiBatActive and UDim2.new(1, -15, 0.5, -5) or UDim2.new(0, 5, 0.5, -5),
            BackgroundColor3 = State.antiBatActive and C.text or C.dotOff,
        }):Play()
    end
    
    if State.antiBatActive then
        if State.antiBatThread then State.antiBatThread:Disconnect() end
        State.antiBatThread = RunService.Heartbeat:Connect(function()
            local char = LocalPlayer.Character
            if not char then return end
            local hum = char:FindFirstChildOfClass("Humanoid")
            local hrp = char:FindFirstChild("HumanoidRootPart")
            if not hum or not hrp then return end
            
            -- Velocidade 2491.41 na direção que está olhando
            local dir = hum.MoveDirection
            if dir.Magnitude <= 0 then
                dir = hrp.CFrame.LookVector
            end
            hrp.Velocity = Vector3.new(dir.X * 2491.41, hrp.Velocity.Y, dir.Z * 2491.41)
        end)
    else
        if State.antiBatThread then
            State.antiBatThread:Disconnect()
            State.antiBatThread = nil
        end
    end
end

-- ----------------------------------------------------------------
--  Infinite Jump
-- ----------------------------------------------------------------
local infJumpStatus, infJumpSwitchBall, infJumpRow, infJumpRowStroke
local infJumpKeyBtn, infJumpKeyBtnStroke

local jumpHeld = false
local lastJumpBoostTime = 0
local JUMP_BOOST_INTERVAL = 0.05

-- Mobile jump button hook
task.spawn(function()
    local pg = LocalPlayer:WaitForChild("PlayerGui", 10)
    if pg then
        local function hookJumpButton(btn)
            if btn:IsA("GuiButton") and btn.Name == "JumpButton" and not btn:GetAttribute("InfJumpHooked") then
                btn:SetAttribute("InfJumpHooked", true)
                btn.MouseButton1Down:Connect(function()
                    if State.infJumpActive then jumpHeld = true end
                end)
                btn.MouseButton1Up:Connect(function() jumpHeld = false end)
                btn.MouseLeave:Connect(function() jumpHeld = false end)
            end
        end
        for _, d in ipairs(pg:GetDescendants()) do hookJumpButton(d) end
        pg.DescendantAdded:Connect(hookJumpButton)
    end
end)

UserInputService.JumpRequest:Connect(function()
    if State.infJumpActive then
        jumpHeld = true
        task.wait(0.05)
        jumpHeld = false
    end
end)

UserInputService.InputBegan:Connect(function(inp, gpe)
    if gpe then return end
    if State.infJumpActive and inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == Enum.KeyCode.Space then
        jumpHeld = true
    end
end)
UserInputService.InputEnded:Connect(function(inp, gpe)
    if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == Enum.KeyCode.Space then
        jumpHeld = false
    end
end)

local function startInfJumpLoop()
    if State.infJumpThread then State.infJumpThread:Disconnect() end
    State.infJumpThread = RunService.Stepped:Connect(function()
        if not State.infJumpActive then return end
        if not jumpHeld then return end
        local now = tick()
        if now - lastJumpBoostTime < JUMP_BOOST_INTERVAL then return end
        lastJumpBoostTime = now
        local char = LocalPlayer.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hum or not hrp or hum.Health <= 0 then return end
        local vel = hrp.AssemblyLinearVelocity
        if vel.Y < 55 then
            hrp.AssemblyLinearVelocity = Vector3.new(vel.X, 65, vel.Z)
        end
    end)
end

local function stopInfJumpLoop()
    if State.infJumpThread then
        State.infJumpThread:Disconnect()
        State.infJumpThread = nil
    end
    jumpHeld = false
    lastJumpBoostTime = 0
end

local function toggle
