--[[
    Auto Farm Script - Orion UI (Compatível com Solara)
    Autor: Dev Experiente
    Jogo: Anime Eternal
    Descrição: Auto farm + Gamemodes (Dungeons Lobby 1, Lobby 2 e Timeline 2)
--]]

--// Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local VirtualUser = game:GetService("VirtualUser")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

--// Carregar Orion UI
local OrionLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/Qanuir/orion-ui/refs/heads/main/source.lua"))()

--// Configurações Globais (Auto Farm)
local Config = {
    AutoFarm = false,
    SelectedMob = nil,
    TeleportDistance = 5,
    AttackRange = 15,
    AttackCooldown = 0.15,
    TeleportInterval = 0.1,
}

--// Estado Interno
local State = {
    CurrentTarget = nil,
    LastAttack = 0,
    LastTeleport = 0,
    MobCache = {},
}

--=============================================================================
-- FUNÇÕES UTILITÁRIAS (MOBS)
--=============================================================================

local function isAliveMob(instance)
    if not instance or not instance.Parent then return false end
    if not instance:IsA("Model") then return false end
    local humanoid = instance:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    if instance == LocalPlayer.Character then return false end
    if Players:GetPlayerFromCharacter(instance) then return false end
    if not instance:IsDescendantOf(Workspace) then return false end
    return true, humanoid
end

local function getMobName(mob)
    local name = mob.Name
    name = name:gsub("%s*%(%d+%)$", "")
    return name
end

local function scanMobs()
    local mobTypes = {}
    for _, descendant in ipairs(Workspace:GetDescendants()) do
        local alive = isAliveMob(descendant)
        if alive then
            local name = getMobName(descendant)
            if not mobTypes[name] then
                mobTypes[name] = { name = name, count = 0, examples = {} }
            end
            mobTypes[name].count = mobTypes[name].count + 1
            table.insert(mobTypes[name].examples, descendant)
        end
    end
    local list = {}
    for _, data in pairs(mobTypes) do
        table.insert(list, data)
    end
    table.sort(list, function(a, b) return a.count > b.count end)
    State.MobCache = mobTypes
    return list
end

local function formatMobList(mobList)
    local formatted = {}
    for _, data in ipairs(mobList) do
        table.insert(formatted, string.format("%s (%d)", data.name, data.count))
    end
    return formatted
end

local function parseMobName(formatted)
    return formatted:match("^(.+)%s+%(%d+%)$") or formatted
end

--=============================================================================
-- MECÂNICA DE TELEPORTE E ATAQUE
--=============================================================================

local function teleportTo(position)
    local char = LocalPlayer.Character
    if not char then return false end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return false end
    root.CFrame = CFrame.new(position)
    return true
end

local function attackMob(mob, humanoid)
    if not mob or not mob.PrimaryPart then return end
    if not humanoid or humanoid.Health <= 0 then return end
    local now = tick()
    if now - State.LastAttack < Config.AttackCooldown then return end
    State.LastAttack = now

    local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildWhichIsA("Tool")
    if tool then tool:Activate() end

    pcall(function()
        VirtualUser:Button1Down(Vector2.new(0, 0))
        task.wait(0.05)
        VirtualUser:Button1Up(Vector2.new(0, 0))
    end)
end

--=============================================================================
-- AUTO FARM LOOP
--=============================================================================

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

        local targetName = Config.SelectedMob
        if not targetName then task.wait(0.2) continue end

        local mobData = State.MobCache[targetName]
        if not mobData then
            scanMobs()
            mobData = State.MobCache[targetName]
        end
        if not mobData or #mobData.examples == 0 then
            task.wait(0.5)
            continue
        end

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
            scanMobs()
            task.wait(0.3)
            continue
        end

        State.CurrentTarget = closestMob
        local mobPos = closestMob.PrimaryPart.Position
        local myPos = myRoot.Position
        local direction = (myPos - mobPos).Unit
        local targetPos = mobPos + (direction * Config.TeleportDistance)
        targetPos = Vector3.new(targetPos.X, mobPos.Y + 3, targetPos.Z)

        if tick() - State.LastTeleport >= Config.TeleportInterval then
            teleportTo(targetPos)
            State.LastTeleport = tick()
        end
        attackMob(closestMob, closestHumanoid)
        task.wait(0.05)
    end
end

local function stopAutoFarm()
    Config.AutoFarm = false
    State.CurrentTarget = nil
    local char = LocalPlayer.Character
    if char then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid:MoveTo(char.HumanoidRootPart.Position) end
    end
