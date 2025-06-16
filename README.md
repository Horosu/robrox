local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
-- Removido: local PlayerGui = LocalPlayer:WaitForChild("PlayerGui") - Vamos fazer a verificação explícita dentro da função

local ESPs = {}
local IsESPEnabled = false
local IsAimbotEnabled = false

-- --- CONFIGURAÇÕES DO ESP ---
local ESP_SETTINGS = {
    NameTextSize = UDim2.new(0, 80, 0, 20),
    NameStudsOffset = Vector3.new(0, 2.5, 0),
    HighlightFillColor = Color3.fromRGB(0, 255, 255),
    HighlightOutlineColor = Color3.fromRGB(255, 0, 255),
    HighlightFillTransparency = 0.4,
    HighlightOutlineTransparency = 0.2,
    MaxDistanceToShow = math.huge
}
-- ---------------------------

-- --- CONFIGURAÇÕES DO AIMBOT ---
local AIMBOT_SETTINGS = {
    AimLockKey = Enum.KeyCode.MouseButton2,
    MaxAimDistance = 300,
    AimPart = "HumanoidRootPart",
    MouseSensitivityMultiplier = 0.5
}
local IsAimingButtonPressed = false
-- ------------------------------

-- --- FUNÇÕES DO ESP --- (Mantidas como antes, mas controladas pelo IsESPEnabled)
local function CreatePlayerESPVisuals(player)
    if ESPs[player] or player == LocalPlayer then return end
    print("[ESP_VISUALS] Criando visuais para: " .. player.Name)
    local character = player.Character or player.CharacterAdded:Wait()
    if not character then warn("[ESP_VISUALS] Erro: Personagem não encontrado para " .. player.Name) return end
    local humanoidRootPart = character:WaitForChild("HumanoidRootPart", 10)
    if not humanoidRootPart then warn("[ESP_VISUALS] Erro: HumanoidRootPart não encontrado para " .. player.Name) return end

    local billboardGui = Instance.new("BillboardGui")
    billboardGui.Name = "BloxFruitsESP_" .. player.Name
    billboardGui.Size = ESP_SETTINGS.NameTextSize
    billboardGui.StudsOffset = ESP_SETTINGS.NameStudsOffset
    billboardGui.AlwaysOnTop = true
    billboardGui.ExtentsOffset = Vector3.new(0, 0, 0)
    billboardGui.Parent = humanoidRootPart
    billboardGui.Enabled = IsESPEnabled
    
    local nameDistanceLabel = Instance.new("TextLabel")
    nameDistanceLabel.Size = UDim2.new(1, 0, 1, 0)
    nameDistanceLabel.BackgroundTransparency = 1
    nameDistanceLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    nameDistanceLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    nameDistanceLabel.TextStrokeTransparency = 0.5
    nameDistanceLabel.Font = Enum.Font.SourceSansBold
    nameDistanceLabel.TextScaled = true
    nameDistanceLabel.Text = player.Name .. " (??m)"
    nameDistanceLabel.TextXAlignment = Enum.TextXAlignment.Center
    nameDistanceLabel.TextYAlignment = Enum.TextYAlignment.Center
    nameDistanceLabel.Parent = billboardGui

    local highlight = Instance.new("Highlight")
    highlight.FillColor = ESP_SETTINGS.HighlightFillColor
    highlight.OutlineColor = ESP_SETTINGS.HighlightOutlineColor
    highlight.FillTransparency = ESP_SETTINGS.HighlightFillTransparency
    highlight.OutlineTransparency = ESP_SETTINGS.OutlineTransparency
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = character
    highlight.Enabled = IsESPEnabled

    ESPs[player] = {
        BillboardGui = billboardGui,
        Highlight = highlight,
        NameDistanceLabel = nameDistanceLabel,
        CharacterAddedConnection = player.CharacterAdded:Connect(function(newCharacter)
            print("[ESP_VISUALS] Personagem de " .. player.Name .. " recarregado. Reanexando ESP visuais.")
            local newHumanoidRootPart = newCharacter:WaitForChild("HumanoidRootPart", 10)
            if newHumanoidRootPart then billboardGui.Parent = newHumanoidRootPart end
            highlight.Parent = newCharacter
            billboardGui.Enabled = IsESPEnabled
            highlight.Enabled = IsESPEnabled
        end),
        AncestryChangedConnection = character.AncestryChanged:Connect(function(obj, parent)
            if obj == character and highlight.Parent ~= character then
                 warn("[ESP_VISUALS] AVISO: Highlight de " .. player.Name .. " foi movido, tentando recolocar.")
                 highlight.Parent = character
                 highlight.Enabled = IsESPEnabled
            end
        end)
    }
    print("[ESP_VISUALS] ESP criado para " .. player.Name)
