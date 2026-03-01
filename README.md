# Script
-- Script MOBILE ULTRA++ para 💪Muscle Masters (Delta 2026) 📱
-- CHASE (sigue + punch closest vivo) | FLY (swim anim, no freeze off) | ESP | TP ALL
-- AUTO PUNCH: Legends-style (equip tool + activate loop, o remote si encuentra) 🔥
-- Para si muere (switch auto) | Fly como swim (state Swimming) | Off: Freefall no freeze

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local chaseEnabled = false
local flyEnabled = false
local punchEnabled = false
local espEnabled = false
local chaseSpeed = 80
local flySpeed = 100
local punchRate = 0.0000001  -- Segs entre punches (rápido pero anti-kick)
local MAX_DIST = 10000099999
local PUNCH_DIST = 25
local bodyVelocity = nil
local espData = {}
local espFolder = nil
local punchConnection = nil

-- =============================================
-- Closest vivo
-- =============================================
local function getClosestPlayer()
    local closest, minDist = nil, MAX_DIST
    local myRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character then
            local root = plr.Character:FindFirstChild("HumanoidRootPart")
            local humanoid = plr.Character:FindFirstChild("Humanoid")
            if root and humanoid and humanoid.Health > 0 then
                local dist = (root.Position - myRoot.Position).Magnitude
                if dist < minDist and dist > 10 then
                    minDist = dist
                    closest = root
                end
            end
        end
    end
    return closest
end

-- =============================================
-- Cleanup movimiento + state
-- =============================================
local function cleanupMovement()
    chaseEnabled = false
    flyEnabled = false
    if bodyVelocity then
        bodyVelocity:Destroy()
        bodyVelocity = nil
    end
    local myHumanoid = player.Character and player.Character:FindFirstChild("Humanoid")
    if myHumanoid then
        myHumanoid:ChangeState(Enum.HumanoidStateType.Freefall)
    end
end

-- =============================================
-- LOOP MOVIMIENTO
-- =============================================
RunService.Heartbeat:Connect(function()
    local myChar = player.Character
    if not myChar or not myChar:FindFirstChild("HumanoidRootPart") then return end
    
    local myRoot = myChar.HumanoidRootPart
    local myHumanoid = myChar:FindFirstChild("Humanoid")
    local dir = nil
    local speed = 0
    
    if chaseEnabled then
        local target = getClosestPlayer()
        if target then
            dir = (target.Position - myRoot.Position).Unit
            speed = chaseSpeed
            camera.CFrame = CFrame.lookAt(camera.CFrame.Position, target.Position)
        end
    elseif flyEnabled then
        dir = camera.CFrame.LookVector
        speed = flySpeed
        if myHumanoid then
            myHumanoid:ChangeState(Enum.HumanoidStateType.Swimming)  -- Swim anim en aire!
        end
    end
    
    if dir then
        if not bodyVelocity then
            bodyVelocity = Instance.new("BodyVelocity")
            bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5)
            bodyVelocity.Parent = myRoot
        end
        local vel = dir * speed
        if chaseEnabled then vel += Vector3.new(0, 8, 0) end
        bodyVelocity.Velocity = vel
    elseif bodyVelocity then
        bodyVelocity.Velocity = Vector3.new(0, 0, 0)
    end
end)