end

--=============================================================================
-- GAMEMODES (DUNGEONS) - ANIME ETERNAL
--=============================================================================

-- Lista de dungeons por lobby
local DungeonLobby1 = {
    "Restaurant Raid",
    "Cursed Raid",
    "Sin Raid",
    "Gleam Raid",
    "Progression Raid",
    "Leaf Raid",
    "Green Planet Raid",
}

local DungeonLobby2 = {
    "Maze 1",
    "Maze 2",
    "Adventurer",
    "Torment",
    "Hollow Raid",
}

local DungeonTimeline2 = {
    "Timeline 2 Raid",
    "Timeline 2 Maze",
    "Timeline 2 Boss",
}

-- Lista combinada para o dropdown
local DungeonList = {}
for _, d in ipairs(DungeonLobby1) do table.insert(DungeonList, "[Lobby 1] " .. d) end
for _, d in ipairs(DungeonLobby2) do table.insert(DungeonList, "[Lobby 2] " .. d) end
for _, d in ipairs(DungeonTimeline2) do table.insert(DungeonList, "[Timeline 2] " .. d) end

-- Mapeamento de termos de busca no Workspace
local DungeonTeleportNames = {
    ["Restaurant Raid"] = {"Restaurant", "RestaurantRaid"},
    ["Cursed Raid"] = {"Cursed", "CursedRaid"},
    ["Sin Raid"] = {"Sin", "SinRaid"},
    ["Gleam Raid"] = {"Gleam", "GleamRaid"},
    ["Progression Raid"] = {"Progression", "ProgressionRaid"},
    ["Leaf Raid"] = {"Leaf", "LeafRaid"},
    ["Green Planet Raid"] = {"GreenPlanet", "GreenPlanetRaid"},
    ["Maze 1"] = {"Maze1", "Maze_1", "MazeLevel1"},
    ["Maze 2"] = {"Maze2", "Maze_2", "MazeLevel2"},
    ["Adventurer"] = {"Adventurer", "DungeonAdventurer"},
    ["Torment"] = {"Torment", "DungeonTorment"},
    ["Hollow Raid"] = {"Hollow", "HollowRaid"},
    ["Timeline 2 Raid"] = {"Timeline2", "Timeline2Raid"},
    ["Timeline 2 Maze"] = {"Timeline2Maze", "T2Maze"},
    ["Timeline 2 Boss"] = {"Timeline2Boss", "T2Boss"},
}

-- Estado do Auto Dungeon
local DungeonState = {
    Active = false,
    SelectedDungeon = nil,
    AutoKillMobs = true,
    LastAttempt = 0,
    EntryDelay = 2,
    InDungeon = false,
}

-- Remove o prefixo [Lobby X] do nome
local function cleanDungeonName(name)
    return name:gsub("^%[.-%]%s*", "")
end

-- Teleporta para o lobby correto
local function goToCorrectLobby(dungeonName)
    local isLobby2 = false
    local isTimeline2 = false

    for _, d in ipairs(DungeonLobby2) do
        if d == dungeonName then isLobby2 = true break end
    end
    for _, d in ipairs(DungeonTimeline2) do
        if d == dungeonName then isTimeline2 = true break end
    end

    local keyword
    if isTimeline2 then
        keyword = "timeline2"
    elseif isLobby2 then
        keyword = "dungeonlobby2"
    else
        keyword = "dungeonlobby"
    end

    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return false end

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Name:lower():find(keyword) then
            char.HumanoidRootPart.CFrame = obj.CFrame + Vector3.new(0, 5, 0)
            task.wait(0.5)
            return true
        end
    end
    return false
end

-- Teleporta para a dungeon específica
local function teleportToDungeon(dungeonName)
    goToCorrectLobby(dungeonName)
    task.wait(0.5)

    local searchTerms = DungeonTeleportNames[dungeonName]
    if not searchTerms then return false end

    local found = nil
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if not obj:IsA("BasePart") then continue end
        local name = obj.Name:lower()
        for _, term in ipairs(searchTerms) do
            if name:find(term:lower()) then
                found = obj
                break
            end
        end
        if found then break end
    end

    if found then
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            char.HumanoidRootPart.CFrame = found.CFrame + Vector3.new(0, 5, 0)
            return true
        end
    end
    return false
end

