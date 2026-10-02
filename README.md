-- Dungeons disponíveis em Anime Eternal (divididas por lobby)
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

-- Combina tudo para o dropdown
local DungeonList = {}
for _, d in ipairs(DungeonLobby1) do table.insert(DungeonList, d) end
for _, d in ipairs(DungeonLobby2) do table.insert(DungeonList, d) end

-- Mapeamento de nomes internos para busca no Workspace
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
}
