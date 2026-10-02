--[[
    Auto Farm Script - Rayfield UI
    Autor: Dev Experiente
    Descrição: Auto farm com detecção dinâmica de mobs, teleporte e ataque automático
--]]

--// Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local VirtualUser = game:GetService("VirtualUser")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

--// Carregar Rayfield UI
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

--// Configurações Globais
local Config = {
    AutoFarm = false,
    SelectedMob = nil,
    TeleportDistance = 5,     -- Distância para teleportar do mob
    AttackRange = 15,          -- Alcance de ataque
    AttackCooldown = 0.15,     -- Cooldown entre ataques
    TeleportInterval = 0.1,    -- Intervalo entre teleportes
    Whitelist = {},            -- Mobs na whitelist (ignorar)
    Blacklist = {},            -- Mobs na blacklist (ignorar)
}

--// Estado interno
local State = {
    CurrentTarget = nil,
    LastAttack = 0,
    LastTeleport = 0,
    Connections = {},
    MobCache = {},
}

--=============================================================================
-- FUNÇÕES UTILITÁRIAS
--=============================================================================

--// Verifica se uma instância é um mob válido (vivo)
local function isAliveMob(instance)
    if not instance or not instance.Parent then return false end
    if not instance:IsA("Model") then return false end
    
    -- Verifica se tem Humanoid
    local humanoid = instance:FindFirstChildOfClass("Humanoid")
    if not humanoid then return false end
    
    -- Verifica se está vivo
    if humanoid.Health <= 0 then return false end
    
    -- Verifica se não é o próprio player ou outro jogador
    if instance == LocalPlayer.Character then return false end
    if Players:GetPlayerFromCharacter(instance) then return false end
    
    -- Verifica se está no workspace (renderizado)
    if not instance:IsDescendantOf(Workspace) then return false end
    
    return true, humanoid
end

--// Verifica se o mob está visível na tela (renderizado)
local function isMobRendered(mob)
    if not mob or not mob.PrimaryPart then return false end
    
    local rootPart = mob.PrimaryPart
    local screenPoint, onScreen = Camera:WorldToViewportPoint(rootPart.Position)
    
    -- Considera renderizado se está na tela ou próximo (dentro de um raio)
    if onScreen then return true end
    
    -- Fallback: verifica distância
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local dist = (char.HumanoidRootPart.Position - rootPart.Position).Magnitude
        return dist <= 500 -- Considera "renderizado" se estiver a menos de 500 studs
    end
    
    return false
end

--// Obtém o nome "amigável" do mob
local function getMobName(mob)
    local name = mob.Name
    
    -- Remove sufixos comuns de duplicatas (ex: "Zombie (1)")
    name = name:gsub("%s*%(%d+%)$", "")
    
    -- Pega o nome do Humanoid se disponível
    local humanoid = mob:FindFirstChildOfClass("Humanoid")
    if humanoid and humanoid.DisplayName and humanoid.DisplayName ~= "" then
        -- Usa o nome do modelo (mais consistente para agrupamento)
    end
    
    return name
end

--// Escaneia o workspace e retorna tipos únicos de mobs
local function scanMobs()
    local mobTypes = {}   -- { [nome] = { exemplos = {}, count = 0 } }
    local allMobs = {}
    
    for _, descendant in ipairs(Workspace:GetDescendants()) do
        local alive, humanoid = isAliveMob(descendant)
        if alive then
            local name = getMobName(descendant)
            
            if not mobTypes[name] then
                mobTypes[name] = { name = name, count = 0, examples = {} }
            end
            
            mobTypes[name].count = mobTypes[name].count + 1
            table.insert(mobTypes[name].examples, descendant)
            table.insert(allMobs, descendant)
        end
    end
    
    -- Converte para lista ordenada por quantidade (decrescente)
    local list = {}
    for _, data in pairs(mobTypes) do
        table.insert(list, data)
    end
    table.sort(list, function(a, b) return a.count > b.count end)
    
    State.MobCache = mobTypes
    return list, allMobs
end

--// Formata a lista de mobs para o dropdown
local function formatMobList(mobList)
    local formatted = {}
    for _, data in ipairs(mobList) do
        table.insert(formatted, string.format("%s (%d)", data.name, data.count))
    end
    return formatted
end

--// Extrai o nome puro do mob a partir da string formatada
local function parseMobName(formatted)
    return formatted:match("^(.+)%s+%(%d+%)$") or formatted
end

--=============================================================================
-- MECÂNICA DE TELEPORTE E ATAQUE
--=============================================================================

--// Teleporta o jogador para uma posição
local function teleportTo(position)
    local char = LocalPlayer.Character
    if not char then return false end
    
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return false end
    
    -- Teleporte via CFrame (mais confiável)
    root.CFrame = CFrame.new(position)
    return true
end

