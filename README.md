-- 高级ESP脚本 v2.3 - 优化版（监测范围1000单位）
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

-- ===== 配置区域 =====
local SETTINGS = {
    -- 敌人设置
    ENEMY_TAGS = {"Rake", "Monster", "Enemy"},
    ENEMY_COLOR = Color3.fromRGB(255, 50, 50),
    
    -- 玩家设置
    PLAYER_COLOR = Color3.fromRGB(50, 150, 255),
    TEAM_COLOR = Color3.fromRGB(50, 255, 100),
    
    -- 显示设置
    SHOW_HEALTH = true,
    SHOW_DISTANCE = true,
    SHOW_TRACER = true,
    TRACER_COLOR = Color3.fromRGB(0, 150, 255),
    ENEMY_TRACER_COLOR = Color3.fromRGB(255, 50, 50),
    TRACER_THICKNESS = 0.1,
    
    -- 高级设置
    MAX_DISTANCE = 1000,  -- 已修改为1000单位
    HEALTH_TEXT_SIZE = 16,
    DISTANCE_TEXT_SIZE = 14,
    NAME_TEXT_SIZE = 18,
    UPDATE_RATE = 0.1,
    
    -- 新增设置
    SCAN_INTERVAL = 5,
    TRACER_FOR_ENEMIES = true
}

-- ===== 核心功能 =====
local espCache = {}
local lastScanTime = 0

