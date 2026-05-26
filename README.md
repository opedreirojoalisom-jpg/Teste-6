-- Main module
local AimBot = {}

-- Configuration
local config = {
    FOV_RADIUS = 100,          -- Field of view circle radius
    AIM_SENSITIVITY = 0.5,     -- Mouse sensitivity multiplier
    DETECTION_THRESHOLD = 50,  -- Minimum enemy size threshold
    TARGET_HEAD_ONLY = true,   -- Only target heads/upper body
    SHOW_CIRCLE = true,        -- Show circle visualization
}

-- State variables
local state = {
    enabled = false,
    fov_circle = nil,
    player = nil,
    camera = nil,
    enemies = {},
    last_target = nil,
}

-- Utility functions
local function screenToWorld(screenPos)
    local workspace = game:GetService("Workspace")
    local camera = workspace.CurrentCamera
    if not camera then return nil end
    
    local ray = Ray.new(
        camera.CFrame.Position,
        (screenPos - Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)).unit * 1000
    )
    
    local part, position = workspace:FindPartOnRay(ray)
    return part, position
end

local function getScreenPosition(worldPos)
    local workspace = game:GetService("Workspace")
    local camera = workspace.CurrentCamera
    if not camera then return Vector2.new() end
    
    local screenPos, onScreen = camera:WorldToViewportPoint(worldPos)
    return Vector2.new(screenPos.X, screenPos.Y)
end

-- UI Components
local function createUI()
    -- Create screen gui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AimBotGUI"
    screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")
    
    -- Create circle overlay
    local circle = Instance.new("ImageLabel")
    circle.Name = "FOVCircle"
    circle.Image = "rbxasset://textures/ui/AimBotCircle.png"
    circle.Size = UDim2.new(0, config.FOV_RADIUS * 2, 0, config.FOV_RADIUS * 2)
    circle.Visible = false
    circle.BackgroundTransparency = 1
    circle.BorderSizePixel = 0
    circle.Parent = screenGui
    
    -- Create status label
    local status = Instance.new("TextLabel")
    status.Name = "Status"
    status.Text = "AimBot: OFF"
    status.TextScaled = true
    status.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    status.BackgroundTransparency = 0.3
    status.Size = UDim2.new(0, 200, 0, 50)
    status.Position = UDim2.new(0.5, -100, 0.1, 0)
    status.TextColor3 = Color3.fromRGB(255, 255, 255)
    status.TextStrokeTransparency = 0.5
    status.Parent = screenGui
    
    -- Create toggle button
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Name = "ToggleAimBot"
    toggleBtn.Text = "TOGGLE AIMBOT"
    toggleBtn.TextScaled = true
    toggleBtn.Size = UDim2.new(0, 150, 0, 50)
    toggleBtn.Position = UDim2.new(0.5, -75, 0.2, 0)
    toggleBtn.BackgroundColor3 = Color3.fromRGB(50, 120, 255)
    toggleBtn.MouseButton1Click:Connect(function()
        state.enabled = not state.enabled
        status.Text = "AimBot: " .. (state.enabled and "ON" or "OFF")
        circle.Visible = state.enabled and config.SHOW_CIRCLE
    end)
    toggleBtn.Parent = screenGui
    
    return circle, status, toggleBtn
end

-- Enemy detection system
local function detectEnemies()
    local players = game:GetService("Players"):GetPlayers()
    local workspace = game:GetService("Workspace")
    local camera = workspace.CurrentCamera
    if not camera then return {} end
    
    local enemies = {}
    
    for _, player in pairs(players) do
        if player ~= game.Players.LocalPlayer then
            local character = player.Character
            if character and character:FindFirstChild("HumanoidRootPart") then
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                local head = character:FindFirstChild("Head")
                
                -- Calculate distance to player
                local distance = (rootPart.Position - camera.CFrame.Position).Magnitude
                
                -- Check if player is in FoV
                local screenPos = getScreenPosition(rootPart.Position)
                local mousePos = Vector2.new(workspace.CurrentCamera.ViewportSize.X/2, workspace.CurrentCamera.ViewportSize.Y/2)
                local distToCenter = (screenPos - mousePos).Magnitude
                
                if distToCenter <= config.FOV_RADIUS then
                    local size = getScreenPosition(head.Position).sub(getScreenPosition(rootPart.Position)).Magnitude
                    
                    if size > config.DETECTION_THRESHOLD then
                        table.insert(enemies, {
                            player = player,
                            character = character,
                            distance = distance,
                            head = head,
                            screenPos = screenPos,
                            size = size
                        })
                    end
                end
            end
        end
    end
    
    -- Sort by distance (closest first)
    table.sort(enemies, function(a, b)
        return a.distance < b.distance
    end)
    
    return enemies
end

-- Aiming system
local function aimAtEnemy(enemy)
    if not enemy.head then return end
    
    local workspace = game:GetService("Workspace")
    local camera = workspace.CurrentCamera
    if not camera then return end
    
    local screenPos = getScreenPosition(enemy.head.Position)
    local mousePos = Vector2.new(workspace.CurrentCamera.ViewportSize.X/2, workspace.CurrentCamera.ViewportSize.Y/2)
    
    -- Calculate aim offset
    local offsetX = (screenPos.X - mousePos.X) * config.AIM_SENSITIVITY
    local offsetY = (screenPos.Y - mousePos.Y) * config.AIM_SENSITIVITY
    
    -- Apply aim correction
    local newCFrame = camera.CFrame * CFrame.new(offsetX, offsetY, 0)
    camera.CFrame = newCFrame
end

-- Main update loop
local function update()
    while true do
        wait(0.05)  -- 20 FPS
        
        if state.enabled then
            -- Detect enemies
            state.enemies = detectEnemies()
            
            if #state.enemies > 0 then
                -- Aim at closest enemy
                aimAtEnemy(state.enemies[1])
                
                -- Update last target
                state.last_target = state.enemies[1]
            end
        end
    end
end

-- Initialize AimBot
function AimBot.Initialize()
    -- Create UI components
    local circle, status, toggleBtn = createUI()
    state.fov_circle = circle
    state.status = status
    state.toggle_btn = toggleBtn
    
    -- Set initial visibility
    circle.Visible = false
    
    -- Start update loop
    spawn(update)
    
    print("Roblox FPS AimBot initialized!")
    print("Use the TOGGLE AIMBOT button to enable/disable")
end

return AimBot
