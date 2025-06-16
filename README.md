local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

local ESPs = {} -- Tabela para armazenar os ESPs (BillboardGui, Highlight, etc.) para cada jogador

-- --- CONFIGURAÇÕES DO ESP ---
-- Ajuste estas configurações para o que você preferir!
local ESP_SETTINGS = {
    NameTextSize = UDim2.new(0, 80, 0, 20), -- Tamanho do BillboardGui (largura, altura) para o nome e distância
    NameStudsOffset = Vector3.new(0, 2.5, 0), -- Posição vertical do ESP acima da cabeça do jogador
    HighlightFillColor = Color3.fromRGB(0, 255, 255), -- Cor do preenchimento do glow: Ciano brilhante
    HighlightOutlineColor = Color3.fromRGB(255, 0, 255), -- Cor do contorno do glow: Rosa Neon
    HighlightFillTransparency = 0.4, -- Transparência do preenchimento (0 = opaco, 1 = invisível)
    HighlightOutlineTransparency = 0.2, -- Transparência do contorno
    MaxDistanceToShow = math.huge -- *** ALTERADO: Distância máxima ilimitada para exibir o ESP. ***
}
-- ---------------------------

-- Função para criar o ESP visual para um jogador (BillboardGui e Highlight)
local function CreatePlayerESPVisuals(player)
    -- Se já existe um ESP para este jogador ou se é o jogador local, não faz nada
    if ESPs[player] or player == LocalPlayer then return end

    print("[ESP_VISUALS] Tentando criar visuais ESP para: " .. player.Name)

    -- Espera que o personagem do jogador exista e que tenha o HumanoidRootPart
    local character = player.Character or player.CharacterAdded:Wait()
    if not character then
        warn("[ESP_VISUALS] ERRO: Personagem de " .. player.Name .. " não encontrado após CharacterAdded:Wait().")
        return
    end
    local humanoidRootPart = character:WaitForChild("HumanoidRootPart", 10) -- Espera um pouco mais para garantir

    -- Se não encontrar o HumanoidRootPart, não podemos anexar o ESP
    if not humanoidRootPart then
        warn("[ESP_VISUALS] ERRO: HumanoidRootPart de " .. player.Name .. " não encontrado após 10s. ESP não criado para este jogador.")
        return
    end

    -- Cria o BillboardGui (o "painel" com o nome e distância)
    local billboardGui = Instance.new("BillboardGui")
    billboardGui.Name = "BloxFruitsESP_" .. player.Name
    billboardGui.Size = ESP_SETTINGS.NameTextSize
    billboardGui.StudsOffset = ESP_SETTINGS.NameStudsOffset
    billboardGui.AlwaysOnTop = true -- Sempre visível, mesmo atrás de objetos
    billboardGui.ExtentsOffset = Vector3.new(0, 0, 0) -- Sem deslocamento adicional
    billboardGui.Parent = humanoidRootPart -- Anexa ao HumanoidRootPart para seguir o jogador
    billboardGui.Enabled = true -- Garante que o BillboardGui esteja ativo por padrão
    print("[ESP_VISUALS] BillboardGui criado e parentado para " .. player.Name .. ". Enabled: " .. tostring(billboardGui.Enabled))

    -- Cria o TextLabel para exibir o nome do jogador e a distância
    local nameDistanceLabel = Instance.new("TextLabel")
    nameDistanceLabel.Size = UDim2.new(1, 0, 1, 0) -- Preenche todo o BillboardGui
    nameDistanceLabel.BackgroundTransparency = 1 -- Fundo transparente
    nameDistanceLabel.TextColor3 = Color3.fromRGB(255, 255, 255) -- Cor do texto: Branco
    nameDistanceLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0) -- Contorno do texto: Preto
    nameDistanceLabel.TextStrokeTransparency = 0.5
    nameDistanceLabel.Font = Enum.Font.SourceSansBold
    nameDistanceLabel.TextScaled = true -- Redimensiona o texto para caber
    nameDistanceLabel.Text = player.Name .. " (??m)" -- Texto inicial (nome + distância desconhecida)
    nameDistanceLabel.TextXAlignment = Enum.TextXAlignment.Center -- Centraliza o texto horizontalmente
    nameDistanceLabel.TextYAlignment = Enum.TextYAlignment.Center -- Centraliza o texto verticalmente
    nameDistanceLabel.Parent = billboardGui
    print("[ESP_VISUALS] TextLabel criado para " .. player.Name)

    -- Cria o Highlight (o efeito de glow/brilho)
    local highlight = Instance.new("Highlight")
    highlight.FillColor = ESP_SETTINGS.HighlightFillColor
    highlight.OutlineColor = ESP_SETTINGS.HighlightOutlineColor
    highlight.FillTransparency = ESP_SETTINGS.HighlightFillTransparency
    highlight.OutlineTransparency = ESP_SETTINGS.OutlineTransparency
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop -- Tenta manter o glow visível através de objetos
    highlight.Parent = character -- Anexa ao Character para aplicar o glow a todo o modelo
    highlight.Enabled = true -- Garante que o Highlight esteja ativo por padrão
    print("[ESP_VISUALS] Highlight criado e parentado para " .. player.Name .. ". Enabled: " .. tostring(highlight.Enabled))

    -- Armazena as referências e as conexões para gerenciar o ESP
    ESPs[player] = {
        BillboardGui = billboardGui,
        Highlight = highlight,
        NameDistanceLabel = nameDistanceLabel, -- Armazena a referência para atualizar o texto
        -- Conecta para reanexar o ESP se o personagem do jogador for recarregado (morrer/renascer)
        CharacterAddedConnection = player.CharacterAdded:Connect(function(newCharacter)
            print("[ESP_VISUALS] Personagem de " .. player.Name .. " recarregado. Reanexando ESP visuals.")
            local newHumanoidRootPart = newCharacter:WaitForChild("HumanoidRootPart", 10)
            if newHumanoidRootPart then
                billboardGui.Parent = newHumanoidRootPart -- Move o BillboardGui para o novo HumanoidRootPart
            else
                warn("[ESP_VISUALS] AVISO: Não foi possível encontrar HumanoidRootPart para o novo personagem de " .. player.Name)
            end
            highlight.Parent = newCharacter -- Move o Highlight para o novo Character
            -- Garante que o Enabled esteja true após recarregamento
            billboardGui.Enabled = true
            highlight.Enabled = true
        end),
        -- Conexão para o evento "AncestryChanged" para garantir que o highlight esteja sempre no Character
        -- Isso ajuda se o highlight for "removido" temporariamente por scripts do jogo
        AncestryChangedConnection = character.AncestryChanged:Connect(function(obj, parent)
            if obj == character and highlight.Parent ~= character then
                 warn("[ESP_VISUALS] AVISO: Highlight de " .. player.Name .. " foi movido do Character, tentando recolocar.")
                 highlight.Parent = character
                 highlight.Enabled = true -- Garante que seja ativado
            end
        end)
    }
    print("[ESP_VISUALS] ESP criado para " .. player.Name)