-- =============================================
-- AUTO PUNCH Legends-style (tool activate + fallback remote)
-- =============================================
local function togglePunch(enable)
    punchEnabled = enable
    if punchConnection then punchConnection:Disconnect() punchConnection = nil end
    
    if enable then
        punchConnection = RunService.Heartbeat:Connect(function()
            local myChar = player.Character
            if not myChar then return end
            
            -- Encontrar punch tool (como en Legends/Masters)
            local punchTool = myChar:FindFirstChildOfClass("Tool") or player.Backpack:FindFirstChildOfClass("Tool")
            if punchTool and punchTool.Name:find("Punch") or punchTool.Name:find("Fist") then  -- Ajusta nombre si difiere
                if not myChar:FindFirstChild(punchTool.Name) then
                    myChar.Humanoid:EquipTool(punchTool)
                end
                punchTool:Activate()
            else
                -- Fallback: Buscar remote de punch (común en simulators)
                local remotes = game.ReplicatedStorage:GetDescendants()
                for _, remote in ipairs(remotes) do
                    if remote:IsA("RemoteEvent") and (remote.Name:lower():find("punch") or remote.Name:lower():find("attack")) then
                        local target = getClosestPlayer()
                        if target then
                            remote:FireServer(target.Parent)  -- Asume arg es target char
                        end
                    end
                end
            end
        end)
    end
end

-- =============================================
-- ESP (igual)
-- =============================================
local function createESP(plr)
    if espData[plr] or plr == player then return end
    espData[plr] = {}
    
    local function onCharAdded(char)
        local root = char:WaitForChild("HumanoidRootPart", 5)
        local head = char:WaitForChild("Head", 5)
        if not root or not head then return end
        
        local box = Instance.new("BoxHandleAdornment")
        box.Name = plr.Name .. "_Hitbox"
        box.Adornee = root
        box.Size = Vector3.new(6, 8, 6)
        box.Color3 = Color3.fromRGB(255, 0, 0)
        box.Transparency = 0.5
        box.AlwaysOnTop = true
        box.ZIndex = 10
        box.Parent = espFolder
        
        local bill = Instance.new("BillboardGui")
        bill.Name = plr.Name .. "_Name"
        bill.Adornee = head
        bill.Size = UDim2.new(0, 120 + #plr.Name * 4, 0, 45)
        bill.StudsOffset = Vector3.new(0, 3.5, 0)
        bill.Parent = espFolder
        bill.LightInfluence = 0
        bill.AlwaysOnTop = true
        
        local text = Instance.new("TextLabel")
        text.Size = UDim2.new(1, 0, 1, 0)
        text.BackgroundTransparency = 1
        text.Text = plr.Name
        text.TextColor3 = Color3.fromRGB(255, 255, 255)
        text.TextStrokeTransparency = 0
        text.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        text.Font = Enum.Font.GothamBold
        text.TextScaled = true
        text.Parent = bill
        
        espData[plr].box = box
        espData[plr].bill = bill
    end
    
    if plr.Character then task.spawn(onCharAdded, plr.Character) end
    plr.CharacterAdded:Connect(onCharAdded)
end

local function removeESP(plr)
    if espData[plr] then
        if espData[plr].box then espData[plr].box:Destroy() end
        if espData[plr].bill then espData[plr].bill:Destroy() end
        espData[plr] = nil
    end
end

local function toggleESP(enable)
    espEnabled = enable
    if enable then
        espFolder = Instance.new("Folder")
        espFolder.Name = "MuscleMastersESP"
        espFolder.Parent = workspace
        for _, plr in ipairs(Players:GetPlayers()) do createESP(plr) end
        Players.PlayerAdded:Connect(createESP)
    else
        for plr, _ in pairs(espData) do removeESP(plr) end
        espData = {}
        if espFolder then espFolder:Destroy() espFolder = nil end
    end
end

Players.PlayerRemoving:Connect(removeESP)

-- =============================================
-- TP ALL
-- =============================================
local function safeTeleportToAll()
    cleanupMovement()
    local myChar = player.Character
    if not myChar or not myChar:FindFirstChild("HumanoidRootPart") then 
        StarterGui:SetCore("SendNotification", {Title="Error", Text="Sin personaje", Duration=2})
        return 
    end
    local myRoot = myChar.HumanoidRootPart
    local count = 0
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
            local targetPos = plr.Character.HumanoidRootPart.Position + Vector3.new(math.random(-4,4), 6, math.random(-4,4))
            local dist = (targetPos - myRoot.Position).Magnitude
            if dist <= MAX_DIST then
                local duration = math.min(dist / 100, 0.6)
                local tween = TweenService:Create(myRoot, TweenInfo.new(duration, Enum.EasingStyle.Quad), {CFrame = CFrame.new(targetPos)})
                tween:Play()
                tween.Completed:Wait()
                count += 1
                task.wait(0.5 + math.random() * 0.3)
            end
        end
    end
    StarterGui:SetCore("SendNotification", {Title="TP ALL", Text=count.." jugadores! 🚀", Duration=3})
end

-- =============================================
-- GUI (igual con sliders)
-- =============================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MuscleMastersHub"
ScreenGui.Parent = player:WaitForChild("PlayerGui")
ScreenGui.ResetOnSpawn = false

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 280, 0, 380)
MainFrame.Position = UDim2.new(1, -300, 1, -400)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BackgroundTransparency = 0.15
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 16)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(100, 200, 255)
MainStroke.Thickness = 2.5
MainStroke.Transparency = 0.3
MainStroke.Parent = MainFrame

