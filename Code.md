-- UpDownServer.lua
-- Script servidor (server-side) — gerencia estado, cria dummies e move personagens
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

-- Configurações
local MOVE_INTERVAL = 0.35 -- tempo entre cada passo de 1 bloco
local DUMMY_OFFSET = 3 -- distância lateral para spawn dos dummies

-- Eventos no ReplicatedStorage (cria se não existir)
local folder = ReplicatedStorage:FindFirstChild("UpDownEvents")
if not folder then
    folder = Instance.new("Folder")
    folder.Name = "UpDownEvents"
    folder.Parent = ReplicatedStorage
end

local function getOrCreateEvent(name)
    local ev = folder:FindFirstChild(name)
    if not ev then
        ev = Instance.new("RemoteEvent")
        ev.Name = name
        ev.Parent = folder
    end
    return ev
end

local EVENT_ACTIVATE = getOrCreateEvent("UD_Activate")
local EVENT_START_UP = getOrCreateEvent("UD_StartUp")
local EVENT_START_DOWN = getOrCreateEvent("UD_StartDown")
local EVENT_DEACTIVATE = getOrCreateEvent("UD_Deactivate")

-- Estado por jogador
local playersState = {}

local function spawnDummyForPlayer(player, name, offsetVec)
    local success, err = pcall(function()
        local plrChar = player.Character
        if not plrChar or not plrChar:FindFirstChild("HumanoidRootPart") then return end
        local hrp = plrChar.HumanoidRootPart

        local model = Instance.new("Model")
        model.Name = name .. "_" .. player.UserId
        model.Parent = workspace

        local root = Instance.new("Part")
        root.Name = "HumanoidRootPart"
        root.Size = Vector3.new(2,2,1)
        root.Anchored = true
        root.CanCollide = false
        root.Position = hrp.Position + offsetVec
        root.Parent = model

        local head = Instance.new("Part")
        head.Name = "Head"
        head.Size = Vector3.new(1,1,1)
        head.Position = root.Position + Vector3.new(0,1.5,0)
        head.Anchored = true
        head.Parent = model

        local humanoid = Instance.new("Humanoid")
        humanoid.Parent = model

        local billboard = Instance.new("BillboardGui", head)
        billboard.Size = UDim2.new(0, 120, 0, 30)
        billboard.AlwaysOnTop = true
        billboard.StudsOffset = Vector3.new(0, 1.5, 0)
        local label = Instance.new("TextLabel", billboard)
        label.Size = UDim2.new(1, 0, 1, 0)
        label.BackgroundTransparency = 1
        label.Text = name
        label.TextColor3 = Color3.new(1,1,1)
        label.Font = Enum.Font.SourceSansBold
        label.TextScaled = true

        return model
    end)
    if not success then
        warn("Erro ao spawnar dummy:", err)
    end
end

local function cleanupDummies(state)
    if state.upDummy and state.upDummy.Parent then
        state.upDummy:Destroy()
        state.upDummy = nil
    end
    if state.downDummy and state.downDummy.Parent then
        state.downDummy:Destroy()
        state.downDummy = nil
    end
end

local function startMoveLoop(player, state)
    -- garante que apenas uma coroutine de movimento exista por jogador
    if state.movingCoroutine then return end

    state.movingCoroutine = coroutine.create(function()
        while state.upActive or state.downActive do
            local char = player.Character
            if not char or not char.Parent then break end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if not hrp or not humanoid then break end
            if humanoid.Health <= 0 then break end

            local delta = 0
            if state.upActive then delta = 1 end
            if state.downActive then delta = -1 end

            if delta ~= 0 then
                local ok, err = pcall(function()
                    -- mover 1 bloco no eixo Y
                    local newPos = hrp.Position + Vector3.new(0, delta, 0)
                    -- preservar orientação:
                    local look = hrp.CFrame.LookVector
                    hrp.CFrame = CFrame.new(newPos, newPos + look)
                end)
                if not ok then
                    warn("Erro ao mover personagem (server):", err)
                end
            end

            wait(MOVE_INTERVAL)
        end
        state.movingCoroutine = nil
    end)

    coroutine.resume(state.movingCoroutine)
end

local function deactivatePlayer(player, state)
    if not state then return end
    state.upActive = false
    state.downActive = false

    -- Tween de volta à altura original, se existir
    local char = player.Character
    if char and char.Parent and state.originalY then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then
            local targetPos = Vector3.new(hrp.Position.X, state.originalY, hrp.Position.Z)
            local targetCFrame = CFrame.new(targetPos, targetPos + hrp.CFrame.LookVector)
            local ok, err = pcall(function()
                local tween = TweenService:Create(hrp, TweenInfo.new(0.7, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {CFrame = targetCFrame})
                tween:Play()
                tween.Completed:Wait()
            end)
            if not ok then
                -- fallback direto
                pcall(function() hrp.CFrame = targetCFrame end)
            end
        end
    end

    cleanupDummies(state)

    -- limpar flags e coroutine (a coroutine encerrará naturalmente)
    state.movingCoroutine = nil
    state.active = false
end

-- Event handlers
EVENT_ACTIVATE.OnServerEvent:Connect(function(player)
    local char = player.Character
    if not char or not char.Parent then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local state = playersState[player] or {}
    state.active = true
    state.upActive = false
    state.downActive = false
    state.originalY = hrp.Position.Y
    playersState[player] = state

    cleanupDummies(state)
    -- spawn dummies no servidor para que todos vejam
    spawnDummyForPlayer(player, "up", Vector3.new(DUMMY_OFFSET, 0, 0))
    spawnDummyForPlayer(player, "down", Vector3.new(-DUMMY_OFFSET, 0, 0))
    -- Armazenar referência (procura pelo modelo recém-criado)
    -- Usa um pequeno delay para garantir criação
    delay(0.05, function()
        if not player.Character then return end
        for _,obj in ipairs(workspace:GetChildren()) do
            if obj:IsA("Model") and obj.Name:match("^up_%d+$") and obj.Name:find(tostring(player.UserId), 1, true) then
                state.upDummy = obj
            end
            if obj:IsA("Model") and obj.Name:match("^down_%d+$") and obj.Name:find(tostring(player.UserId), 1, true) then
                state.downDummy = obj
            end
        end
    end)
end)

EVENT_START_UP.OnServerEvent:Connect(function(player)
    local state = playersState[player]
    if not state then return end
    state.upActive = true
    state.downActive = false
    startMoveLoop(player, state)
end)

EVENT_START_DOWN.OnServerEvent:Connect(function(player)
    local state = playersState[player]
    if not state then return end
    state.downActive = true
    state.upActive = false
    startMoveLoop(player, state)
end)

EVENT_DEACTIVATE.OnServerEvent:Connect(function(player)
    local state = playersState[player]
    if not state then return end
    deactivatePlayer(player, state)
end)

-- Limpeza quando jogador sai
Players.PlayerRemoving:Connect(function(player)
    local state = playersState[player]
    if state then
        cleanupDummies(state)
        playersState[player] = nil
    end
end)

-- Segurança: se character morrer ou respawnar, resetar estado
Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function(char)
        local state = playersState[player]
        if state and state.active then
            -- garante que depois do respawn possamos definir originalY novamente (apenas quando o personagem estiver pronto)
            local hrp = char:WaitForChild("HumanoidRootPart", 5)
            if hrp then
                state.originalY = hrp.Position.Y
            end
        end
    end)
end)