-- Detecta se o jogador está dentro de uma dungeon
local function isInDungeon()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return false end
    local pos = char.HumanoidRootPart.Position

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("Model") or obj:IsA("Folder") then
            local name = obj.Name:lower()
            if name:find("dungeon") or name:find("raid") or name:find("wave") or name:find("maze") then
                local part = obj:FindFirstChildWhichIsA("BasePart", true)
                if part and (pos - part.Position).Magnitude < 500 then
                    return true
                end
            end
        end
    end
    return false
end

-- Loop do Auto Dungeon
local function autoDungeonLoop()
    while DungeonState.Active do
        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then
            task.wait(0.5)
            continue
        end

        local now = tick()
        if now - DungeonState.LastAttempt >= DungeonState.EntryDelay then
            DungeonState.LastAttempt = now

            if not isInDungeon() then
                if DungeonState.SelectedDungeon then
                    teleportToDungeon(DungeonState.SelectedDungeon)
                    task.wait(1)

                    -- Procura prompts/portas próximas para entrar
                    for _, obj in ipairs(Workspace:GetDescendants()) do
                        if obj:IsA("ProximityPrompt") then
                            local parent = obj.Parent
                            local name = parent and parent.Name:lower() or ""
                            if name:find("door") or name:find("entrance") or name:find("portal") or name:find("dungeon") then
                                local part = obj.Parent
                                if part and part:IsA("BasePart") then
                                    local dist = (char.HumanoidRootPart.Position - part.Position).Magnitude
                                    if dist < 20 then
                                        pcall(function() fireproximityprompt(obj) end)
                                    end
                                end
                            end
                        elseif obj:IsA("ClickDetector") then
                            local part = obj.Parent
                            if part and part:IsA("BasePart") then
                                local dist = (char.HumanoidRootPart.Position - part.Position).Magnitude
                                if dist < 20 then
                                    pcall(function() fireclickdetector(obj) end)
                                end
                            end
                        end
                    end
                end
            else
                DungeonState.InDungeon = true
            end
        end

        -- Auto kill dentro da dungeon
        if DungeonState.InDungeon and DungeonState.AutoKillMobs then
            local myRoot = char.HumanoidRootPart
            local myHumanoid = char:FindFirstChildOfClass("Humanoid")

            if myHumanoid and myHumanoid.Health > 0 then
                local closestMob, closestDist, closestHumanoid = nil, math.huge, nil

                for _, descendant in ipairs(Workspace:GetDescendants()) do
                    if not descendant:IsA("Model") then continue end
                    local humanoid = descendant:FindFirstChildOfClass("Humanoid")
                    if not humanoid or humanoid.Health <= 0 then continue end
                    if descendant == char then continue end
                    if Players:GetPlayerFromCharacter(descendant) then continue end

                    if descendant.PrimaryPart then
                        local dist = (myRoot.Position - descendant.PrimaryPart.Position).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closestMob = descendant
                            closestHumanoid = humanoid
                        end
                    end
                end

                if closestMob and closestMob.PrimaryPart then
                    local mobPos = closestMob.PrimaryPart.Position
                    local direction = (myRoot.Position - mobPos).Unit
                    local targetPos = mobPos + (direction * Config.TeleportDistance)
                    targetPos = Vector3.new(targetPos.X, mobPos.Y + 3, targetPos.Z)
                    myRoot.CFrame = CFrame.new(targetPos)
                    attackMob(closestMob, closestHumanoid)
                end
            end
        end

        task.wait(0.1)
    end
end

--=============================================================================
-- INTERFACE (ORION UI)
--=============================================================================

local Window = OrionLib:MakeWindow({
    Name = "Auto Farm Pro - Anime Eternal",
    HidePremium = false,
    SaveConfig = false,
    ConfigFolder = "AutoFarmPro"
})

--======================== ABA 1: AUTO FARM ============================
local FarmTab = Window:MakeTab({
    Name = "Auto Farm",
    Icon = "rbxassetid://4483362458",
    PremiumOnly = false
})

FarmTab:AddSection({ Name = "Status" })
local StatusLabel = FarmTab:AddLabel("Mobs detectados: 0")

FarmTab:AddSection({ Name = "Seleção de Alvo" })

