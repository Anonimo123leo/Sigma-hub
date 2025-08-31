-- Evitar múltiplas execuções
if getgenv().MyExploitUI then return end
getgenv().MyExploitUI = true

-- Serviços
local Players = game:GetService("Players")
local player = Players.LocalPlayer
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

-- Variáveis principais
local savedCFrame = nil
local tpCooldown = false
local isFlying = false
local flightSpeed = 100 -- Velocidade do voo reduzida

-- DEBUG
local function debugLog(msg)
    print("[MyExploitUI] "..tostring(msg))
end

-- Checagem se personagem existe
local function hasCharacter()
    return player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChildOfClass("Humanoid")
end

-- Função para obter a primeira tool visível
local function getFirstVisibleTool()
    local tools = {}
    
    -- Procura por tools no workspace
    for _, item in ipairs(workspace:GetDescendants()) do
        if item:IsA("Tool") and item:FindFirstChild("Handle") then
            if item.Handle and item.Handle.Position.Magnitude > 0 then
                table.insert(tools, item)
            end
        end
    end
    
    if #tools > 0 then
        debugLog("Tool encontrada: " .. tools[1].Name)
        return tools[1]
    else
        debugLog("Nenhuma tool encontrada")
        return nil
    end
end

-- Função para equipar e ativar a tool
local function equipAndUseTool(tool)
    if not hasCharacter() or not tool then return false end
    
    local character = player.Character
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local hrp = character:FindFirstChild("HumanoidRootPart")
    
    if not humanoid or not hrp then return false end
    
    debugLog("Usando tool: " .. tool.Name)
    
    -- Salva estado original
    local originalCollidable = hrp.CanCollide
    
    -- Torna HRP colidível
    hrp.CanCollide = true
    
    -- Move tool para o personagem (equipa)
    tool.Parent = character
    
    -- Pequeno delay para equipar
    task.wait(0.3)
    
    -- Tenta ativar a tool de várias formas
    pcall(function()
        for i = 1, 5 do
            tool:Activate()
            debugLog("Ativação " .. i)
            task.wait(0.1)
        end
    end)
    
    -- Procura por eventos remotos
    pcall(function()
        for _, descendant in ipairs(tool:GetDescendants()) do
            if descendant:IsA("RemoteEvent") then
                pcall(function() descendant:FireServer() end)
                pcall(function() descendant:FireServer("activate") end)
                pcall(function() descendant:FireServer("fire") end)
                pcall(function() descendant:FireServer(hrp.Position) end)
                debugLog("RemoteEvent acionado: " .. descendant.Name)
            end
        end
    end)
    
    -- Efeito visual simples
    local effect = Instance.new("Part")
    effect.Size = Vector3.new(2, 2, 2)
    effect.Position = hrp.Position
    effect.Anchored = true
    effect.CanCollide = false
    effect.Transparency = 0.8
    effect.Material = Enum.Material.Neon
    effect.Color = Color3.fromRGB(255, 50, 50)
    effect.Parent = workspace
    game:GetService("Debris"):AddItem(effect, 1)
    
    -- Devolve tool e restaura colisão
    task.wait(0.5)
    if tool and tool.Parent == character then
        tool.Parent = workspace
    end
    
    hrp.CanCollide = originalCollidable
    
    debugLog("Uso da tool concluído")
    return true
end

-- Função de voo simples e confiável
local function simpleFlyToLocation(destination)
    if not hasCharacter() or isFlying then return false end
    
    local hrp = player.Character.HumanoidRootPart
    local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
    
    if not hrp or not humanoid then return false end
    
    isFlying = true
    debugLog("Iniciando voo para: " .. tostring(destination.Position))
    
    -- Movimento direto sem tween complexo
    local startTime = tick()
    local startPos = hrp.Position
    local endPos = destination.Position
    local distance = (endPos - startPos).Magnitude
    local duration = distance / flightSpeed
    
    humanoid.AutoRotate = false
    
    -- Loop de movimento suave
    local connection
    connection = RunService.Heartbeat:Connect(function(delta)
        if not isFlying or not hrp or not hrp.Parent then
            connection:Disconnect()
            return
        end
        
        local elapsed = tick() - startTime
        local progress = math.min(elapsed / duration, 1)
        
        if progress < 1 then
            local newPosition = startPos + (endPos - startPos) * progress
            hrp.CFrame = CFrame.new(newPosition) * (destination - destination.Position)
        else
            hrp.CFrame = destination
            connection:Disconnect()
            isFlying = false
            humanoid.AutoRotate = true
            debugLog("Voo concluído!")
        end
    end)
    
    -- Timeout de segurança
    task.delay(duration + 2, function()
        if isFlying then
            connection:Disconnect()
            isFlying = false
            if hrp and hrp.Parent then
                hrp.CFrame = destination
                humanoid.AutoRotate = true
            end
            debugLog("Voo finalizado por timeout")
        end
    end)
    
    return true
end

-------------------------
-- FUNÇÕES DE GUI
-------------------------
local function dragify(frame)
    local dragging, dragInput, dragStart, startPos
    
    local function update(input)
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end
    
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            update(input)
        end
    end)
end

