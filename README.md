-- gerasHUB v2.8 - ESP Simples 500m (sem esqueleto) + Aimbot Botão GUI
-- Right Shift abre | Botão Aimbot liga/desliga | LMB trava | Solta ao virar câmera

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local mouse = player:GetMouse()
local camera = workspace.CurrentCamera

-- Configs
local ESPToggle = false
local AimActive = false
local LockedTarget = nil
local ESPHighlights = {}  -- Armazena os Highlights
local AIM_SMOOTH = 0.35
local RELEASE_THRESHOLD = 0.65
local PREDICT_STRENGTH = 0.7
local ESP_MAX_DISTANCE = 500  -- 500 studs

-- PlayerGui setup
local pg = player:WaitForChild("PlayerGui", 10)
if not pg then return end
for _, gui in pairs(pg:GetChildren()) do
    if gui:IsA("ScreenGui") and gui.Name:find("gerasHUB") then gui:Destroy() end
end

local sg = Instance.new("ScreenGui")
sg.Name = "gerasHUB_v28"
sg.ResetOnSpawn = false
sg.Parent = pg

local frame = Instance.new("Frame")
frame.Parent = sg
frame.Size = UDim2.new(0, 280, 0, 180)
frame.Position = UDim2.new(0.5, -140, 0.5, -90)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
frame.BorderSizePixel = 0
frame.Visible = false

local title = Instance.new("TextLabel")
title.Parent = frame
title.Size = UDim2.new(1, 0, 0, 35)
title.BackgroundTransparency = 1
title.Text = "gerasHUB v2.8 - ESP 500m"
title.TextColor3 = Color3.fromRGB(200, 220, 255)
title.TextScaled = true
title.Font = Enum.Font.GothamBold

local espBtn = Instance.new("TextButton")
espBtn.Parent = frame
espBtn.Size = UDim2.new(0.9, 0, 0, 40)
espBtn.Position = UDim2.new(0.05, 0, 0.22, 0)
espBtn.BackgroundColor3 = Color3.fromRGB(180, 60, 60)
espBtn.Text = "ESP: OFF"
espBtn.TextColor3 = Color3.fromRGB(255,255,255)
espBtn.TextScaled = true
espBtn.Font = Enum.Font.GothamSemibold

local aimBtn = Instance.new("TextButton")
aimBtn.Parent = frame
aimBtn.Size = UDim2.new(0.9, 0, 0, 40)
aimBtn.Position = UDim2.new(0.05, 0, 0.52, 0)
aimBtn.BackgroundColor3 = Color3.fromRGB(180, 60, 60)
aimBtn.Text = "Aimbot: OFF\n(Clique aqui para ligar)"
aimBtn.TextColor3 = Color3.fromRGB(255,255,255)
aimBtn.TextScaled = true
aimBtn.Font = Enum.Font.GothamSemibold

local unlockBtn = Instance.new("TextButton")
unlockBtn.Parent = frame
unlockBtn.Size = UDim2.new(0.9, 0, 0, 35)
unlockBtn.Position = UDim2.new(0.05, 0, 0.80, 0)
unlockBtn.BackgroundColor3 = Color3.fromRGB(100, 150, 255)
unlockBtn.Text = "🔓 Unlock Movimento"
unlockBtn.TextColor3 = Color3.fromRGB(255,255,255)
unlockBtn.TextScaled = true
unlockBtn.Font = Enum.Font.GothamSemibold

-- Cantos
for _, obj in ipairs({frame, espBtn, aimBtn, unlockBtn}) do
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = obj
end

-- Drag
local dragging, dragStart, startPos = false, nil, nil
title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = frame.Position
    end
end)
title.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- Toggle GUI
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.RightShift then
        frame.Visible = not frame.Visible
    end
end)

espBtn.MouseButton1Click:Connect(function()
    ESPToggle = not ESPToggle
    espBtn.Text = "ESP: " .. (ESPToggle and "ON" or "OFF")
    espBtn.BackgroundColor3 = ESPToggle and Color3.fromRGB(60,180,60) or Color3.fromRGB(180,60,60)
end)

aimBtn.MouseButton1Click:Connect(function()
    AimActive = not AimActive
    aimBtn.Text = AimActive and "Aimbot: ON\n(Clique aqui para desligar)" or "Aimbot: OFF\n(Clique aqui para ligar)"
    aimBtn.BackgroundColor3 = AimActive and Color3.fromRGB(60,180,60) or Color3.fromRGB(180,60,60)
    if not AimActive then LockedTarget = nil end
end)