end

local function RemovePlayerESPVisuals(player)
    if ESPs[player] then
        print("[ESP_VISUALS] Removendo visuais ESP para " .. player.Name)
        if ESPs[player].BillboardGui then ESPs[player].BillboardGui:Destroy() end
        if ESPs[player].Highlight then ESPs[player].Highlight:Destroy() end
        if ESPs[player].CharacterAddedConnection then ESPs[player].CharacterAddedConnection:Disconnect() end
        if ESPs[player].AncestryChangedConnection then ESPs[player].AncestryChangedConnection:Disconnect() end
        ESPs[player] = nil
        print("[ESP_VISUALS] Visuais ESP removidos e dados limpos para " .. player.Name)
    end
end

local function ToggleESP(enable)
    print("[TOGGLE_ESP] Função ToggleESP chamada com estado: " .. tostring(enable))
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            if enable then
                local esp = ESPs[player] or CreatePlayerESPVisuals(player)
                if esp and esp.BillboardGui and esp.Highlight then
                    esp.BillboardGui.Enabled = true
                    esp.Highlight.Enabled = true
                end
            else
                RemovePlayerESPVisuals(player)
            end
        end
    end
    IsESPEnabled = enable
end

-- --- FUNÇÕES DO AIMBOT ---
local function GetClosestTarget()
    local closestTarget = nil
    local shortestDistance = AIMBOT_SETTINGS.MaxAimDistance
    local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localRoot then return nil end

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local targetCharacter = player.Character
            local targetHumanoid = targetCharacter and targetCharacter:FindFirstChildOfClass("Humanoid")
            local targetAimPart = targetCharacter and targetCharacter:FindFirstChild(AIMBOT_SETTINGS.AimPart)

            if targetHumanoid and targetHumanoid.Health > 0 and targetAimPart then
                local distance = (localRoot.Position - targetAimPart.Position).Magnitude
                if distance < shortestDistance then
                    closestTarget = targetAimPart
                    shortestDistance = distance
                end
            end
        end
    end
    return closestTarget
end

local function MoveMouseToTarget(targetPart)
    local camera = Workspace.CurrentCamera
    if not camera then return end

    local screenPoint, onScreen = camera:WorldToScreenPoint(targetPart.Position)

    if onScreen then
        local mouse = UserInputService:GetMouseLocation()
        local currentX, currentY = mouse.X, mouse.Y

        local targetX = screenPoint.X
        local targetY = screenPoint.Y

        local deltaX = targetX - currentX
        local deltaY = targetY - currentY

        local moveX = deltaX * AIMBOT_SETTINGS.MouseSensitivityMultiplier
        local moveY = deltaY * AIMBOT_SETTINGS.MouseSensitivityMultiplier
        
        if math.abs(moveX) < 1 and math.abs(deltaX) > 0 then moveX = math.sign(deltaX) end
        if math.abs(moveY) < 1 and math.abs(deltaY) > 0 then moveY = math.sign(deltaY) end

        UserInputService:SetMouseLocation(Vector2.new(currentX + moveX, currentY + moveY))
        -- print(string.format("[AIMBOT_MOUSE] Movendo mouse para (%d, %d). Delta: (%.2f, %.2f)", targetX, targetY, moveX, moveY)) -- Removido para evitar spam no console
    end
end


-- --- LOOP PRINCIPAL (RenderStepped) ---