local function createButton(parent, text, yPos)
    local btn = Instance.new("TextButton")
    btn.Parent = parent
    btn.Size = UDim2.new(0.9, 0, 0, 40)
    btn.Position = UDim2.new(0.05, 0, yPos, 0)
    btn.BackgroundColor3 = Color3.fromRGB(100, 180, 255)
    btn.BorderSizePixel = 0
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 18
    btn.AutoButtonColor = true
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
    return btn
end

-- ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MyExploitUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = CoreGui

-- Proteção de GUI
if syn and syn.protect_gui then
    syn.protect_gui(screenGui)
elseif gethui then
    screenGui.Parent = gethui()
elseif get_hidden_ui then
    screenGui.Parent = get_hidden_ui()
end

-- Bolinha
local ball = Instance.new("ImageButton")
ball.Size = UDim2.new(0, 60, 0, 60)
ball.Position = UDim2.new(0.05, 0, 0.3, 0)
ball.Image = "rbxassetid://12657626227"
ball.BackgroundTransparency = 1
ball.AutoButtonColor = false
ball.Parent = screenGui

Instance.new("UICorner", ball).CornerRadius = UDim.new(1, 0)
local stroke = Instance.new("UIStroke", ball)
stroke.Color = Color3.fromRGB(0, 0, 80)
stroke.Thickness = 2
dragify(ball)

-- Frame pequeno
local smallFrame = Instance.new("Frame")
smallFrame.Size = UDim2.new(0, 220, 0, 140)
smallFrame.Position = UDim2.new(0.2, 0, 0.3, 0)
smallFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 60)
smallFrame.BorderSizePixel = 0
smallFrame.Visible = false
smallFrame.Parent = screenGui

Instance.new("UICorner", smallFrame).CornerRadius = UDim.new(0, 10)
dragify(smallFrame)

-- Botões
local saveBtn = createButton(smallFrame, "📍Save location", 0.05)
local tpBtn = createButton(smallFrame, "🚀Instant Steal", 0.55)

-- Toggle GUI
ball.MouseButton1Click:Connect(function()
    smallFrame.Visible = not smallFrame.Visible
end)

-------------------------
-- FUNÇÕES DE JOGO
-------------------------

-- Salvar localização
saveBtn.MouseButton1Click:Connect(function()
    if hasCharacter() then
        savedCFrame = player.Character.HumanoidRootPart.CFrame
        debugLog("Localização salva: "..tostring(savedCFrame.Position))
        saveBtn.Text = "📍Saved!"
        task.wait(1)
        saveBtn.Text = "📍Save location"
    else
        debugLog("Erro: Personagem não encontrado")
    end
end)

-- Função principal simplificada
local function instantSteal()
    if not savedCFrame then 
        debugLog("Erro: Nenhuma posição salva")
        return false 
    end
    
    if not hasCharacter() then 
        debugLog("Erro: Personagem não existe")
        return false 
    end
    
    debugLog("🚀 INICIANDO...")
    
    -- 1. Tenta usar uma tool se existir
    local tool = getFirstVisibleTool()
    if tool then
        debugLog("🔫 USANDO TOOL...")
        equipAndUseTool(tool)
        task.wait(0.5)
    else
        debugLog("⚠️ Nenhuma tool encontrada, pulando etapa")
    end
    
    -- 2. Voo para localização
    debugLog("✈️ VOANDO...")
    simpleFlyToLocation(savedCFrame)
    
    debugLog("✅ CONCLUÍDO!")
    return true
end

-- Função Instant Steal com cooldown
tpBtn.MouseButton1Click:Connect(function()
    if tpCooldown then 
        debugLog("Aguarde o cooldown")
        return 
    end
    
    if not savedCFrame then 
        debugLog("Erro: posição não salva") 
        return 
    end

    tpCooldown = true
    local originalText = tpBtn.Text
    local originalColor = tpBtn.BackgroundColor3
    
    tpBtn.Text = "⏳ Ativando..."
    tpBtn.BackgroundColor3 = Color3.fromRGB(255, 100, 100)
    
    -- Executa em uma thread separada para não travar a UI
    task.spawn(function()
        local success = pcall(function()
            instantSteal()
        end)
        
        if not success then
            debugLog("❌ Erro durante execução")
        end
        
        -- Cooldown
        for i = 3, 1, -1 do
            if tpBtn and tpBtn.Parent then
                tpBtn.Text = "⏳ "..i.."s"
            end
            task.wait(1)
        end
        
        tpCooldown = false
        if tpBtn and tpBtn.Parent then
            tpBtn.Text = originalText
            tpBtn.BackgroundColor3 = originalColor
        end
    end)
end)

-- Fechar GUI com ESC
UserInputService.InputBegan:Connect(function(input, processed)
    if not processed and input.KeyCode == Enum.KeyCode.Escape then
        if smallFrame.Visible then
            smallFrame.Visible = false
        end
    end
end)

-- Inicialização
debugLog("====================================")
debugLog("Interface carregada com sucesso!")
debugLog("Modo: Tool + Voo simplificado")
debugLog("Instruções:")
debugLog("1. Clique em 'Save location' para salvar")
debugLog("2. Clique em 'Instant Steal' para executar")
debugLog("3. ESC para fechar a janela")
debugLog("====================================")
