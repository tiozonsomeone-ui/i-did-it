# i-did-it # Folder: ReplicatedStorage/TechsModule.lua

local TechsModule = {}

TechsModule.Techs = {
    {
        id = "dash_strike",
        name = "Dash Strike",
        sequence = {"E", "Mouse1"},
        window = 0.6,
        cooldown = 1.2,
        staminaCost = 10,
    },
    {
        id = "spin_tech",
        name = "Spin Tech",
        sequence = {"Q", "Q", "Mouse1"},
        window = 1.0,
        cooldown = 3.0,
        staminaCost = 20,
    },
}

TechsModule._byId = {}
for _, t in ipairs(TechsModule.Techs) do
    TechsModule._byId[t.id] = t
end

function TechsModule.FindMatchingTech(buffer)
    if not buffer or #buffer == 0 then return nil end
    for _, tech in ipairs(TechsModule.Techs) do
        local seq = tech.sequence
        local L = #seq
        if #buffer >= L then
            local ok = true
            local startIndex = #buffer - L + 1
            if buffer[#buffer].time - buffer[startIndex].time > tech.window then
                ok = false
            else
                for i = 1, L do
                    local bufEntry = buffer[startIndex + i - 1]
                    if not bufEntry or bufEntry.key ~= seq[i] then
                        ok = false; break
                    end
                end
            end
            if ok then return tech end
        end
    end
    return nil
end

return TechsModule


# Folder: StarterPlayerScripts/ComboClient.lua

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

local Techs = require(ReplicatedStorage:WaitForChild("TechsModule"))
local DoTechEvent = ReplicatedStorage:WaitForChild("DoTechEvent")

local INPUT_BUFFER_MAX = 12
local buffer = {}

local function pushInput(key)
    table.insert(buffer, {key = key, time = tick()})
    if #buffer > INPUT_BUFFER_MAX then table.remove(buffer, 1) end
    local match = Techs.FindMatchingTech(buffer)
    if match then
        DoTechEvent:FireServer(match.id)
        buffer = {}
    end
end

UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        pushInput(input.KeyCode.Name)
    elseif input.UserInputType == Enum.UserInputType.MouseButton1 then
        pushInput("Mouse1")
    elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
        pushInput("Mouse2")
    end
end)

DoTechEvent.OnClientEvent:Connect(function(eventType, data)
end)


# Folder: ServerScriptService/TechServer.lua

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local Techs = require(ReplicatedStorage:WaitForChild("TechsModule"))
local DoTechEvent = ReplicatedStorage:WaitForChild("DoTechEvent")

local playerState = {}
local DEFAULT_STAMINA = 100

local function ensureState(plr)
    local id = plr.UserId
    if not playerState[id] then
        playerState[id] = {cooldowns = {}, stamina = DEFAULT_STAMINA}
    end
    return playerState[id]
end

local function canPerformTech(plr, tech)
    local st = ensureState(plr)
    local cd = st.cooldowns[tech.id]
    if cd and tick() < cd then return false, "On cooldown" end
    if st.stamina < (tech.staminaCost or 0) then return false, "Not enough stamina" end
    return true
end

local function applyCooldown(plr, tech)
    local st = ensureState(plr)
    st.cooldowns[tech.id] = tick() + (tech.cooldown or 0)
    st.stamina = math.max(0, st.stamina - (tech.staminaCost or 0))
end

DoTechEvent.OnServerEvent:Connect(function(player, techId)
    local tech = Techs._byId[techId]
    if not tech then
        DoTechEvent:FireClient(player, "Rejected", "Invalid tech")
        return
    end

    local ok, reason = canPerformTech(player, tech)
    if not ok then
        DoTechEvent:FireClient(player, "Rejected", reason)
        return
    end

    applyCooldown(player, tech)

    local char = player.Character
    if char and char:FindFirstChild("Humanoid") then
        local humanoid = char.Humanoid

        local animationsFolder = ReplicatedStorage:FindFirstChild("TechAnimations")
        if animationsFolder and animationsFolder:FindFirstChild(tech.id) then
            local anim = animationsFolder[tech.id]
            local track = humanoid:LoadAnimation(anim)
            track:Play()
        end

        local root = char:FindFirstChild("HumanoidRootPart")
        if root then
            local origin = root.Position
            local direction = root.CFrame.LookVector * 4
            local params = RaycastParams.new()
            params.FilterDescendantsInstances = {char}
            params.FilterType = Enum.RaycastFilterType.Blacklist
            local ray = workspace:Raycast(origin, direction, params)
            if ray and ray.Instance and ray.Instance.Parent then
                local hitChar = ray.Instance.Parent
                local hitHum = hitChar:FindFirstChild("Humanoid")
                if hitHum and hitHum.Health > 0 then
                    hitHum:TakeDamage(10)
                end
            end
        end
    end

    DoTechEvent:FireClient(player, "Accepted", {techId = techId})
end)