local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 35)
TitleBar.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
TitleBar.Parent = MainFrame

local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0, 16)
TitleCorner.Parent = TitleBar

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, 0, 1, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "💪 Brazzers ULTRA++"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextScaled = true
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.Parent = TitleBar

local btnSize = UDim2.new(0.3, -10, 0, 55)
local row1Y = UDim2.new(0, 0, 0.16, 0)
local row2Y = UDim2.new(0, 0, 0.48, 0)
local posL = UDim2.new(0.02, 0, 0, 0)
local posM = UDim2.new(0.34, 0, 0, 0)
local posR = UDim2.new(0.66, 0, 0, 0)

local FollowBtn = Instance.new("TextButton")
FollowBtn.Size = btnSize
FollowBtn.Position = posL + row1Y
FollowBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
FollowBtn.Text = "FOLLOW\nOFF"
FollowBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
FollowBtn.TextScaled = true
FollowBtn.Font = Enum.Font.GothamBold
FollowBtn.Parent = MainFrame
local FollowCorner = Instance.new("UICorner", FollowBtn) FollowCorner.CornerRadius = UDim.new(0, 12)

local FlyBtn = Instance.new("TextButton")
FlyBtn.Size = btnSize
FlyBtn.Position = posM + row1Y
FlyBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 50)
FlyBtn.Text = "FLY\nOFF"
FlyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
FlyBtn.TextScaled = true
FlyBtn.Font = Enum.Font.GothamBold
FlyBtn.Parent = MainFrame
local FlyCorner = Instance.new("UICorner", FlyBtn) FlyCorner.CornerRadius = UDim.new(0, 12)

local EspBtn = Instance.new("TextButton")
EspBtn.Size = btnSize
EspBtn.Position = posR + row1Y
EspBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 200)
EspBtn.Text = "ESP\nOFF"
EspBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
EspBtn.TextScaled = true
EspBtn.Font = Enum.Font.GothamBold
EspBtn.Parent = MainFrame
local EspCorner = Instance.new("UICorner", EspBtn) EspCorner.CornerRadius = UDim.new(0, 12)

local PunchBtn = Instance.new("TextButton")
PunchBtn.Size = btnSize
PunchBtn.Position = posL + row2Y
PunchBtn.BackgroundColor3 = Color3.fromRGB(255, 150, 50)
PunchBtn.Text = "PUNCH\nOFF"
PunchBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
PunchBtn.TextScaled = true
PunchBtn.Font = Enum.Font.GothamBold
PunchBtn.Parent = MainFrame
local PunchCorner = Instance.new("UICorner", PunchBtn) PunchCorner.CornerRadius = UDim.new(0, 12)