local function createESP(target)
    if not target or not target:FindFirstChild("HumanoidRootPart") then return end
    
    if espCache[target] then return end
    
    local isEnemy = false
    local isTeammate = false
    local espColor = SETTINGS.PLAYER_COLOR
    
    for _, tag in ipairs(SETTINGS.ENEMY_TAGS) do
        if target.Name:find(tag) or target:FindFirstChild(tag) then
            isEnemy = true
            espColor = SETTINGS.ENEMY_COLOR
            break
        end
    end
    
    if not isEnemy and target:FindFirstChild("Team") then
        if LocalPlayer.Team and target.Team == LocalPlayer.Team then
            isTeammate = true
            espColor = SETTINGS.TEAM_COLOR
        end
    end
    
    local highlight = Instance.new("Highlight")
    highlight.Name = "AdvancedESP_Highlight"
    highlight.FillColor = espColor
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.FillTransparency = 0.3
    highlight.OutlineTransparency = 0
    highlight.Parent = target
    
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "AdvancedESP_Billboard"
    billboard.Adornee = target.HumanoidRootPart
    billboard.Size = UDim2.new(0, 150, 0, 60)
    billboard.StudsOffset = Vector3.new(0, 3.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = target
    
    local distanceText = Instance.new("TextLabel")
    distanceText.Name = "ESP_Distance"
    distanceText.Size = UDim2.new(1, 0, 0.3, 0)
    distanceText.Position = UDim2.new(0, 0, 0, 0)
    distanceText.BackgroundTransparency = 1
    distanceText.TextColor3 = Color3.fromRGB(200, 200, 255)
    distanceText.TextStrokeTransparency = 0.5
    distanceText.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    distanceText.TextSize = SETTINGS.DISTANCE_TEXT_SIZE
    distanceText.Font = Enum.Font.SourceSansBold
    distanceText.TextXAlignment = Enum.TextXAlignment.Center
    distanceText.Parent = billboard
    
    local healthText = Instance.new("TextLabel")
    healthText.Name = "ESP_Health"
    healthText.Size = UDim2.new(1, 0, 0.4, 0)
    healthText.Position = UDim2.new(0, 0, 0.3, 0)
    healthText.BackgroundTransparency = 1
    healthText.TextColor3 = Color3.fromRGB(255, 255, 255)
    healthText.TextStrokeTransparency = 0.5
    healthText.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    healthText.TextSize = SETTINGS.HEALTH_TEXT_SIZE
    healthText.Font = Enum.Font.SourceSansBold
    healthText.TextXAlignment = Enum.TextXAlignment.Center
    healthText.Parent = billboard
    
    local nameText = Instance.new("TextLabel")
    nameText.Name = "ESP_Name"
    nameText.Size = UDim2.new(1, 0, 0.3, 0)
    nameText.Position = UDim2.new(0, 0, 0.7, 0)
    nameText.BackgroundTransparency = 1
    nameText.TextColor3 = espColor
    nameText.TextStrokeTransparency = 0.5
    nameText.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    nameText.TextSize = SETTINGS.NAME_TEXT_SIZE
    nameText.Font = Enum.Font.SourceSansBold
    nameText.TextXAlignment = Enum.TextXAlignment.Center
    nameText.Parent = billboard
    
    local tracer
    if SETTINGS.SHOW_TRACER and (not isEnemy or SETTINGS.TRACER_FOR_ENEMIES) then
        local startAttachment = Instance.new("Attachment")
        startAttachment.Name = "TracerStart"
        startAttachment.Parent = workspace.CurrentCamera
        
        local endAttachment = Instance.new("Attachment")
        endAttachment.Name = "TracerEnd"
        endAttachment.Parent = target.HumanoidRootPart
        
        tracer = Instance.new("Beam")
        tracer.Name = "ESP_Tracer"
        tracer.Attachment0 = startAttachment
        tracer.Attachment1 = endAttachment
        tracer.Color = ColorSequence.new(isEnemy and SETTINGS.ENEMY_TRACER_COLOR or SETTINGS.TRACER_COLOR)
        tracer.Width0 = SETTINGS.TRACER_THICKNESS
        tracer.Width1 = SETTINGS.TRACER_THICKNESS * 0.8
        tracer.LightEmission = 0.8
        tracer.Transparency = NumberSequence.new(0.5)
        tracer.FaceCamera = true
        tracer.Parent = target.HumanoidRootPart
    end
    
    espCache[target] = {
        highlight = highlight,
        billboard = billboard,
        tracer = tracer,
        humanoid = target:FindFirstChildOfClass("Humanoid"),
        rootPart = target.HumanoidRootPart,
        startAttachment = tracer and tracer.Attachment0,
        endAttachment = tracer and tracer.Attachment1,
        isEnemy = isEnemy
    }
end

local function fullScan()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            createESP(player.Character)
        end
    end
    
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("Model") and descendant:FindFirstChild("HumanoidRootPart") then
            for _, tag in ipairs(SETTINGS.ENEMY_TAGS) do
                if descendant.Name:find(tag) or descendant:FindFirstChild(tag) then
                    createESP(descendant)
                    break
                end
            end
        end
    end
end

-- ===== 更新循环 =====
local lastUpdate = 0
RunService.Heartbeat:Connect(function(deltaTime)
    lastUpdate = lastUpdate + deltaTime
    lastScanTime = lastScanTime + deltaTime
    
    if lastScanTime >= SETTINGS.SCAN_INTERVAL then
        lastScanTime = 0
        fullScan()
    end
    
    if lastUpdate < SETTINGS.UPDATE_RATE then return end
    lastUpdate = 0
    
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return end
    local localRoot = LocalPlayer.Character.HumanoidRootPart
    
    for target, data in pairs(espCache) do
        if not target.Parent then
            if data.highlight then data.highlight:Destroy() end
            if data.billboard then data.billboard:Destroy() end
            if data.tracer then data.tracer:Destroy() end
            espCache[target] = nil
        else
            local distance = (data.rootPart.Position - localRoot.Position).Magnitude
            local shouldShow = distance <= SETTINGS.MAX_DISTANCE
            
            data.highlight.Enabled = shouldShow
            data.billboard.Enabled = shouldShow
            
            if data.tracer then
                data.tracer.Enabled = shouldShow
                if data.startAttachment then
                    data.startAttachment.WorldPosition = localRoot.Position - Vector3.new(0, 1.5, 0)
                end
            end
            
            if shouldShow then
                if SETTINGS.SHOW_DISTANCE then
                    data.billboard.ESP_Distance.Text = string.format("%.1fm", distance/3.571)
                else
                    data.billboard.ESP_Distance.Text = ""
                end
                
                if SETTINGS.SHOW_HEALTH and data.humanoid then
                    data.billboard.ESP_Health.Text = string.format("HP: %d/%d", 
                        math.floor(data.humanoid.Health), 
                        math.floor(data.humanoid.MaxHealth))
                else
                    data.billboard.ESP_Health.Text = ""
                end
                
                data.billboard.ESP_Name.Text = target.Name
            end
        end
    end
end)

-- ===== 初始化和事件监听 =====
fullScan()

Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function(character)
        createESP(character)
    end)
end)

workspace.DescendantAdded:Connect(function(descendant)
    if descendant:IsA("Model") and descendant:FindFirstChild("HumanoidRootPart") then
        for _, tag in ipairs(SETTINGS.ENEMY_TAGS) do
            if descendant.Name:find(tag) or descendant:FindFirstChild(tag) then
                createESP(descendant)
                break
            end
        end
    end
end)

LocalPlayer.CharacterRemoving:Connect(function()
    for _, data in pairs(espCache) do
        if data.highlight then data.highlight:Destroy() end
        if data.billboard then data.billboard:Destroy() end
        if data.tracer then data.tracer:Destroy() end
    end
    espCache = {}
end)# esp2.3