# README (Suggested)
-- Roblox Combo / Tech System
-- Add TechsModule.lua into ReplicatedStorage.
-- Add ComboClient.lua into StarterPlayerScripts.
-- Add TechServer.lua into ServerScriptService.
-- Create a RemoteEvent named "DoTechEvent" in ReplicatedStorage.
-- Create a folder named "TechAnimations" in ReplicatedStorage and add Animation objects named after tech IDs.
-- Customize techs inside TechsModule.lua.
-- Roblox Combo / Tech System (split files)
-- 3 files below: ModuleScript, LocalScript, ServerScript.
-- Put ModuleScript in ReplicatedStorage named "TechsModule".
-- Put RemoteEvent in ReplicatedStorage named "DoTechEvent".
-- Put a Folder in ReplicatedStorage named "TechAnimations" containing Animation instances named after tech IDs.

-- ===== ModuleScript: ReplicatedStorage/TechsModule =====
local TechsModule = {}

-- Define your techs here. Edit sequences, windows (seconds), cooldowns, staminaCost, and other metadata.
-- sequence is an array of strings representing input names: keyboard keys ("Q","E","R"...), "Mouse1", "Mouse2".
TechsModule.Techs = {
    {
        id = "dash_strike",
        name = "Dash Strike",
        sequence = {"E", "Mouse1"},
        window = 0.6,
        cooldown = 1.2,
        staminaCost = 10,
    },
    {
        id = "spin_tech",
        name = "Spin Tech",
        sequence = {"Q", "Q", "Mouse1"},
        window = 1.0,
        cooldown = 3.0,
        staminaCost = 20,
    },
}

-- internal lookup
TechsModule._byId = {}
for _, t in ipairs(TechsModule.Techs) do
    TechsModule._byId[t.id] = t
end

-- Finds the first matching tech from a buffer (buffer: array of {key, time}, oldest first)
function TechsModule.FindMatchingTech(buffer)
    if not buffer or #buffer == 0 then return nil end
    for _, tech in ipairs(TechsModule.Techs) do
        local seq = tech.sequence
        local L = #seq
        if #buffer >= L then
            local ok = true
            local startIndex = #buffer - L + 1
            if buffer[#buffer].time - buffer[startIndex].time > tech.window then
                ok = false
            else
                for i = 1, L do
                    local bufEntry = buffer[startIndex + i - 1]
                    if not bufEntry or bufEntry.key ~= seq[i] then
                        ok = false; break
                    end
                end
            end
            if ok then return tech end
        end
    end
    return nil
end

return TechsModule


-- ===== LocalScript: StarterPlayerScripts/ComboClient =====
-- Place as a LocalScript under StarterPlayerScripts
-- Requires: ReplicatedStorage.DoTechEvent (RemoteEvent) and ReplicatedStorage.TechsModule (ModuleScript)

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

local Techs = require(ReplicatedStorage:WaitForChild("TechsModule"))
local DoTechEvent = ReplicatedStorage:WaitForChild("DoTechEvent")

local INPUT_BUFFER_MAX = 12
local buffer = {} -- list of {key=string, time=number}, oldest first

local function pushInput(key)
    table.insert(buffer, {key = key, time = tick()})
    if #buffer > INPUT_BUFFER_MAX then table.remove(buffer, 1) end
    local match = Techs.FindMatchingTech(buffer)
    if match then
        DoTechEvent:FireServer(match.id)
        buffer = {} -- clear to prevent repeat triggers
    end
end

UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        pushInput(input.KeyCode.Name)
    elseif input.UserInputType == Enum.UserInputType.MouseButton1 then
        pushInput("Mouse1")
    elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
        pushInput("Mouse2")
    end
end)