local TeleBtn = Instance.new("TextButton")
TeleBtn.Size = btnSize
TeleBtn.Position = posM + row2Y
TeleBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 255)
TeleBtn.Text = "TP\nALL"
TeleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
TeleBtn.TextScaled = true
TeleBtn.Font = Enum.Font.GothamBold
TeleBtn.Parent = MainFrame
local TeleCorner = Instance.new("UICorner", TeleBtn) TeleCorner.CornerRadius = UDim.new(0, 12)

local ChaseLabel = Instance.new("TextLabel")
ChaseLabel.Size = UDim2.new(0.28, 0, 0, 30)
ChaseLabel.Position = UDim2.new(0.02, 0, 0.72, 0)
ChaseLabel.BackgroundTransparency = 1
ChaseLabel.Text = "Chase Spd:"
ChaseLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
ChaseLabel.TextScaled = true
ChaseLabel.Font = Enum.Font.Gotham
ChaseLabel.TextXAlignment = Enum.TextXAlignment.Left
ChaseLabel.Parent = MainFrame

local ChaseBox = Instance.new("TextBox")
ChaseBox.Size = UDim2.new(0, 70, 0, 35)
ChaseBox.Position = UDim2.new(0.31, 0, 0.70, 0)
ChaseBox.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
ChaseBox.Text = "80"
ChaseBox.TextColor3 = Color3.fromRGB(255, 255, 255)
ChaseBox.TextScaled = true
ChaseBox.Font = Enum.Font.Gotham
ChaseBox.Parent = MainFrame
local ChaseBoxCorner = Instance.new("UICorner", ChaseBox) ChaseBoxCorner.CornerRadius = UDim.new(0, 8)

local FlyLabel = Instance.new("TextLabel")
FlyLabel.Size = UDim2.new(0.28, 0, 0, 30)
FlyLabel.Position = UDim2.new(0.02, 0, 0.82, 0)
FlyLabel.BackgroundTransparency = 1
FlyLabel.Text = "Fly Spd:"
FlyLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
FlyLabel.TextScaled = true
FlyLabel.Font = Enum.Font.Gotham
FlyLabel.TextXAlignment = Enum.TextXAlignment.Left
FlyLabel.Parent = MainFrame

local FlyBox = Instance.new("TextBox")
FlyBox.Size = UDim2.new(0, 70, 0, 35)
FlyBox.Position = UDim2.new(0.31, 0, 0.80, 0)
FlyBox.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
FlyBox.Text = "50"
FlyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
FlyBox.TextScaled = true
FlyBox.Font = Enum.Font.Gotham
FlyBox.Parent = MainFrame
local FlyBoxCorner = Instance.new("UICorner", FlyBox) FlyBoxCorner.CornerRadius = UDim.new(0, 8)

local ApplyChaseBtn = Instance.new("TextButton")
ApplyChaseBtn.Size = UDim2.new(0, 60, 0, 35)
ApplyChaseBtn.Position = UDim2.new(0.42, 0, 0.70, 0)
ApplyChaseBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
ApplyChaseBtn.Text = "Set"
ApplyChaseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ApplyChaseBtn.TextScaled = true
ApplyChaseBtn.Font = Enum.Font.GothamBold
ApplyChaseBtn.Parent = MainFrame
local ApplyChaseCorner = Instance.new("UICorner", ApplyChaseBtn) ApplyChaseCorner.CornerRadius = UDim.new(0, 8)

local ApplyFlyBtn = Instance.new("TextButton")
ApplyFlyBtn.Size = UDim2.new(0, 60, 0, 35)
ApplyFlyBtn.Position = UDim2.new(0.42, 0, 0.80, 0)
ApplyFlyBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
ApplyFlyBtn.Text = "Set"
ApplyFlyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ApplyFlyBtn.TextScaled = true
ApplyFlyBtn.Font = Enum.Font.GothamBold
ApplyFlyBtn.Parent = MainFrame
local ApplyFlyCorner = Instance.new("UICorner", ApplyFlyBtn) ApplyFlyCorner.CornerRadius = UDim.new(0, 8)