local mobDropdown
local function refreshDropdown()
    local mobList = scanMobs()
    local formatted = formatMobList(mobList)
    if #formatted == 0 then formatted = { "Nenhum mob encontrado" } end

    local total = 0
    for _, data in ipairs(mobList) do total = total + data.count end
    StatusLabel:Set("Mobs detectados: " .. total .. " | Tipos: " .. #mobList)

    if mobDropdown then pcall(function() mobDropdown:Destroy() end) end
    mobDropdown = FarmTab:AddDropdown({
        Name = "Tipo de Mob",
        Default = formatted[1],
        Options = formatted,
        Flag = "SelectedMob",
        Callback = function(option)
            local selected = type(option) == "table" and option[1] or option
            if selected and selected ~= "Nenhum mob encontrado" then
                Config.SelectedMob = parseMobName(selected)
            else
                Config.SelectedMob = nil
            end
        end
    })
end

FarmTab:AddButton({
    Name = "🔄 Atualizar Lista de Mobs",
    Callback = function()
        refreshDropdown()
        OrionLib:MakeNotification({ Name = "Lista Atualizada", Content = "Os mobs foram reescaneados.", Time = 3 })
    end
})

FarmTab:AddSection({ Name = "Configurações" })

FarmTab:AddSlider({
    Name = "Distância de Teleporte",
    Min = 1, Max = 30, Default = Config.TeleportDistance,
    Color = Color3.fromRGB(255,255,255), Increment = 1, ValueName = "studs",
    Callback = function(value) Config.TeleportDistance = value end
})

FarmTab:AddSlider({
    Name = "Cooldown de Ataque",
    Min = 0.05, Max = 1, Default = Config.AttackCooldown,
    Color = Color3.fromRGB(255,255,255), Increment = 0.05, ValueName = "s",
    Callback = function(value) Config.AttackCooldown = value end
})

FarmTab:AddToggle({
    Name = "⚔️ Ativar Auto Farm",
    Default = false, Flag = "AutoFarmToggle",
    Callback = function(value)
        if value then
            if not Config.SelectedMob then
                OrionLib:MakeNotification({ Name = "Aviso", Content = "Selecione um tipo de mob primeiro!", Time = 4 })
                return
            end
            Config.AutoFarm = true
            task.spawn(autoFarmLoop)
            OrionLib:MakeNotification({ Name = "Auto Farm Ativado", Content = "Atacando: " .. Config.SelectedMob, Time = 3 })
        else
            stopAutoFarm()
            OrionLib:MakeNotification({ Name = "Auto Farm Desativado", Content = "O farm foi interrompido.", Time = 3 })
        end
    end
})

FarmTab:AddSection({ Name = "Utilidades" })
FarmTab:AddButton({ Name = "📊 Rescanear Workspace", Callback = function() refreshDropdown() end })
FarmTab:AddButton({
    Name = "🛑 Parar Tudo",
    Callback = function()
        stopAutoFarm()
        if OrionLib.Flags and OrionLib.Flags.AutoFarmToggle then
            OrionLib.Flags.AutoFarmToggle:Set(false)
        end
    end
})

--======================== ABA 2: GAMEMODES ============================
local GamemodeTab = Window:MakeTab({
    Name = "Gamemodes",
    Icon = "rbxassetid://4483362458",
    PremiumOnly = false
})

GamemodeTab:AddSection({ Name = "Auto Dungeon" })

GamemodeTab:AddDropdown({
    Name = "Selecionar Dungeon",
    Default = DungeonList[1],
    Options = DungeonList,
    Flag = "SelectedDungeon",
    Callback = function(option)
        local selected = type(option) == "table" and option[1] or option
        DungeonState.SelectedDungeon = cleanDungeonName(selected)
    end
})

GamemodeTab:AddToggle({
    Name = "⚔️ Auto Enter Dungeon",
    Default = false, Flag = "AutoDungeonToggle",
    Callback = function(value)
        if value then
            if not DungeonState.SelectedDungeon then
                OrionLib:MakeNotification({ Name = "Aviso", Content = "Selecione uma dungeon primeiro!", Time = 3 })
                return
            end
            DungeonState.Active = true
            task.spawn(autoDungeonLoop)
            OrionLib:MakeNotification({ Name = "Auto Dungeon Ativado", Content = "Entrando em: " .. DungeonState.SelectedDungeon, Time = 3 })
        else
            DungeonState.Active = false
            DungeonState.InDungeon = false
            OrionLib:MakeNotification({ Name = "Auto Dungeon Desativado", Content = "O modo automático foi interrompido.", Time = 3 })
        end
    end
})

GamemodeTab:AddToggle({
    Name = "Matar Mobs Automaticamente",
    Default = true, Flag = "AutoKillToggle",
    Callback = function(value) DungeonState.AutoKillMobs = value end
})

GamemodeTab:AddSlider({
    Name = "Delay entre Tentativas",
    Min = 0.5, Max = 10, Default = 2,
    Color = Color3.fromRGB(255,255,255), Increment = 0.5, ValueName = "s",
    Callback = function(value) DungeonState.EntryDelay = value end
})

GamemodeTab:AddSection({ Name = "Utilidades" })

GamemodeTab:AddButton({
    Name = "📍 Teleportar para Dungeon",
    Callback = function()
        if DungeonState.SelectedDungeon then
            if teleportToDungeon(DungeonState.SelectedDungeon) then
                OrionLib:MakeNotification({ Name = "Teleporte", Content = "Indo para: " .. DungeonState.SelectedDungeon, Time = 2 })
            else
                OrionLib:MakeNotification({ Name = "Erro", Content = "Dungeon não encontrada no mapa atual.", Time = 3 })
            end
        else
            OrionLib:MakeNotification({ Name = "Aviso", Content = "Selecione uma dungeon primeiro.", Time = 3 })
        end
    end
})

GamemodeTab:AddButton({
    Name = "🚪 Tentar Entrar (Prompt/Porta)",
    Callback = function()
        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then return end
        local myPos = char.HumanoidRootPart.Position
        local found = false

        for _, obj in ipairs(Workspace:GetDescendants()) do
            if obj:IsA("ProximityPrompt") and obj.Parent and obj.Parent:IsA("BasePart") then
                local dist = (myPos - obj.Parent.Position).Magnitude
                if dist < 25 then
                    pcall(function() fireproximityprompt(obj) end)
                    found = true
                end
            elseif obj:IsA("ClickDetector") and obj.Parent and obj.Parent:IsA("BasePart") then
                local dist = (myPos - obj.Parent.Position).Magnitude
                if dist < 25 then
                    pcall(function() fireclickdetector(obj) end)
                    found = true
                end
            end
        end

        if found then
            OrionLib:MakeNotification({ Name = "Entrada", Content = "Tentando entrar na dungeon...", Time = 2 })
        else
            OrionLib:MakeNotification({ Name = "Erro", Content = "Nenhuma porta/prompt próximo.", Time = 3 })
        end
    end
})

GamemodeTab:AddSection({ Name = "Info" })

GamemodeTab:AddParagraph("Dungeons Disponíveis",
    "Lobby 1: Restaurant, Cursed, Sin, Gleam, Progression, Leaf, Green Planet.\n" ..
    "Lobby 2: Maze 1, Maze 2, Adventurer, Torment, Hollow Raid.\n" ..
    "Timeline 2: Raid, Maze, Boss."
)

GamemodeTab:AddParagraph("Aviso",
    "As dungeons requerem chaves no inventário. O script NÃO gera chaves. " ..
    "Para Maze 2, é preciso limpar o Maze 1 primeiro."
)

--======================== ABA 3: INFO ============================
local InfoTab = Window:MakeTab({
    Name = "Info",
    Icon = "rbxassetid://4483362458",
    PremiumOnly = false
})

InfoTab:AddParagraph("Como Usar",
    "1. Clique em 'Atualizar Lista de Mobs' para escanear o mapa.\n" ..
    "2. Selecione o tipo de mob no dropdown.\n" ..
    "3. Ajuste as configurações conforme necessário.\n" ..
    "4. Ative o toggle 'Ativar Auto Farm'."
)

InfoTab:AddParagraph("Gamemodes",
    "1. Abra a aba Gamemodes.\n" ..
    "2. Selecione a dungeon desejada.\n" ..
    "3. Ative 'Auto Enter Dungeon'.\n" ..
    "4. Marque 'Matar Mobs Automaticamente' para auto-kill dentro."
)

InfoTab:AddParagraph("Aviso",
    "Use com responsabilidade. Alguns jogos possuem anti-cheat que podem detectar teleportes rápidos."
)

--// Auto-scan inicial e periódico
task.spawn(function()
    task.wait(1)
    refreshDropdown()
end)

task.spawn(function()
    while true do
        task.wait(10)
        if not Config.AutoFarm then refreshDropdown() end
    end
end)

OrionLib:MakeNotification({
    Name = "Auto Farm Pro",
    Content = "Script carregado com sucesso!",
    Time = 5
})

print("[AutoFarm] Script carregado com sucesso!")
