-- AraDuels (BLUE THEME) - COM MENU ANTI ANTI DESYNC
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
    antiBatThread = nil,
    guiVisible = true
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
local antiBatBind = { type = "keyboard", value = Enum.KeyCode.Z }
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

-- Bloqueia Mobile
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
    if blocked then
        -- Bloqueia teclas WASD no PC
        local keys = {Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D}
        for _, key in ipairs(keys) do
            local conn = UIS.InputBegan:Connect(function(inp, gpe)
                if gpe then return end
                if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == key then
                    inp:Consume()
                end
            end)
            table.insert(movementConnections, conn)
        end
        
        -- Bloqueia joystick virtual no Mobile
        blockMobileMovement(true)
        
    else
        -- Remove bloqueios do PC
        for _, conn in ipairs(movementConnections) do
            conn:Disconnect()
        end
        movementConnections = {}
        
        -- Libera Mobile
        blockMobileMovement(false)
    end
end

-- ----------------------------------------------------------------
--  Anti Bat
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
--  FUNÇÃO PARA CRIAR GUI (estilo Anti Anti Desync)
-- ----------------------------------------------------------------
local function getGuiParent()
    if typeof(gethui) == "function" then
        local success, result = pcall(gethui)
        if success and result then return result end
    end
    local success, coreGui = pcall(function() return game:GetService("CoreGui") end)
    if success and coreGui then return coreGui end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local function createInstance(className, properties, parent)
    local instance = Instance.new(className)
    for property, value in pairs(properties or {}) do
        instance[property] = value
    end
    instance.Parent = parent
    return instance
end

local function addCorner(object, radius)
    return createInstance("UICorner", { CornerRadius = UDim.new(0, radius) }, object)
end

-- ----------------------------------------------------------------
--  CRIAR GUI ESTILO ANTI ANTI DESYNC
-- ----------------------------------------------------------------
local parentGui = getGuiParent()

-- Remove GUI antiga se existir
if parentGui:FindFirstChild("LarpAntiAntiDesyncGui") then
    parentGui.LarpAntiAntiDesyncGui:Destroy()
end