unlockBtn.MouseButton1Click:Connect(function()
    pcall(function()
        camera.CameraType = Enum.CameraType.Custom
        if player.Character and player.Character:FindFirstChild("Humanoid") then
            player.Character.Humanoid.PlatformStand = false
        end
    end)
    LockedTarget = nil
end)

-- ESP simples com Highlight (500m, visível pelas paredes)
RunService.Heartbeat:Connect(function()
    if not ESPToggle then
        for _, hl in pairs(ESPHighlights) do
            pcall(function() hl:Destroy() end)
        end
        ESPHighlights = {}
        return
    end

    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end
    local myPos = player.Character.HumanoidRootPart.Position

    -- Limpa distantes
    for i = #ESPHighlights, 1, -1 do
        local hl = ESPHighlights[i]
        if hl and hl.Adornee then
            local char = hl.Adornee
            if char and char:FindFirstChild("HumanoidRootPart") then
                local dist = (myPos - char.HumanoidRootPart.Position).Magnitude
                if dist > ESP_MAX_DISTANCE then
                    hl:Destroy()
                    table.remove(ESPHighlights, i)
                end
            else
                hl:Destroy()
                table.remove(ESPHighlights, i)
            end
        end
    end

    -- Adiciona novos
    for _, other in pairs(Players:GetPlayers()) do
        if other == player or not other.Character or not other.Character:FindFirstChild("HumanoidRootPart") then continue end

        local char = other.Character
        local root = char.HumanoidRootPart
        local dist = (myPos - root.Position).Magnitude

        if dist <= ESP_MAX_DISTANCE then
            local alreadyHas = false
            for _, hl in pairs(ESPHighlights) do
                if hl.Adornee == char then
                    alreadyHas = true
                    break
                end
            end

            if not alreadyHas then
                local hl = Instance.new("Highlight")
                hl.Name = "ESPHighlight"
                hl.Adornee = char
                hl.FillColor = Color3.fromRGB(255, 80, 80)
                hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                hl.FillTransparency = 0.5
                hl.OutlineTransparency = 0
                hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop  -- Visível pelas paredes
                hl.Parent = char
                table.insert(ESPHighlights, hl)
            end
        end
    end
end)

-- Target part
local function getBestTargetPart(model)
    return model:FindFirstChild("Head") or model:FindFirstChild("UpperTorso") or model:FindFirstChild("Torso") or model:FindFirstChild("HumanoidRootPart")
end

-- LMB Lock
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe or not AimActive then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local tgt = mouse.Target
        if tgt then
            local model = tgt:FindFirstAncestorWhichIsA("Model")
            if model and model ~= player.Character and model:FindFirstChild("Humanoid") and model.Humanoid.Health > 0 then
                LockedTarget = model
                print("🔒 LOCK: " .. model.Name)
            end
        end
    end
end)

-- Aimbot
RunService.Heartbeat:Connect(function()
    if not AimActive or not LockedTarget then return end
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end

    local aimPart = getBestTargetPart(LockedTarget)
    if not aimPart or not LockedTarget:FindFirstChild("Humanoid") or LockedTarget.Humanoid.Health <= 0 then
        LockedTarget = nil
        return
    end

    local targetPos = aimPart.Position
    if LockedTarget:FindFirstChild("HumanoidRootPart") then
        local vel = LockedTarget.HumanoidRootPart.Velocity
        targetPos = targetPos + vel * 0.1 * PREDICT_STRENGTH
    end

    local camPos = camera.CFrame.Position
    local targetDir = (targetPos - camPos).Unit
    local currentLook = camera.CFrame.LookVector

    if currentLook:Dot(targetDir) < RELEASE_THRESHOLD then
        LockedTarget = nil
        print("🔓 Lock solto: câmera movida pro lado")
        return
    end

    local goal = CFrame.lookAt(camPos, targetPos)
    camera.CFrame = camera.CFrame:Lerp(goal, AIM_SMOOTH)
end)

player.CharacterAdded:Connect(function(newChar)
    task.wait(1)
    LockedTarget = nil
end)

print("gerasHUB v2.8 carregado! • ESP simples (Highlight vermelho) até 500m • Visível pelas paredes • Ative no botão ESP")
