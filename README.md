-- ===== CONFIGURAÇÕES =====
local DISTANCIA_MAXIMA = 5          -- raio para retornar ao walkpoint (studs)
local INTERVALO_VERIFICACAO = 0.5   -- tempo entre verificações (segundos)
local RAIO_INTERACAO = 19           -- raio para detectar interações (studs)
local INTERVALO_INTERACAO = 1.0     -- tempo entre tentativas de interação

-- ===== VARIÁVEIS =====
local walkpoint = nil
local autoReturnAtivo = true
local autoInteragirAtivo = false
local simularMovimentoAtivo = false

local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()

-- Serviço para simular atividade (anti-AFK)
local VirtualUser = game:GetService("VirtualUser")

-- ===== CRIAR GUI =====
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = player.PlayerGui

local function criarBotao(texto, posY, cor, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 180, 0, 40)
    btn.Position = UDim2.new(0, 10, 0, posY)
    btn.Text = texto
    btn.BackgroundColor3 = cor
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Parent = frame
    btn.MouseButton1Click:Connect(callback)
    return btn
end

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 200, 0, 250) -- aumentei para caber mais botões
frame.Position = UDim2.new(0, 10, 0, 10)
frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
frame.BackgroundTransparency = 0.2
frame.Parent = screenGui

local btnWalk = criarBotao("Marcar Walkpoint", 10, Color3.fromRGB(0,120,255), function()
    if character and character:FindFirstChild("HumanoidRootPart") then
        walkpoint = character.HumanoidRootPart.Position
        btnWalk.Text = "Walkpoint marcado!"
        task.wait(1)
        btnWalk.Text = "Marcar Walkpoint"
    end
end)

local btnReturn = criarBotao("Auto-Retorno: ON", 55, Color3.fromRGB(0,200,0), function()
    autoReturnAtivo = not autoReturnAtivo
    btnReturn.Text = autoReturnAtivo and "Auto-Retorno: ON" or "Auto-Retorno: OFF"
    btnReturn.BackgroundColor3 = autoReturnAtivo and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
end)

local btnInteragir = criarBotao("Auto-Interagir: OFF", 100, Color3.fromRGB(200,0,0), function()
    autoInteragirAtivo = not autoInteragirAtivo
    btnInteragir.Text = autoInteragirAtivo and "Auto-Interagir: ON" or "Auto-Interagir: OFF"
    btnInteragir.BackgroundColor3 = autoInteragirAtivo and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
end)

local btnMovimento = criarBotao("Simular Movimento: OFF", 145, Color3.fromRGB(200,0,0), function()
    simularMovimentoAtivo = not simularMovimentoAtivo
    btnMovimento.Text = simularMovimentoAtivo and "Simular Movimento: ON" or "Simular Movimento: OFF"
    btnMovimento.BackgroundColor3 = simularMovimentoAtivo and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
end)

-- Botão para interagir manualmente com "E" (útil para testes)
local btnManualE = criarBotao("Pressionar E (manual)", 190, Color3.fromRGB(255,165,0), function()
    -- Simula pressionar a tecla E no teclado (funciona para ProximityPrompt)
    local VirtualInput = game:GetService("VirtualInputManager")
    VirtualInput:SendKeyEvent(true, "E", false, game)
    task.wait(0.1)
    VirtualInput:SendKeyEvent(false, "E", false, game)
end)

-- ===== FUNÇÕES =====

-- Atualizar referência do personagem
local function onCharacterAdded(newChar)
    character = newChar
end
player.CharacterAdded:Connect(onCharacterAdded)

-- Função para interagir automaticamente com ProximityPrompt
local function interagirAuto()
    if not autoInteragirAtivo then return end
    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local pos = root.Position
    -- Procurar todos os ProximityPrompt no mundo (ou apenas em partes próximas?)
    -- Podemos usar workspace:GetDescendants() e filtrar por ProximityPrompt
    local prompts = workspace:GetDescendants()
    for _, prompt in ipairs(prompts) do
        if prompt:IsA("ProximityPrompt") then
            -- Verificar distância do prompt até o personagem
            local promptPos = prompt.Parent and prompt.Parent:FindFirstChild("Position") and prompt.Parent.Position
            if not promptPos then
                -- Se o prompt não tiver posição direta, tentamos pegar a posição do objeto pai
                local parent = prompt.Parent
                if parent and parent:IsA("BasePart") then
                    promptPos = parent.Position
                else
                    -- Se não for uma parte, ignoramos
                    continue
                end
            end
            local dist = (pos - promptPos).Magnitude
            if dist <= RAIO_INTERACAO then
                -- Ativar o prompt (simula o pressionamento de E)
                -- Verificar se o prompt está habilitado e se o jogador está apto
                if prompt.Enabled and prompt:IsEnabled() then
                    -- Método para acionar: InputHoldBegin e InputHoldEnd
                    prompt:InputHoldBegin()
                    task.wait(0.1)
                    prompt:InputHoldEnd()
                    -- Para evitar spam, podemos sair do loop após uma interação
                    break
                end
            end
        end
    end
end

-- ===== LOOPS PRINCIPAIS =====

-- Loop de retorno ao walkpoint e interação automática
spawn(function()
    while true do
        task.wait(INTERVALO_VERIFICACAO)

        -- 1. Retorno automático
        if autoReturnAtivo and walkpoint then
            local char = player.Character
            if char then
                local root = char:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (root.Position - walkpoint).Magnitude
                    if dist > DISTANCIA_MAXIMA then
                        root.CFrame = CFrame.new(walkpoint)
                        local humanoid = char:FindFirstChildOfClass("Humanoid")
                        if humanoid then
                            humanoid.PlatformStand = true
                            task.wait(0.1)
                            humanoid.PlatformStand = false
                        end
                    end
                end
            end
        end

        -- 2. Interação automática (executa a cada INTERVALO_VERIFICACAO, mas podemos controlar separadamente)
        -- Para não sobrecarregar, vou chamar a função de interação a cada INTERVALO_INTERACAO
        -- Vou usar um contador ou timer separado:
    end
end)

-- Loop separado para interação com intervalo próprio
spawn(function()
    while true do
        task.wait(INTERVALO_INTERACAO)
        if autoInteragirAtivo then
            interagirAuto()
        end
    end
end)

-- Loop de simulação de movimento (anti-AFK)
spawn(function()
    while true do
        task.wait(60) -- a cada 60 segundos (ajuste conforme necessidade)
        if simularMovimentoAtivo then
            -- Simular atividade usando VirtualUser (método comum para evitar AFK)
            pcall(function()
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new())
                -- também pode simular movimento de câmera ou teclas
            end)
        end
    end
end)

-- Opcional: também podemos simular movimento pequeno a cada vez que o personagem estiver parado,
-- mas isso pode atrapalhar o walkpoint, então deixamos apenas o VirtualUser.