--// Executa ataque no mob
local function attackMob(mob, humanoid)
    if not mob or not mob.PrimaryPart then return end
    if not humanoid or humanoid.Health <= 0 then return end
    
    local now = tick()
    if now - State.LastAttack < Config.AttackCooldown then return end
    State.LastAttack = now
    
    -- Método 1: Simular clique (funciona na maioria dos jogos)
    local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildWhichIsA("Tool")
    if tool then
        tool:Activate()
    end
    
    -- Método 2: Ataque via Humanoid (para alguns jogos)
    local myChar = LocalPlayer.Character
    if myChar then
        local myHumanoid = myChar:FindFirstChildOfClass("Humanoid")
        if myHumanoid then
            -- Alguns jogos usam isso
            pcall(function()
                myHumanoid:MoveTo(mob.PrimaryPart.Position)
            end)
        end
    end
    
    -- Método 3: VirtualUser (simula input real)
    pcall(function()
        VirtualUser:Button1Down(Vector2.new(0, 0))
        task.wait(0.05)
        VirtualUser:Button1Up(Vector2.new(0, 0))
    end)
end

--// Loop principal do Auto Farm
local function autoFarmLoop()
    while Config.AutoFarm do
        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then
            task.wait(0.5)
            continue
        end
        
        local myRoot = char.HumanoidRootPart
        local myHumanoid = char:FindFirstChildOfClass("Humanoid")
        
        if not myHumanoid or myHumanoid.Health <= 0 then
            task.wait(0.5)
            continue
        end
        
        -- Obtém mobs do tipo selecionado
        local targetName = Config.SelectedMob
        if not targetName then
            task.wait(0.2)
            continue
        end
        
        local mobData = State.MobCache[targetName]
        if not mobData then
            -- Reescaneia se não encontrar
            scanMobs()
            mobData = State.MobCache[targetName]
        end
        
        if not mobData or #mobData.examples == 0 then
            task.wait(0.5)
            continue
        end
        
        -- Encontra o mob mais próximo e vivo
        local closestMob, closestDist, closestHumanoid = nil, math.huge, nil
        
        for _, mob in ipairs(mobData.examples) do
            if not mob.Parent then continue end
            
            local alive, humanoid = isAliveMob(mob)
            if not alive then continue end
            
            if mob.PrimaryPart then
                local dist = (myRoot.Position - mob.PrimaryPart.Position).Magnitude
                if dist < closestDist then
                    closestDist = dist
                    closestMob = mob
                    closestHumanoid = humanoid
                end
            end
        end
        
        if not closestMob then
            -- Nenhum mob vivo deste tipo, reescaneia
            scanMobs()
            task.wait(0.3)
            continue
        end
        
        State.CurrentTarget = closestMob
        
        -- Teleporta para perto do mob
        local mobPos = closestMob.PrimaryPart.Position
        local myPos = myRoot.Position
        local direction = (myPos - mobPos).Unit
        
        -- Posição de destino (atrás do mob, na distância configurada)
        local targetPos = mobPos + (direction * Config.TeleportDistance)
        -- Mantém a altura do jogador
        targetPos = Vector3.new(targetPos.X, mobPos.Y + 3, targetPos.Z)
        
        -- Teleporta
        if tick() - State.LastTeleport >= Config.TeleportInterval then
            teleportTo(targetPos)
            State.LastTeleport = tick()
        end
        
        -- Ataca
        attackMob(closestMob, closestHumanoid)
        
        task.wait(0.05)
    end
end

--// Para o auto farm
local function stopAutoFarm()
    Config.AutoFarm = false
    State.CurrentTarget = nil
    
    -- Para o movimento
    local char = LocalPlayer.Character
    if char then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid:MoveTo(char.HumanoidRootPart.Position)
        end
    end
end

--=============================================================================
-- INTERFACE (RAYFIELD)
--=============================================================================

--// Cria a janela
local Window = Rayfield:CreateWindow({
    Name = "Auto Farm Pro",
    LoadingTitle = "Carregando Auto Farm...",
    LoadingSubtitle = "by Dev Experiente",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "AutoFarmPro",
        FileName = "Config"
    },
    Discord = {
        Enabled = false,
    },
    KeySystem = false,
})

--// Aba principal
local FarmTab = Window:CreateTab("Auto Farm", 4483362458) -- ícone de espada

--// Seção de status
local StatusSection = FarmTab:CreateSection("Status")

local StatusLabel = FarmTab:CreateLabel("Mobs detectados: 0", 4483362458)

--// Seção de seleção
local SelectionSection = FarmTab:CreateSection("Seleção de Alvo")

