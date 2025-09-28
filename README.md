# Scriptsbloxfrut
Bloxfruit
roblox-elephant-brainrot-system/
├─ README.md
├─ LICENSE
├─ .gitignore
├─ server/
│  ├─ ElephantSpawnerServer.lua
│  ├─ DuplicateServer.lua
│  └─ KillAllServer_example.lua        -- opcional, exemplo do KillAll seguro (do pacote anterior)
├─ client/
│  ├─ Local_ElephantNotify.lua
│  ├─ Local_DuplicateButton.lua
│  └─ Local_KillAllButton.lua         -- opcional
├─ modules/
│  ├─ InventoryModule.lua
│  └─ WeaponSet_LastAttacker.lua
└─ assets/
   ├─ README_Templates.md             -- instruções para criar templates e spawn points no StudioREADME.mdElephantSpawnerServer.luaDuplicateServer.luaKillAllServer_example.luaLocal_ElephantNotify.luaLocal_DuplicateButton.luaLocal_KillAllButton.luaInventoryModule.luaWeaponSet_LastAttacker.luaREADME_Templates.md# Roblox Elephant & Brainrot System

Este repositório contém um sistema para Roblox Studio que:
- Spawna aleatoriamente elefantes (Strawberry, Garama, Madung) a cada intervalo (2 horas por padrão).
- Ao morrer, o elefante dá um item `Brainrot` ao jogador que o derrotou (ou jogador mais próximo).
- Sistema de duplicação de `Brainrot` (servidor-side) com custo, cooldown e retorno ao cliente.
- Exemplo de como marcar armas para registrar o `LastAttacker` no humanoid (necessário para creditar kills corretamente).
- Scripts cliente para notificação e UI.

> **IMPORTANTE**: Use este código **somente** em jogos que você controla. Não use para trapacear em jogos de terceiros.

## Estrutura
Consulte a raiz do repositório para ver os arquivos e pastas.

## Instalação rápida (Roblox Studio)
1. Abra seu jogo no Roblox Studio.
2. Em `ReplicatedStorage` crie:
   - Folder `ElephantTemplates` (coloque lá os modelos Strawberry/Garama/Madung).
   - RemoteEvents: `RequestDuplicateBrainrot`, `DuplicateResult`, `ElephantSpawnedEvent`.
3. Em `Workspace` crie Folder `ElephantSpawns` com Parts posicionados para spawn.
4. Em `ServerScriptService` coloque `ElephantSpawnerServer.lua` e `DuplicateServer.lua`.
5. Em `StarterGui` crie botões e adicione os LocalScripts em `client/`.
6. Adapte `InventoryModule.lua` se você usa DataStores ou outro inventário.

Leia os comentários de cada arquivo para instruções detalhadas.

## Git / GitHub
Veja as instruções no final do README para criar e subir o repositório ao GitHub.

## Licença
MITMIT License

Copyright (c) 2025 SeuNome

Permission is hereby granted, free of charge, to any person obtaining a copy
...
(standard MIT license text — substitua "SeuNome" pelo seu nome)2025# ignore system files
.DS_Store
Thumbs.db

# ignore Studio/Roblox local files (if any)
rbxlx
rbxl
*.tmpThumbs.db-- ElephantSpawnerServer.lua
-- Spawna elefantes periodicamente e dá Brainrot a quem matou.
-- Coloque em ServerScriptService.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")

-- CONFIG
local INTERVAL_SECONDS = 60 * 60 * 2  -- 2 horas default (para testes use 60 or 300)
local TEMPLATE_FOLDER_NAME = "ElephantTemplates" -- folder em ReplicatedStorage
local SPAWN_FOLDER_NAME = "ElephantSpawns"       -- folder em Workspace
local BRAINROT_ITEM_NAME = "Brainrot"
local MAX_ACTIVE = 3

-- Events opcionais
local ElephantSpawnedEvent = ReplicatedStorage:FindFirstChild("ElephantSpawnedEvent")

-- localizar templates e spawns
local templatesFolder = ReplicatedStorage:FindFirstChild(TEMPLATE_FOLDER_NAME)
if not templatesFolder then
    warn("ElephantSpawner: Folder '"..TEMPLATE_FOLDER_NAME.."' não encontrada em ReplicatedStorage.")
    return
end
local spawnFolder = Workspace:FindFirstChild(SPAWN_FOLDER_NAME)
if not spawnFolder then
    warn("ElephantSpawner: Folder '"..SPAWN_FOLDER_NAME.."' não encontrada em Workspace.")
    return
end

local templates = templatesFolder:GetChildren()
local spawnPoints = spawnFolder:GetChildren()
if #templates == 0 then warn("ElephantSpawner: nenhum template encontrado.") return end
if #spawnPoints == 0 then warn("ElephantSpawner: nenhum spawn point encontrado.") return end

