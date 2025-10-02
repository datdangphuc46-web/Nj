-- ClientStamina (LocalScript in StarterPlayerScripts)
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ContextActionService = game:GetService("ContextActionService")

local player = Players.LocalPlayer
local character = nil

local StaminaEvent = ReplicatedStorage:WaitForChild("StaminaEvent")

-- CONFIG
local MAX_STAMINA = 100
local SPRINT_DRAIN_PER_SEC = 20        -- stamina drained per second while sprinting
local SPRINT_SPEED_MULTIPLIER = 1.7    -- sprint speed relative to WalkSpeed
local REGEN_PER_SEC = 15               -- stamina regen per sec when not sprinting, after delay
local REGEN_DELAY = 1.2                -- seconds to wait after stopping sprint before regen starts
local MIN_STAMINA_TO_SPRINT = 5        -- must have at least this to start sprint

-- UI setup (create if not present)
local playerGui = player:WaitForChild("PlayerGui")
local screenGui = playerGui:FindFirstChild("StaminaGui")
if not screenGui then
    screenGui = Instance.new("ScreenGui")
    screenGui.Name = "StaminaGui"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui

    local bg = Instance.new("Frame", screenGui)
    bg.Name = "BG"
    bg.AnchorPoint = Vector2.new(0, 1)
    bg.Position = UDim2.new(0.02, 0, 0.95, 0)
    bg.Size = UDim2.new(0.25, 0, 0.04, 0)
    bg.BackgroundTransparency = 0.5
    bg.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    bg.BorderSizePixel = 0
    bg.ZIndex = 2

    local bar = Instance.new("Frame", bg)
    bar.Name = "Bar"
    bar.AnchorPoint = Vector2.new(0, 0.5)
    bar.Position = UDim2.new(0, 0, 0.5, 0)
    bar.Size = UDim2.new(1, -4, 0.8, 0)
    bar.Position = UDim2.new(0, 2, 0.5, 0)
    bar.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
    bar.BorderSizePixel = 0

    local filler = Instance.new("Frame", bar)
    filler.Name = "Fill"
    filler.AnchorPoint = Vector2.new(0, 0.5)
    filler.Position = UDim2.new(0, 0, 0.5, 0)
    filler.Size = UDim2.new(1, 0, 1, 0)
    filler.BackgroundTransparency = 0
    filler.BorderSizePixel = 0

    local text = Instance.new("TextLabel", bg)
    text.Name = "Text"
    text.AnchorPoint = Vector2.new(0.5, 0.5)
    text.Position = UDim2.new(0.5, 0, 0.5, 0)
    text.Size = UDim2.new(1, 0, 1, 0)
    text.BackgroundTransparency = 1
    text.Text = "STA: 100/100"
    text.TextScaled = true
    text.TextColor3 = Color3.fromRGB(255,255,255)
    text.ZIndex = 3
end

local fill = screenGui:WaitForChild("BG"):WaitForChild("Bar"):WaitForChild("Fill")
local textLabel = screenGui:BG:WaitForChild("Text")

-- local stamina state (client-side predicted)
local stamina = MAX_STAMINA
local isSprinting = false
local lastSprintStop = -999
local lastUpdateTick = tick()

-- apply sprint to humanoid
local function setSprintEnabled(enabled)
    local char = player.Character or player.CharacterAdded:Wait()
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    if enabled and stamina > MIN_STAMINA_TO_SPRINT then
        humanoid.WalkSpeed = humanoid.WalkSpeed * SPRINT_SPEED_MULTIPLIER
    else
        -- reset to default 16 (if changed elsewhere this will override)
        humanoid.WalkSpeed = 16
    end
end

-- better approach: store base speed
local baseWalkSpeed = 16
local function updateWalkSpeed()
    local char = player.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    if isSprinting and stamina > MIN_STAMINA_TO_SPRINT then
        humanoid.WalkSpeed = baseWalkSpeed * SPRINT_SPEED_MULTIPLIER
    else
        humanoid.WalkSpeed = baseWalkSpeed
    end
end

-- input action for sprint (toggle while hold)
local SPRINT_ACTION = "SprintAction"
local function sprintAction(actionName, inputState, inputObject)
    if inputState == Enum.UserInputState.Begin then
        -- start attempt to sprint
        if stamina > MIN_STAMINA_TO_SPRINT then
            isSprinting = true
            lastUpdateTick = tick()
            updateWalkSpeed()
        end
    elseif inputState == Enum.UserInputState.End then
        isSprinting = false
        lastSprintStop = tick()
        updateWalkSpeed()
    end
    return Enum.ContextActionResult.Sink
end

-- bind shift (mobile/keyboard safe)
ContextActionService:BindActionAtPriority(SPRINT_ACTION, sprintAction, false, Enum.ContextActionPriority.High.Value, Enum.KeyCode.LeftShift, Enum.UserInputType.Touch)

-- Accept also left control or gamepad (optional)
UserInputService.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == Enum.KeyCode.LeftControl then
        if stamina > MIN_STAMINA_TO_SPRINT then
            isSprinting = true
            updateWalkSpeed()
        end
    end
end)
UserInputService.InputEnded:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode == Enum.KeyCode.LeftControl then
        isSprinting = false
        lastSprintStop = tick()
        updateWalkSpeed()
    end
end)

-- Update UI helper
local function updateUI()
    local pct = math.clamp(stamina / MAX_STAMINA, 0, 1)
    fill.Size = UDim2.new(pct, 0, 1, 0)
    textLabel.Text = string.format("STA: %d/%d", math.floor(stamina+0.5), MAX_STAMINA)
end

-- Receive updates from server (authoritative)
StaminaEvent.OnClientEvent:Connect(function(action, arg)
    if action == "StaminaUpdate" then
        local serverVal = tonumber(arg) or 0
        stamina = math.clamp(serverVal, 0, MAX_STAMINA)
        updateUI()
    end
end)

-- initial request for sync
StaminaEvent:FireServer("RequestFullSync")

-- Main loop for client-side prediction (visual, and send small requests to server)
local accumSend = 0
RunService.RenderStepped:Connect(function(dt)
    -- if sprinting, drain
    if isSprinting and stamina > 0 then
        local drain = SPRINT_DRAIN_PER_SEC