--// Dropdown de mobs (será atualizado dinamicamente)
local mobDropdown
local function refreshDropdown()
    local mobList = scanMobs()
    local formatted = formatMobList(mobList)
    
    if #formatted == 0 then
        formatted = { "Nenhum mob encontrado" }
    end
    
    -- Atualiza o label de status
    local total = 0
    for _, data in ipairs(mobList) do
        total = total + data.count
    end
    StatusLabel:Set("Mobs detectados: " .. total .. " | Tipos: " .. #mobList)
    
    -- Recria o dropdown (Rayfield não tem update nativo de opções)
    if mobDropdown then
        pcall(function() mobDropdown:Destroy() end)
    end
    
    mobDropdown = FarmTab:CreateDropdown({
        Name = "Tipo de Mob",
        Options = formatted,
        CurrentOption = { formatted[1] },
        MultipleOptions = false,
        Flag = "SelectedMob",
        Callback = function(option)
            local selected
            if type(option) == "table" then
                selected = option[1]
            else
                selected = option
            end
            
            if selected and selected ~= "Nenhum mob encontrado" then
                Config.SelectedMob = parseMobName(selected)
            else
                Config.SelectedMob = nil
            end
        end,
    })
end

--// Botão para atualizar lista
FarmTab:CreateButton({
    Name = "🔄 Atualizar Lista de Mobs",
    Callback = function()
        refreshDropdown()
        Rayfield:Notify({
            Title = "Lista Atualizada",
            Content = "Os mobs foram reescaneados.",
            Duration = 3,
            Image = 4483362458,
        })
    end,
})

--// Seção de configurações
local SettingsSection = FarmTab:CreateSection("Configurações")

FarmTab:CreateSlider({
    Name = "Distância de Teleporte",
    Range = {1, 30},
    Increment = 1,
    Suffix = "studs",
    CurrentValue = Config.TeleportDistance,
    Flag = "TeleportDistance",
    Callback = function(value)
        Config.TeleportDistance = value
    end,
})

FarmTab:CreateSlider({
    Name = "Alcance de Ataque",
    Range = {5, 50},
    Increment = 1,
    Suffix = "studs",
    CurrentValue = Config.AttackRange,
    Flag = "AttackRange",
    Callback = function(value)
        Config.AttackRange = value
    end,
})

FarmTab:CreateSlider({
    Name = "Cooldown de Ataque",
    Range = {0.05, 1},
    Increment = 0.05,
    Suffix = "s",
    CurrentValue = Config.AttackCooldown,
    Flag = "AttackCooldown",
    Callback = function(value)
        Config.AttackCooldown = value
    end,
})

--// Toggle principal
FarmTab:CreateToggle({
    Name = "⚔️ Ativar Auto Farm",
    CurrentValue = false,
    Flag = "AutoFarmToggle",
    Callback = function(value)
        if value then
            if not Config.SelectedMob then
                Rayfield:Notify({
                    Title = "Aviso",
                    Content = "Selecione um tipo de mob primeiro!",
                    Duration = 4,
                    Image = 4483362458,
                })
                return
            end
            
            Config.AutoFarm = true
            task.spawn(autoFarmLoop)
            
            Rayfield:Notify({
                Title = "Auto Farm Ativado",
                Content = "Atacando: " .. Config.SelectedMob,
                Duration = 3,
                Image = 4483362458,
            })
        else
            stopAutoFarm()
            Rayfield:Notify({
                Title = "Auto Farm Desativado",
                Content = "O farm foi interrompido.",
                Duration = 3,
                Image = 4483362458,
            })
        end
    end,
})

--// Seção de utilidades
local UtilsSection = FarmTab:CreateSection("Utilidades")

FarmTab:CreateButton({
    Name = "📊 Rescanear Workspace",
    Callback = function()
        refreshDropdown()
    end,
})

FarmTab:CreateButton({
    Name = "🛑 Parar Tudo",
    Callback = function()
        stopAutoFarm()
        -- Desativa o toggle visualmente
        if Rayfield.Flags and Rayfield.Flags.AutoFarmToggle then
            Rayfield.Flags.AutoFarmToggle:Set(false)
        end
    end,
})

--// Aba de informações
local InfoTab = Window:CreateTab("Info", 4483362458)

InfoTab:CreateParagraph({
    Title = "Como Usar",
    Content = "1. Clique em 'Atualizar Lista de Mobs' para escanear o mapa.\n" ..
              "2. Selecione o tipo de mob no dropdown.\n" ..
              "3. Ajuste as configurações conforme necessário.\n" ..
              "4. Ative o toggle 'Ativar Auto Farm'.\n\n" ..
              "O script irá teleportar e atacar automaticamente os mobs selecionados.",
})

InfoTab:CreateParagraph({
    Title = "Aviso",
    Content = "Use com responsabilidade. Alguns jogos possuem anti-cheat que podem detectar teleportes rápidos.",
})

--// Auto-scan inicial
task.spawn(function()
    task.wait(1)
    refreshDropdown()
end)

--// Auto-rescan periódico (a cada 10 segundos)
task.spawn(function()
    while true do
        task.wait(10)
        if not Config.AutoFarm then
            refreshDropdown()
        end
    end
end)

--// Notificação de carregamento
Rayfield:Notify({
    Title = "Auto Farm Pro",
    Content = "Script carregado com sucesso!",
    Duration = 5,
    Image = 4483362458,
})

print("[AutoFarm] Script carregado com sucesso!")