-- ativo
local activeNPCs = {}

local function findNearestPlayer(position, maxDistance)
    local bestPlayer = nil
    local bestDist = maxDistance or 30
    for _, p in pairs(Players:GetPlayers()) do
        local char = p.Character
        local humanoid = char and char:FindFirstChildWhichIsA("Humanoid")
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if humanoid and hrp and humanoid.Health > 0 then
            local dist = (hrp.Position - position).Magnitude
            if dist < bestDist then
                bestDist = dist
                bestPlayer = p
            end
        end
    end
    return bestPlayer
end

local function addItemToPlayer(player, itemName, amount)
    amount = amount or 1
    local inv = player:FindFirstChild("Inventory")
    if not inv then
        inv = Instance.new("Folder")
        inv.Name = "Inventory"
        inv.Parent = player
    end
    local obj = inv:FindFirstChild(itemName)
    if not obj then
        obj = Instance.new("IntValue")
        obj.Name = itemName
        obj.Value = 0
        obj.Parent = inv
    end
    obj.Value = obj.Value + amount
end

local function hookNPC(npc)
    if not npc then return end
    local humanoid = npc:FindFirstChildWhichIsA("Humanoid", true)
    if not humanoid then return end

    activeNPCs[npc] = true

    humanoid.Died:Connect(function()
        local giverPlayer = nil
        local creator = humanoid:FindFirstChild("LastAttacker") or humanoid:FindFirstChild("creator")
        if creator and creator.Value then
            if typeof(creator.Value) == "Instance" then
                if creator.Value:IsA("Player") then
                    giverPlayer = creator.Value
                elseif creator.Value:IsA("Model") then
                    local pl = Players:GetPlayerFromCharacter(creator.Value)
                    if pl then giverPlayer = pl end
                end
            end
        end

        if not giverPlayer then
            local pos = npc:FindFirstChild("HumanoidRootPart") and npc.HumanoidRootPart.Position or npc:GetModelCFrame().p
            giverPlayer = findNearestPlayer(pos, 30)
        end

        if giverPlayer then
            addItemToPlayer(giverPlayer, BRAINROT_ITEM_NAME, 1)
            print(string.format("%s recebeu 1 %s por matar %s", giverPlayer.Name, BRAINROT_ITEM_NAME, npc.Name))
        else
            print("Nenhum jogador elegível para receber Brainrot (npc: "..tostring(npc.Name)..")")
        end

        activeNPCs[npc] = nil
    end)
end

local function spawnElephant(template, spawnPoint)
    local clone = template:Clone()
    local primary = clone:FindFirstChild("HumanoidRootPart") or clone:FindFirstChildWhichIsA("BasePart")
    if not primary then
        warn("Template "..template.Name.." não tem HumanoidRootPart/PrimaryPart.")
        return nil
    end
    clone:SetPrimaryPartCFrame(spawnPoint.CFrame + Vector3.new(0, 2, 0))
    clone.Parent = Workspace
    clone.Name = template.Name .. "_" .. tostring(math.random(1000,9999))

    local humanoid = clone:FindFirstChildWhichIsA("Humanoid", true)
    if humanoid then
        humanoid.MaxHealth = humanoid.MaxHealth or 400
        humanoid.Health = humanoid.MaxHealth
    end

    hookNPC(clone)

    Debris:AddItem(clone, 3600)
    if ElephantSpawnedEvent then
        pcall(function()
            ElephantSpawnedEvent:FireAllClients(clone.Name, template.Name, spawnPoint.Name)
        end)
    end
    return clone
end

spawn(function()
    while true do
        local activeCount = 0
        for _ in pairs(activeNPCs) do activeCount = activeCount + 1 end

        if activeCount < MAX_ACTIVE then
            local template = templates[math.random(1, #templates)]
            local spawnPoint = spawnPoints[math.random(1, #spawnPoints)]
            local npc = spawnElephant(template, spawnPoint)
            if npc then
                print("Elefante spawned:", npc.Name, "template:", template.Name)
            end
        else
            print("Máximo de elefantes ativos atingido:", activeCount)
        end

        local waited = 0
        while waited < INTERVAL_SECONDS do
            local s = math.min(10, INTERVAL_SECONDS - waited)
            wait(s)
            waited = waited + s
        end
    end
end)6023030p.Characterhumanoid.Health1inv.Nameinv.Parentobj.Nameobj.Valueobj.Parentcreator.Valuenpc.HumanoidRootPart.PositiongiverPlayer.Namenpc.Nametemplate.Nameclone.Parentclone.Namehumanoid.MaxHealth400spawnPoint.Name