RunService.RenderStepped:Connect(function()
    local localCharacter = LocalPlayer.Character
    local localRoot = localCharacter and localCharacter:FindFirstChild("HumanoidRootPart")
    
    if not localRoot then return end

    -- Lógica do ESP (agora depende de IsESPEnabled)
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local espData = ESPs[player]
            local targetCharacter = player.Character
            local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")

            if targetRoot then
                if not espData then
                    CreatePlayerESPVisuals(player)
                    espData = ESPs[player]
                end

                if espData and espData.BillboardGui and espData.Highlight and espData.NameDistanceLabel then
                    -- Atualiza estado de Enabled com base em IsESPEnabled E distância
                    local shouldBeEnabled = IsESPEnabled and ((localRoot.Position - targetRoot.Position).Magnitude <= ESP_SETTINGS.MaxDistanceToShow)
                    
                    if espData.BillboardGui.Enabled ~= shouldBeEnabled then
                        espData.BillboardGui.Enabled = shouldBeEnabled
                    end
                    if espData.Highlight.Enabled ~= shouldBeEnabled then
                        espData.Highlight.Enabled = shouldBeEnabled
                    end

                    local distance = math.floor((localRoot.Position - targetRoot.Position).Magnitude)
                    espData.NameDistanceLabel.Text = player.Name .. " (" .. distance .. "m)"
                end
            else
                if espData then RemovePlayerESPVisuals(player) end
            end
        end
    end

    -- Lógica do Aimbot (agora depende de IsAimbotEnabled E IsAimingButtonPressed)
    if IsAimbotEnabled and IsAimingButtonPressed then
        local target = GetClosestTarget()
        if target then
            MoveMouseToTarget(target)
        else
            -- print("[AIMBOT] Nenhum alvo encontrado dentro da distância máxima.") -- Removido para evitar spam no console
        end
    end
end)

-- --- EVENTOS DE INPUT (para o Aimbot - Tecla de mira) ---

UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
    -- Verifica se o Aimbot está ativado pela GUI e se a tecla de mira foi pressionada
    if input.KeyCode == AIMBOT_SETTINGS.AimLockKey and not gameProcessedEvent and IsAimbotEnabled then
        IsAimingButtonPressed = true
        print("[AIMBOT] Botão de mira pressionado. Aimbot de mouse ativo.")
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessedEvent)
    -- Verifica se o Aimbot está ativado pela GUI e se a tecla de mira foi solta
    if input.KeyCode == AIMBOT_SETTINGS.AimLockKey and not gameProcessedEvent and IsAimbotEnabled then
        IsAimingButtonPressed = false
        print("[AIMBOT] Botão de mira solto. Aimbot de mouse inativo.")
    end
end)


-- --- EVENTOS DE JOGADOR ---
Players.PlayerAdded:Connect(function(player)
    print("[EVENT] Jogador " .. player.Name .. " entrou.")
    -- A criação dos visuais ESP será gerenciada pelo loop RenderStepped
end)

Players.PlayerRemoving:Connect(RemovePlayerESPVisuals)

-- --- INICIALIZAÇÃO ---

-- Tenta criar a HUD após um pequeno delay, caso o PlayerGui precise de tempo para carregar
print("[INICIALIZAÇÃO] Tentando criar o painel de controle da GUI...")
-- Adicione um pequeno atraso para dar tempo ao PlayerGui
task.wait(1) 

local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
if playerGui then
    print("[INICIALIZAÇÃO] PlayerGui encontrado. Prosseguindo com a criação da HUD.")
    CreateControlPanel()
else
    warn("[INICIALIZAÇÃO] ERRO: PlayerGui não encontrado após 1 segundo. A HUD pode não aparecer.")
end


-- Ativa o ESP para jogadores existentes assim que o script carrega (se IsESPEnabled for true)
-- O RenderStepped e PlayerAdded cuidam da criação contínua, mas isso garante que os já presentes sejam processados
-- O estado inicial IsESPEnabled é 'false', então não ativará por padrão.
-- A ativação será feita via GUI.

print("Blox Fruits ESP e Aimbot carregados!")
print("Use o painel na tela para ativar/desativar as funções.")
print("Para Aimbot, ative no painel e pressione o scroll do mouse no jogo.")
print("*** Verifique o console do seu executor para mensagens de depuração ***")