end

-- Função para remover o ESP de um jogador
local function RemovePlayerESPVisuals(player)
    if ESPs[player] then
        print("[ESP_VISUALS] Removendo ESP para " .. player.Name)
        -- Destrói os objetos da GUI
        if ESPs[player].BillboardGui then ESPs[player].BillboardGui:Destroy() end
        if ESPs[player].Highlight then ESPs[player].Highlight:Destroy() end

        -- Desconecta todas as conexões para evitar vazamentos de memória
        if ESPs[player].CharacterAddedConnection then ESPs[player].CharacterAddedConnection:Disconnect() end
        if ESPs[player].AncestryChangedConnection then ESPs[player].AncestryChangedConnection:Disconnect() end

        ESPs[player] = nil -- Remove a entrada da tabela
        print("[ESP_VISUALS] ESP removido para " .. player.Name)
    end
end

-- --- LOOP DE ATUALIZAÇÃO (DISTÂNCIA E VISIBILIDADE) ---
RunService.RenderStepped:Connect(function()
    local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localRoot then return end -- Se não temos nosso próprio rootPart, não podemos calcular distâncias

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local espData = ESPs[player]
            local targetCharacter = player.Character
            local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")

            if targetRoot then
                if not espData then -- Se não temos ESPData para este jogador, mas ele tem HumanoidRootPart, tente criar.
                    CreatePlayerESPVisuals(player) -- Tentará criar o ESP para este jogador
                    espData = ESPs[player] -- Tente obter a referência recém-criada
                end

                -- Se o espData ainda é nil (falhou na criação) ou os objetos não existem, pule
                if not espData or not espData.BillboardGui or not espData.Highlight or not espData.NameDistanceLabel then
                    continue
                end

                local distance = math.floor((localRoot.Position - targetRoot.Position).Magnitude) -- Distância arredondada para inteiro
                espData.NameDistanceLabel.Text = player.Name .. " (" .. distance .. "m)"

                -- A visibilidade agora é sempre TRUE, pois MaxDistanceToShow é math.huge
                -- (Mas a lógica de enable/disable ainda é útil se você quiser reverter MaxDistanceToShow)
                if not espData.BillboardGui.Enabled then espData.BillboardGui.Enabled = true end
                if not espData.Highlight.Enabled then espData.Highlight.Enabled = true end
                
            else
                -- Se o personagem ou rootPart do alvo sumiu, e ainda temos o ESP no cache, remova-o.
                if espData then
                    RemovePlayerESPVisuals(player)
                end
            end
        end
    end
end)

-- --- EVENTOS DE JOGADOR ---

-- Conecta-se ao evento PlayerAdded para criar ESPs para novos jogadores
Players.PlayerAdded:Connect(function(player)
    print("[EVENT] Jogador " .. player.Name .. " entrou. Criando ESP.")
    CreatePlayerESPVisuals(player)
end)

-- Conecta-se ao evento PlayerRemoving para remover ESPs de jogadores que saem
Players.PlayerRemoving:Connect(RemovePlayerESPVisuals)

-- --- INICIALIZAÇÃO AUTOMÁTICA ---

-- Itera sobre os jogadores já existentes no servidor e cria o ESP para eles
print("[INICIALIZAÇÃO] Ativando ESP automaticamente para todos os jogadores existentes...")
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then -- Ignora o jogador local
        CreatePlayerESPVisuals(player)
    end
end

print("Blox Fruits ESP ativado (Versão Direta - Alcance Ilimitado)! Verifique a tela e o console.")
print("Se o ESP não aparecer, o problema está fora do script (ex: anti-cheat do jogo, executor).")
