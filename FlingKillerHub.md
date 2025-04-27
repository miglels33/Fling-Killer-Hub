-- Fling Killer GUI - By: Sonic Scripts
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")

-- SETTINGS (Adjust as needed)
local FLING_DURATION = 5     -- Fling duration in seconds
local BASE_DISTANCE = 1.5    -- Distance from target
local FLING_POWER = 500      -- Horizontal fling force (increase to throw farther)
local FLING_UP = 100         -- Vertical fling force (increase to throw higher)
local SPEED_MULTIPLIER = 4.0 -- Speed of back-and-forth movement

-- Create GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlingKillerGUI"
screenGui.Parent = player.PlayerGui

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 220, 0, 350)
mainFrame.Position = UDim2.new(0, 10, 0.5, -175)
mainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
mainFrame.BackgroundTransparency = 0.3
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 30)
title.Text = "FLING KILLER - Select Target:"
title.TextColor3 = Color3.new(1, 1, 1)
title.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
title.Parent = mainFrame

local playerList = Instance.new("ScrollingFrame")
playerList.Size = UDim2.new(1, -10, 1, -100)
playerList.Position = UDim2.new(0, 5, 0, 35)
playerList.BackgroundTransparency = 1
playerList.ScrollBarThickness = 6
playerList.Parent = mainFrame

local flingButton = Instance.new("TextButton")
flingButton.Size = UDim2.new(1, -10, 0, 40)
flingButton.Position = UDim2.new(0, 5, 1, -55)
flingButton.Text = "ACTIVATE FLING KILLER"
flingButton.TextColor3 = Color3.new(1, 1, 1)
flingButton.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
flingButton.Parent = mainFrame

local closeButton = Instance.new("TextButton")
closeButton.Size = UDim2.new(0, 25, 0, 25)
closeButton.Position = UDim2.new(1, -30, 0, 5)
closeButton.Text = "X"
closeButton.TextColor3 = Color3.new(1, 1, 1)
closeButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
closeButton.Parent = mainFrame

-- Variables
local selectedPlayer = nil
local isFlingActive = false

-- Refresh player list
local function refreshPlayerList()
    for _, child in ipairs(playerList:GetChildren()) do
        if child:IsA("TextButton") then
            child:Destroy()
        end
    end
    
    local yPos = 0
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            local playerButton = Instance.new("TextButton")
            playerButton.Size = UDim2.new(1, 0, 0, 32)
            playerButton.Position = UDim2.new(0, 0, 0, yPos)
            playerButton.Text = "▶ "..otherPlayer.Name
            playerButton.TextColor3 = Color3.new(1, 1, 1)
            playerButton.BackgroundColor3 = Color3.fromRGB(70, 70, 90)
            playerButton.Parent = playerList
            
            playerButton.MouseButton1Click:Connect(function()
                selectedPlayer = otherPlayer
                for _, btn in ipairs(playerList:GetChildren()) do
                    if btn:IsA("TextButton") then
                        btn.BackgroundColor3 = Color3.fromRGB(70, 70, 90)
                        btn.Text = btn.Text:gsub("✓ ", "▶ ")
                    end
                end
                playerButton.BackgroundColor3 = Color3.fromRGB(40, 120, 200)
                playerButton.Text = "✓ "..playerButton.Text:sub(3)
            end)
            
            yPos = yPos + 35
        end
    end
    
    playerList.CanvasSize = UDim2.new(0, 0, 0, yPos)
    
    if yPos == 0 then
        local msg = Instance.new("TextLabel")
        msg.Text = "No players available"
        msg.Size = UDim2.new(1, 0, 0, 30)
        msg.Position = UDim2.new(0, 0, 0, 0)
        msg.TextColor3 = Color3.new(0.8, 0.8, 0.8)
        msg.BackgroundTransparency = 1
        msg.Parent = playerList
    end
end

-- Teleport behind target
local function teleportBehind(target)
    if not target or not target.Character then return end
    
    local targetRoot = target.Character:FindFirstChild("HumanoidRootPart")
    if not targetRoot then return end
    
    local behindOffset = -targetRoot.CFrame.LookVector * BASE_DISTANCE
    humanoidRootPart.CFrame = CFrame.new(targetRoot.Position + behindOffset, targetRoot.Position)
end

-- Main fling effect
local function activateFlingKiller(target)
    if not target or not target.Character or isFlingActive then return end
    
    isFlingActive = true
    flingButton.Text = "FLING KILLER ACTIVE!"
    flingButton.BackgroundColor3 = Color3.fromRGB(220, 30, 30)
    
    local targetRoot = target.Character:FindFirstChild("HumanoidRootPart")
    if not targetRoot then
        isFlingActive = false
        flingButton.Text = "ACTIVATE FLING KILLER"
        flingButton.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
        return
    end
    
    -- Save original state
    local originalCFrame = humanoidRootPart.CFrame
    local originalCollision = humanoidRootPart.CanCollide
    humanoidRootPart.CanCollide = false
    
    -- Create velocity controller on target
    local targetVelocity = Instance.new("BodyVelocity")
    targetVelocity.Velocity = Vector3.new(0, 0, 0)
    targetVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    targetVelocity.Parent = targetRoot
    
    -- Control variables
    local startTime = time()
    local direction = -1
    local oscillationSpeed = 0
    
    -- Main fling loop
    local flingLoop
    flingLoop = RunService.Heartbeat:Connect(function(delta)
        if not targetRoot or (time() - startTime) > FLING_DURATION then
            flingLoop:Disconnect()
            return
        end
        
        -- Ultra-fast oscillation (back-and-forth)
        direction = direction * -1
        oscillationSpeed = oscillationSpeed + delta * SPEED_MULTIPLIER
        
        -- Calculate position
        local offset = targetRoot.CFrame.LookVector * direction * BASE_DISTANCE
        local verticalOffset = Vector3.new(0, math.sin(oscillationSpeed) * 0.5, 0)
        
        humanoidRootPart.CFrame = CFrame.new(
            targetRoot.Position + offset + verticalOffset,
            targetRoot.Position
        )
        
        -- Apply EXTREME force to target
        local forceDirection = (targetRoot.Position - humanoidRootPart.Position).Unit
        targetVelocity.Velocity = forceDirection * FLING_POWER + Vector3.new(0, FLING_UP, 0)
    end)
    
    -- Wait for completion
    repeat task.wait() until (time() - startTime) > FLING_DURATION
    
    -- Cleanup
    if targetVelocity then targetVelocity:Destroy() end
    humanoidRootPart.CFrame = originalCFrame
    humanoidRootPart.CanCollide = originalCollision
    
    isFlingActive = false
    flingButton.Text = "ACTIVATE FLING KILLER"
    flingButton.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
end

-- Connect events
flingButton.MouseButton1Click:Connect(function()
    if selectedPlayer and not isFlingActive then
        teleportBehind(selectedPlayer)
        activateFlingKiller(selectedPlayer)
    end
end)

closeButton.MouseButton1Click:Connect(function()
    screenGui:Destroy()
end)

-- Initialize
refreshPlayerList()
Players.PlayerAdded:Connect(refreshPlayerList)
Players.PlayerRemoving:Connect(refreshPlayerList)
