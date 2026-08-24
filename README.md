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
--  ANTI BAT - SETHIDDENPROPERTY NO INIMIGO + VELOCIDADE (SEM TP)
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
        TweenService:Create(antiBatRow, TweenInfo.new(0.2),