-- =============================================
-- Botones
-- =============================================
FollowBtn.Activated:Connect(function()
    chaseEnabled = not chaseEnabled
    FollowBtn.BackgroundColor3 = chaseEnabled and Color3.fromRGB(50, 200, 50) or Color3.fromRGB(200, 50, 50)
    FollowBtn.Text = chaseEnabled and "FOLLOW\nON" or "FOLLOW\nOFF"
    StarterGui:SetCore("SendNotification", {Title="FOLLOW", Text=chaseEnabled and "🟢 Sigue vivo!" or "🔴 Off", Duration=2})
end)

FlyBtn.Activated:Connect(function()
    flyEnabled = not flyEnabled
    FlyBtn.BackgroundColor3 = flyEnabled and Color3.fromRGB(50, 200, 150) or Color3.fromRGB(200, 150, 50)
    FlyBtn.Text = flyEnabled and "FLY\nON" or "FLY\nOFF"
    if not flyEnabled then
        cleanupMovement()  -- Fuerza freefall no freeze
    end
    StarterGui:SetCore("SendNotification", {Title="FLY", Text=flyEnabled and "🟢 Swim en aire!" or "🔴 Off (cae)", Duration=2})
end)

EspBtn.Activated:Connect(function()
    toggleESP(not espEnabled)
    EspBtn.BackgroundColor3 = espEnabled and Color3.fromRGB(100, 50, 200) or Color3.fromRGB(150, 50, 200)
    EspBtn.Text = espEnabled and "ESP\nON" or "ESP\nOFF"
    StarterGui:SetCore("SendNotification", {Title="ESP", Text=espEnabled and "👁️ On" or "Off", Duration=2})
end)

PunchBtn.Activated:Connect(function()
    togglePunch(not punchEnabled)
    PunchBtn.BackgroundColor3 = punchEnabled and Color3.fromRGB(255, 100, 0) or Color3.fromRGB(255, 150, 50)
    PunchBtn.Text = punchEnabled and "PUNCH\nON" or "PUNCH\nOFF"
    StarterGui:SetCore("SendNotification", {Title="PUNCH", Text=punchEnabled and "🔥 Legends style OP!" or "Off", Duration=2})
end)

TeleBtn.Activated:Connect(function()
    task.spawn(safeTeleportToAll)
end)

ApplyChaseBtn.Activated:Connect(function()
    local spd = tonumber(ChaseBox.Text)
    if spd then
        chaseSpeed = math.max(16, math.min(300, spd))
        ChaseBox.Text = tostring(chaseSpeed)
        StarterGui:SetCore("SendNotification", {Title="Chase Speed", Text=chaseSpeed, Duration=2})
    end
end)

ApplyFlyBtn.Activated:Connect(function()
    local spd = tonumber(FlyBox.Text)
    if spd then
        flySpeed = math.max(16, math.min(300, spd))
        FlyBox.Text = tostring(flySpeed)
        StarterGui:SetCore("SendNotification", {Title="Fly Speed", Text=flySpeed, Duration=2})
    end
end)

-- =============================================
-- Draggable
-- =============================================
local dragging = false
local dragStart = nil
local startPos = nil

TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- Respawn cleanup
player.CharacterAdded:Connect(function()
    task.wait(2)
    cleanupMovement()
    punchEnabled = false
    PunchBtn.BackgroundColor3 = Color3.fromRGB(255, 150, 50)
    PunchBtn.Text = "PUNCH\nOFF"
end)

-- Listo! 🔥📱
StarterGui:SetCore("SendNotification", {
    Title = "💪 BRAZZERS++", 
    Text = "PUNCH Legends fix + FLY swim no freeze! Prueba 💥", 
    Duration = 10
})
print("Hub cargado | Punch: Legends tool/remote | Fly: Swim anim + freefall off")