local screenGui = createInstance("ScreenGui", {
    Name = "LarpAntiAntiDesyncGui",
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, parentGui)

-- Main Frame
local mainFrame = createInstance("Frame", {
    Size = UDim2.fromOffset(240, 130),
    Position = UDim2.new(0.5, -120, 0.5, -65),
    BackgroundColor3 = Color3.fromRGB(220, 220, 220),
    BorderSizePixel = 0,
    ClipsDescendants = true,
    Active = true,
    ZIndex = 1,
}, screenGui)
addCorner(mainFrame, 18)

-- Background Image
createInstance("ImageLabel", {
    Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1,
    Image = "rbxassetid://98596557474777",
    ScaleType = Enum.ScaleType.Crop,
    ZIndex = 2,
}, mainFrame)

-- Title Text
createInstance("TextLabel", {
    Size = UDim2.new(1, -70, 0, 30),
    Position = UDim2.fromOffset(12, 6),
    BackgroundTransparency = 1,
    Text = "Anti Anti Desync",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 14,
    TextXAlignment = Enum.TextXAlignment.Left,
    ZIndex = 3,
}, mainFrame)

-- Control Buttons
local minBtn = createInstance("TextButton", {
    Size = UDim2.fromOffset(24, 24),
    Position = UDim2.new(1, -56, 0, 8),
    BackgroundColor3 = Color3.fromRGB(18, 18, 18),
    Text = "-",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 16,
    AutoButtonColor = false,
    ZIndex = 3,
}, mainFrame)
addCorner(minBtn, 9)

local closeBtn = createInstance("TextButton", {
    Size = UDim2.fromOffset(24, 24),
    Position = UDim2.new(1, -28, 0, 8),
    BackgroundColor3 = Color3.fromRGB(18, 18, 18),
    Text = "X",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 13,
    AutoButtonColor = false,
    ZIndex = 3,
}, mainFrame)
addCorner(closeBtn, 9)

-- Activate Button
local toggleButton = createInstance("TextButton", {
    Size = UDim2.new(1, -24, 0, 36),
    Position = UDim2.fromOffset(12, 40),
    BackgroundColor3 = Color3.fromRGB(18, 18, 18),
    Text = "ACTIVATE",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 15,
    AutoButtonColor = false,
    ZIndex = 3,
}, mainFrame)
addCorner(toggleButton, 12)

-- Keybind Section
createInstance("TextLabel", {
    Size = UDim2.fromOffset(80, 24),
    Position = UDim2.fromOffset(12, 90),
    BackgroundTransparency = 1,
    Text = "KEYBIND",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 13,
    TextXAlignment = Enum.TextXAlignment.Left,
    ZIndex = 3,
}, mainFrame)

local keybindBox = createInstance("TextBox", {
    Size = UDim2.fromOffset(42, 28),
    Position = UDim2.new(1, -54, 0, 88),
    BackgroundColor3 = Color3.fromRGB(18, 18, 18),
    Text = "Z",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 14,
    ClearTextOnFocus = false,
    ZIndex = 3,
}, mainFrame)
addCorner(keybindBox, 10)

-- ----------------------------------------------------------------
--  LOGICA DO ANTI ANTI DESYNC (Freeze outros players)
-- ----------------------------------------------------------------
local freezeConnections = {}
local freezeActive = false

local function toggleFreeze(isEnabled)
    freezeActive = isEnabled
    
    if isEnabled then
        local connection = RunService.Stepped:Connect(function()
            if not freezeActive then
                connection:Disconnect()
                return
            end
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    local character = player.Character
                    if character then
                        local humanoid = character:FindFirstChildWhichIsA("Humanoid")
                        if humanoid then
                            humanoid.WalkSpeed = 0
                            humanoid.JumpPower = 0
                            humanoid.AutoRotate = false
                        end
                        for _, object in ipairs(character:GetDescendants()) do
                            if object:IsA("BasePart") then
                                object.CanCollide = false
                            end
                        end
                    end
                end
            end
        end)
        table.insert(freezeConnections, connection)
    else
        freezeActive = false
        for _, connection in ipairs(freezeConnections) do
            if typeof(connection) == "RBXScriptConnection" then
                connection:Disconnect()
            end
        end
        freezeConnections = {}
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local character = player.Character
                if character then
                    local humanoid = character:FindFirstChildWhichIsA("Humanoid")
                    if humanoid then
                        humanoid.WalkSpeed = 16
                        humanoid.JumpPower = 50
                        humanoid.AutoRotate = true
                    end
                end
            end
        end
    end
end

-- ----------------------------------------------------------------
--  LOGICA DOS BOTOES
-- ----------------------------------------------------------------
local isActive = false
local boundKey = Enum.KeyCode.Z

local function updateButtonState(state)
    isActive = state
    if state then
        toggleButton.Text = "DEACTIVATE"
        toggleButton.BackgroundColor3 = Color3.fromRGB(200, 30, 30)
        toggleFreeze(true)
        toggleAntiBat() -- Ativa o Anti-Bat também
    else
        toggleButton.Text = "ACTIVATE"
        toggleButton.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
        toggleFreeze(false)
        if State.antiBatActive then
            toggleAntiBat() -- Desativa o Anti-Bat
        end
    end
end

toggleButton.Activated:Connect(function()
    updateButtonState(not isActive)
end)

closeBtn.Activated:Connect(function()
    updateButtonState(false)
    if State.antiBatActive then
        toggleAntiBat()
    end
    screenGui:Destroy()
end)

minBtn.Activated:Connect(function()
    mainFrame.Visible = not mainFrame.Visible
end)

-- Keybind Processing
keybindBox.FocusLost:Connect(function()
    local input = keybindBox.Text:upper()
    if #input > 0 then
        local char = string.sub(input, 1, 1)
        for _, enumKey in ipairs(Enum.KeyCode:GetEnumItems()) do
            if enumKey.Name == char or enumKey.Name == "Z" and char == "Z" then
                boundKey = enumKey
                keybindBox.Text = char
                antiBatBind.value = enumKey
                saveConfig()
                return
            end
        end
    end
end)

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == boundKey then
        updateButtonState(not isActive)
    end
end)

-- Dragging Logic
local isDragging = false
local dragOffset = Vector2.new()
local startPosition = UDim2.new()

mainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        isDragging = true
        dragOffset = input.Position
        startPosition = mainFrame.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if isDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = Vector2.new(input.Position.X - dragOffset.X, input.Position.Y - dragOffset.Y)
        mainFrame.Position = UDim2.new(startPosition.X.Scale, startPosition.X.Offset + delta.X, startPosition.Y.Scale, startPosition.Y.Offset + delta.Y)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        isDragging = false
    end
end)

-- Inicia desativado
updateButtonState(false)

print("AraDuels loaded - Anti-Bat com menu Anti Anti Desync!")