-- Optional: listen for server responses to show HUD feedback
DoTechEvent.OnClientEvent:Connect(function(eventType, data)
    -- eventType: "Accepted" or "Rejected"
    -- Implement GUI feedback as needed
end)


-- ===== ServerScript: ServerScriptService/TechServer =====
-- Place as a server Script under ServerScriptService
-- Requires: ReplicatedStorage.DoTechEvent (RemoteEvent) and ReplicatedStorage.TechsModule

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Techs = require(ReplicatedStorage:WaitForChild("TechsModule"))
local DoTechEvent = ReplicatedStorage:WaitForChild("DoTechEvent")

local playerState = {} -- per-player state: {cooldowns = {}, stamina = number}
local DEFAULT_STAMINA = 100

local function ensureState(plr)
    local id = plr.UserId
    if not playerState[id] then
        playerState[id] = {cooldowns = {}, stamina = DEFAULT_STAMINA}
    end
    return playerState[id]
end

local function canPerformTech(plr, tech)
    local st = ensureState(plr)
    local cd = st.cooldowns[tech.id]
    if cd and tick() < cd then return false, "On cooldown" end
    if st.stamina < (tech.staminaCost or 0) then return false, "Not enough stamina" end
    return true
end

local function applyCooldown(plr, tech)
    local st = ensureState(plr)
    st.cooldowns[tech.id] = tick() + (tech.cooldown or 0)
    st.stamina = math.max(0, st.stamina - (tech.staminaCost or 0))
end

DoTechEvent.OnServerEvent:Connect(function(player, techId)
    local tech = Techs._byId[techId]
    if not tech then
        DoTechEvent:FireClient(player, "Rejected", "Invalid tech")
        return
    end

    local ok, reason = canPerformTech(player, tech)
    if not ok then
        DoTechEvent:FireClient(player, "Rejected", reason)
        return
    end

    -- Apply cooldown / cost
    applyCooldown(player, tech)

    -- Execute tech effects server-side (animations, damage, movement). Keep authoritative checks here.
    local char = player.Character
    if char and char:FindFirstChild("Humanoid") then
        local humanoid = char.Humanoid
        -- Play animation if present
        local animationsFolder = ReplicatedStorage:FindFirstChild("TechAnimations")
        if animationsFolder and animationsFolder:FindFirstChild(tech.id) then
            local anim = animationsFolder[tech.id]
            local track = humanoid:LoadAnimation(anim)
            track:Play()
        end

        -- Example placeholder: simple hit detection in front using Raycast
        -- (Replace this with proper hitboxes/Region3/raycast tailored to your game)
        local root = char:FindFirstChild("HumanoidRootPart")
        if root then
            local origin = root.Position
            local direction = root.CFrame.LookVector * 4 -- 4 studs forward
            local params = RaycastParams.new()
            params.FilterDescendantsInstances = {char}
            params.FilterType = Enum.RaycastFilterType.Blacklist
            local ray = workspace:Raycast(origin, direction, params)
            if ray and ray.Instance and ray.Instance.Parent then
                local hitChar = ray.Instance.Parent
                local hitHum = hitChar:FindFirstChild("Humanoid")
                if hitHum and hitHum.Health > 0 then
                    -- apply damage (example)
                    hitHum:TakeDamage(10)
                end
            end
        end
    end

    DoTechEvent:FireClient(player, "Accepted", {techId = techId})
end)


-- ===== Notes & Next steps =====
-- 1) Split these three sections into the corresponding files in Roblox Studio.
-- 2) Add Animation objects under ReplicatedStorage.TechAnimations named exactly as each tech.id.
-- 3) Balance: adjust windows, cooldowns, stamina costs.
-- 4) Improve detection: use hitboxes, Region3, or custom server-side checks to avoid cheats.
-- 5) UI: create a client GUI to show stamina, cooldowns, and feedback from the server events.

-- If you'd like, I can now:
-- A) Produce the three files as separate copy-paste blocks (ready for Roblox Studio). 
-- B) Add 6 extra example techs with sequences and suggested animation names.
-- C) Replace the placeholder raycast with a Region3 hitbox system and provide the code.
-- D) Provide step-by-step paste/install instructions for Roblox Studio (exactly where to place each file).

-- Tell me which of A/B/C/D you want and I will update the document accordingly.
