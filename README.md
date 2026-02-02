-- LocalScript.lua
-- Colocar em StarterPlayer > StarterPlayerScripts
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local function waitForCharacter()
    local char = player.Character or player.CharacterAdded:Wait()
    local hrp = char:WaitForChild("HumanoidRootPart")
    return char, hrp
end

local char, hrp = waitForCharacter()
local origCFrame = hrp.CFrame

-- Recria referência quando o personagem respawnar
player.CharacterAdded:Connect(function(c)
    char = c
    hrp = char:WaitForChild("HumanoidRootPart")
    origCFrame = hrp.CFrame
end)

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "UpDownGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

-- Frame inicial com botão Ativar
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 180, 0, 70)
mainFrame.Position = UDim2.new(0.5, -90, 0.1, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
mainFrame.AnchorPoint = Vector2.new(0.5, 0)
mainFrame.Parent = screenGui

local ativarBtn = Instance.new("TextButton")
ativarBtn.Name = "Ativar"
ativarBtn.Size = UDim2.new(0, 160, 0, 40)
ativarBtn.Position = UDim2.new(0, 10, 0, 15)
ativarBtn.Text = "Ativar"
ativarBtn.Font = Enum.Font.SourceSansBold
ativarBtn.TextSize = 20
ativarBtn.BackgroundColor3 = Color3.fromRGB(59,153,72)
ativarBtn.TextColor3 = Color3.fromRGB(255,255,255)
ativarBtn.Parent = mainFrame

-- Frame secundário com botões Up, Down, Desativar (inicialmente invisível)
local controlFrame = Instance.new("Frame")
controlFrame.Name = "ControlFrame"
controlFrame.Size = UDim2.new(0, 220, 0, 110)
controlFrame.Position = UDim2.new(0.5, -110, 0.1, 0)
controlFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
controlFrame.AnchorPoint = Vector2.new(0.5, 0)
controlFrame.Visible = false
controlFrame.Parent = screenGui

local upBtn = Instance.new("TextButton")
upBtn.Name = "Up"
upBtn.Size = UDim2.new(0, 190, 0, 30)
upBtn.Position = UDim2.new(0, 15, 0, 10)
upBtn.Text = "Up (subir)"
upBtn.Font = Enum.Font.SourceSans
upBtn.TextSize = 18
upBtn.BackgroundColor3 = Color3.fromRGB(70,130,255)
upBtn.TextColor3 = Color3.fromRGB(255,255,255)
upBtn.Parent = controlFrame

local downBtn = Instance.new("TextButton")
downBtn.Name = "Down"
downBtn.Size = UDim2.new(0, 190, 0, 30)
downBtn.Position = UDim2.new(0, 15, 0, 45)
downBtn.Text = "Down (descer)"
downBtn.Font = Enum.Font.SourceSans
downBtn.TextSize = 18
downBtn.BackgroundColor3 = Color3.fromRGB(255,165,0)
downBtn.TextColor3 = Color3.fromRGB(255,255,255)
downBtn.Parent = controlFrame

local desativarBtn = Instance.new("TextButton")
desativarBtn.Name = "Desativar"
desativarBtn.Size = UDim2.new(0, 190, 0, 30)
desativarBtn.Position = UDim2.new(0, 15, 0, 80)
desativarBtn.Text = "Desativar"
desativarBtn.Font = Enum.Font.SourceSansBold
desativarBtn.TextSize = 18
desativarBtn.BackgroundColor3 = Color3.fromRGB(200,50,50)
desativarBtn.TextColor3 = Color3.fromRGB(255,255,255)
desativarBtn.Parent = controlFrame

-- Estado
local upActive = false
local downActive = false
local accumulatedOffset = 0 -- quanto subimos (+) ou descemos (-) em blocos
local upDummy, downDummy

-- Função para criar um dummy simples (client-side)
local function spawnDummy(name, offset)
    local model = Instance.new("Model")
    model.Name = name
    model.Parent = workspace

    local root = Instance.new("Part")
    root.Name = "HumanoidRootPart"
    root.Size = Vector3.new(2, 2, 1)
    root.Anchored = true
    root.CanCollide = false
    root.Position = hrp.Position + offset
    root.Parent = model

    local head = Instance.new("Part")
    head.Name = "Head"
    head.Size = Vector3.new(1, 1, 1)
    head.Position = root.Position + Vector3.new(0, 1.5, 0)
    head.Anchored = true
    head.Parent = model

    local humanoid = Instance.new("Humanoid")
    humanoid.Parent = model

    -- Billboard com o nome para ficar claro
    local billboard = Instance.new("BillboardGui", head)
    billboard.Size = UDim2.new(0, 100, 0, 30)
    billboard.AlwaysOnTop = true
    local label = Instance.new("TextLabel", billboard)
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.new(1,1,1)
    label.Font = Enum.Font.SourceSansBold
    label.TextScaled = true

    return model
end

local function cleanupDummies()
    if upDummy and upDummy.Parent then upDummy:Destroy() end
    if downDummy and downDummy.Parent then downDummy:Destroy() end
    upDummy = nil
    downDummy = nil
end

-- Loops que fazem o jogador subir/descer 1 bloco repetidamente enquanto o botão estiver ativo
local function startUpLoop()
    if upActive then return end
    upActive = true
    downActive = false
    spawn(function()
        while upActive do
            if not hrp or not hrp.Parent then break end
            -- mover 1 bloco para cima
            hrp.CFrame = hrp.CFrame + Vector3.new(0, 1, 0)
            accumulatedOffset = accumulatedOffset + 1
            wait(0.4)
        end
    end)
end

local function startDownLoop()
    if downActive then return end
    downActive = true
    upActive = false
    spawn(function()
        while downActive do
            if not hrp or not hrp.Parent then break end
            -- mover 1 bloco para baixo
            hrp.CFrame = hrp.CFrame + Vector3.new(0, -1, 0)
            accumulatedOffset = accumulatedOffset - 1
            wait(0.4)
        end
    end)
end

local function deactivateAll()
    upActive = false
    downActive = false

    -- Tweenar de volta à posição original (apenas Y para "descer todos os blocos")
    if hrp and hrp.Parent then
        local targetCFrame = origCFrame
        -- Se quiser manter a posição XZ atual e apenas ajustar Y:
        targetCFrame = CFrame.new(hrp.Position.X, origCFrame.Position.Y, hrp.Position.Z)
        local tween = TweenService:Create(hrp, TweenInfo.new(0.8, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {CFrame = targetCFrame})
        tween:Play()
        tween.Completed:Wait()
    end

    accumulatedOffset = 0
    cleanupDummies()

    -- trocar de GUI: mostrar novamente a inicial
    controlFrame.Visible = false
    mainFrame.Visible = true
end

-- Botões
ativarBtn.MouseButton1Click:Connect(function()
    -- criar dummies e trocar GUI
    cleanupDummies()
    upDummy = spawnDummy("up", Vector3.new(3, 0, 0))
    downDummy = spawnDummy("down", Vector3.new(-3, 0, 0))

    mainFrame.Visible = false
    controlFrame.Visible = true
end)

upBtn.MouseButton1Click:Connect(function()
    startUpLoop()
end)

downBtn.MouseButton1Click:Connect(function()
    startDownLoop()
end)

desativarBtn.MouseButton1Click:Connect(function()
    deactivateAll()
end)

-- Se a GUI for removida/recriada por respawn, recriar (opcional)
playerGui.ChildRemoved:Connect(function(child)
    if child == screenGui then
        -- nada por enquanto
    end
end)
