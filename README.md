local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

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

-- ========== FUNÇÃO PARA PEGAR O JOGADOR MAIS PRÓXIMO ==========
local function getClosestPlayer()
    local closest = nil
    local closestDist = math.huge
    local char = LocalPlayer.Character
    if not char then return nil end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local targetChar = player.Character
            if targetChar then
                local targetHrp = targetChar:FindFirstChild("HumanoidRootPart")
                if targetHrp then
                    local dist = (hrp.Position - targetHrp.Position).Magnitude
                    if dist < closestDist then
                        closestDist = dist
                        closest = player
                    end
                end
            end
        end
    end
    return closest
end

-- ========== VERIFICA SE SETHIDDENPROPERTY EXISTE ==========
local hasHiddenProperty = type(sethiddenproperty) == "function" or type(syn and syn.sethiddenproperty) == "function"

print("sethiddenproperty disponível:", hasHiddenProperty)

-- ----------------------------------------------------------------
--  ANTI BAT - SÓ TROCA DE FÍSICA (SEM TP, SEM VELOCIDADE)
-- ----------------------------------------------------------------
local antiBatStatus, antiBatSwitchBall, antiBatRow, antiBatRowStroke
local antiBatKeyBtn, antiBatKeyBtnStroke

local function toggleAntiBat()
    State.antiBatActive = not State.antiBatActive
    
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
            
            -- ========== TROCA DE FÍSICA (SUA HRP COM A DO INIMIGO) ==========
            local target = getClosestPlayer()
            if target and target.Character then
                local tr = target.Character:FindFirstChild("HumanoidRootPart")
                if tr then
                    -- syn.sethiddenproperty: TROCA A FÍSICA
                    if hasHiddenProperty then
                        if type(sethiddenproperty) == "function" then
                            sethiddenproperty(tr, "PhysicsRepRootPart", hrp)
                        elseif syn and type(syn.sethiddenproperty) == "function" then
                            syn.sethiddenproperty(tr, "PhysicsRepRootPart", hrp)
                        end
                    end
                end
            end
            
            -- ========== SEM VELOCIDADE, SEM TP ==========
            -- Você fica parado, só troca a física
            
            -- ========== CÂMERA LIVRE ==========
            local cam = Workspace.CurrentCamera
            if cam and cam.CameraType ~= Enum.CameraType.Scriptable then
                cam.CameraType = Enum.CameraType.Custom
            end
        end)
    else
        if State.antiBatThread then
            State.antiBatThread:Disconnect()
            State.antiBatThread = nil
        end
    end
end

-- ----------------------------------------------------------------
--  FUNÇÃO PARA CRIAR GUI
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

if parentGui:FindFirstChild("LarpAntiAntiDesyncGui") then
    parentGui.LarpAntiAntiDesyncGui:Destroy()
end

local screenGui = createInstance("ScreenGui", {
    Name = "LarpAntiAntiDesyncGui",
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, parentGui)

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

createInstance("ImageLabel", {
    Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1,
    Image = "rbxassetid://98596557474777",
    ScaleType = Enum.ScaleType.Crop,
    ZIndex = 2,
}, mainFrame)

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
    Text = "O",
    TextColor3 = Color3.fromRGB(255, 255, 255),
    Font = Enum.Font.FredokaOne,
    TextSize = 14,
    ClearTextOnFocus = false,
    ZIndex = 3,
}, mainFrame)
addCorner(keybindBox, 10)

-- ----------------------------------------------------------------
--  LOGICA DOS BOTOES
-- ----------------------------------------------------------------
local isActive = false
local boundKey = Enum.KeyCode.O

local function updateButtonState(state)
    isActive = state
    if state then
        toggleButton.Text = "DEACTIVATE"
        toggleButton.BackgroundColor3 = Color3.fromRGB(200, 30, 30)
        toggleAntiBat()
    else
        toggleButton.Text = "ACTIVATE"
        toggleButton.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
        if State.antiBatActive then
            toggleAntiBat()
        end
    end
end

toggleButton.Activated:Connect(function()
    updateButtonState(not isActive)
end)

closeBtn.Activated:Connect(function()
    updateButtonState(false)
    screenGui:Destroy()
end)

minBtn.Activated:Connect(function()
    mainFrame.Visible = not mainFrame.Visible
end)

keybindBox.FocusLost:Connect(function()
    local input = keybindBox.Text:upper()
    if #input > 0 then
        local char = string.sub(input, 1, 1)
        for _, enumKey in ipairs(Enum.KeyCode:GetEnumItems()) do
            if enumKey.Name == char or enumKey.Name == "O" and char == "O" then
                boundKey = enumKey
                keybindBox.Text = char
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

-- Dragging
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

updateButtonState(false)

print("AraDuels loaded – SÓ TROCA DE FÍSICA! SEM TP, SEM VELOCIDADE!")
