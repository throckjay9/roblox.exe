-- Free Fire Max Roblox Aimbot + ESP + GUI
-- Features: Antenna, ESP Line, ESP Box
-- GUI toggled by pressing F and draggable
-- Use responsibly, ensure you have permissions for testing

-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

-- Configuration
local ESPConfig = {
    Antenna = true,
    ESPLines = true,
    ESPBoxes = true,
}
local AimbotEnabled = false  -- Default off, toggle via GUI
local AimbotFOV = 80 -- degrees
local AimPartName = "Head" -- Aim at head only

-- Check if GUI already exists to prevent duplication
local existingGui = game.CoreGui:FindFirstChild("FreeFireMaxAimbotGUI")
if existingGui then
    existingGui:Destroy()
end

-- GUI Setup
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FreeFireMaxAimbotGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 1000
ScreenGui.Parent = game.CoreGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 280, 0, 170)
MainFrame.Position = UDim2.new(0.5, -140, 0.5, -85)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui
MainFrame.Active = true
MainFrame.Draggable = true

-- Title
local Title = Instance.new("TextLabel")
Title.Name = "Title"
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
Title.BorderSizePixel = 0
Title.Text = "Free Fire Max Aimbot & ESP"
Title.TextColor3 = Color3.fromRGB(200, 200, 200)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 18
Title.Parent = MainFrame

-- Checkbox creation function
local function CreateCheckbox(name, position, default)
    local CheckFrame = Instance.new("Frame")
    CheckFrame.Name = name .. "Frame"
    CheckFrame.Size = UDim2.new(1, -20, 0, 30)
    CheckFrame.Position = position
    CheckFrame.BackgroundTransparency = 1
    CheckFrame.Parent = MainFrame

    local CheckBox = Instance.new("TextButton")
    CheckBox.Name = name .. "Check"
    CheckBox.Size = UDim2.new(0, 20, 0, 20)
    CheckBox.Position = UDim2.new(0, 0, 0.5, -10)
    CheckBox.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    CheckBox.BorderSizePixel = 0
    CheckBox.Text = ""
    CheckBox.AutoButtonColor = false
    CheckBox.Parent = CheckFrame

    local CheckMark = Instance.new("ImageLabel")
    CheckMark.Name = "CheckMark"
    CheckMark.Size = UDim2.new(1, -4, 1, -4)
    CheckMark.Position = UDim2.new(0, 2, 0, 2)
    CheckMark.BackgroundTransparency = 1
    CheckMark.Image = "rbxassetid://8549149078" -- checkmark image from Roblox assets
    CheckMark.Visible = default
    CheckMark.Parent = CheckBox

    local Label = Instance.new("TextLabel")
    Label.Name = name .. "Label"
    Label.Size = UDim2.new(1, -30, 1, 0)
    Label.Position = UDim2.new(0, 30, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = name
    Label.TextColor3 = Color3.fromRGB(210, 210, 210)
    Label.Font = Enum.Font.Gotham
    Label.TextSize = 16
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Parent = CheckFrame

    local checked = default

    CheckBox.MouseButton1Click:Connect(function()
        checked = not checked
        CheckMark.Visible = checked

        if name == "ESP Antenna" then
            ESPConfig.Antenna = checked
        elseif name == "ESP Line" then
            ESPConfig.ESPLines = checked
        elseif name == "ESP Box" then
            ESPConfig.ESPBoxes = checked
        elseif name == "Aimbot" then
            AimbotEnabled = checked
        end
    end)

    return CheckFrame
end

-- Create checkboxes
local antennaCheckbox = CreateCheckbox("ESP Antenna", UDim2.new(0, 10, 0, 40), true)
local espLineCheckbox = CreateCheckbox("ESP Line", UDim2.new(0, 10, 0, 70), true)
local espBoxCheckbox = CreateCheckbox("ESP Box", UDim2.new(0, 10, 0, 100), true)
local aimbotCheckbox = CreateCheckbox("Aimbot", UDim2.new(0, 10, 0, 130), false)

-- GUI Toggle
local guiEnabled = false
ScreenGui.Enabled = guiEnabled

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.F then
        guiEnabled = not guiEnabled
        ScreenGui.Enabled = guiEnabled
    end
end)

-- Drawing helper functions and objects
local drawingNew = Drawing and Drawing.new or nil
local ESPObjects = {}

-- Creates or retrieves existing ESP drawings for a given player
local function CreateESPForPlayer(player)
    local esp = ESPObjects[player]
    if esp then
        return esp
    end

    esp = {}

    -- Box
    esp.Box = drawingNew and drawingNew("Square")
    if esp.Box then
        esp.Box.Visible = false
        esp.Box.Thickness = 2
        esp.Box.Color = Color3.new(1, 1, 1)
        esp.Box.Filled = false
    end

    -- Line
    esp.Line = drawingNew and drawingNew("Line")
    if esp.Line then
        esp.Line.Visible = false
        esp.Line.Thickness = 1.5
        esp.Line.Color = Color3.new(1, 1, 1)
    end

    -- Antenna
    esp.Antenna = drawingNew and drawingNew("Line")
    if esp.Antenna then
        esp.Antenna.Visible = false
        esp.Antenna.Thickness = 1.5
        esp.Antenna.Color = Color3.new(1, 1, 1)
    end

    ESPObjects[player] = esp
    return esp
end

-- Remove ESP when player leaves
Players.PlayerRemoving:Connect(function(player)
    local esp = ESPObjects[player]
    if esp then
        if esp.Box then esp.Box.Visible = false; esp.Box:Remove() end
        if esp.Line then esp.Line.Visible = false; esp.Line:Remove() end
        if esp.Antenna then esp.Antenna.Visible = false; esp.Antenna:Remove() end
        ESPObjects[player] = nil
    end
end)

-- Get closest target to crosshair within FOV
local function GetClosestTarget()
    local closestPlayer = nil
    local closestDistance = AimbotFOV

    local camera = workspace.CurrentCamera
    local centerX, centerY = camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(AimPartName) and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health > 0 then
            local part = player.Character[AimPartName]
            local pos, onScreen = camera:WorldToViewportPoint(part.Position)
            if onScreen then
                local dx = pos.X - centerX
                local dy = pos.Y - centerY
                local distanceFromCenter = math.sqrt(dx * dx + dy * dy)
                if distanceFromCenter < closestDistance then
                    closestDistance = distanceFromCenter
                    closestPlayer = player
                end
            end
        end
    end
    return closestPlayer
end

-- Aimbot aiming function focusing on head
local function AimAt(target)
    if not target or not target.Character or not target.Character:FindFirstChild(AimPartName) then return end
    local camera = workspace.CurrentCamera
    local part = target.Character[AimPartName]

    local pos, onScreen = camera:WorldToViewportPoint(part.Position)
    if not onScreen then return end

    -- Smooth aim movement with lerp
    local mouseLocation = Vector2.new(UserInputService:GetMouseLocation().X, UserInputService:GetMouseLocation().Y)
    local targetPos = Vector2.new(pos.X, pos.Y)
    local delta = targetPos - mouseLocation
    local smoothFactor = 0.3 -- smaller is faster

    local newMousePos = mouseLocation + delta * smoothFactor
    -- Move camera towards newMousePos
    local cameraCF = camera.CFrame
    local ray = camera:ScreenPointToRay(newMousePos.X, newMousePos.Y)
    local direction = ray.Direction
    local newCFrame = CFrame.new(cameraCF.Position, cameraCF.Position + direction)
    camera.CFrame = newCFrame
end

-- ESP Update function
local function UpdateESP()
    local camera = workspace.CurrentCamera
    local centerX, centerY = camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health > 0 then
            local character = player.Character
            local esp = CreateESPForPlayer(player)

            local head = character:FindFirstChild("Head")
            if not head then
                if esp.Box then esp.Box.Visible = false end
                if esp.Line then esp.Line.Visible = false end
                if esp.Antenna then esp.Antenna.Visible = false end
                continue
            end
            local rootPart = character:FindFirstChild("HumanoidRootPart")
            if not rootPart then
                if esp.Box then esp.Box.Visible = false end
                if esp.Line then esp.Line.Visible = false end
                if esp.Antenna then esp.Antenna.Visible = false end
                continue
            end

            local headPos = head.Position
            local rootPos = rootPart.Position

            local headPos2D, headOnScreen = camera:WorldToViewportPoint(headPos)
            local rootPos2D, rootOnScreen = camera:WorldToViewportPoint(rootPos)

            if ESPConfig.ESPBoxes and esp.Box and headOnScreen and rootOnScreen then
                local boxHeight = math.abs(headPos2D.Y - rootPos2D.Y)
                local boxWidth = boxHeight / 2
                esp.Box.Size = boxHeight
                esp.Box.Position = Vector2.new(headPos2D.X - boxWidth / 2, headPos2D.Y)
                esp.Box.Color = Color3.new(1, 0, 0)
                esp.Box.Visible = true
            else
                if esp.Box then esp.Box.Visible = false end
            end

            if ESPConfig.ESPLines and esp.Line and headOnScreen then
                esp.Line.From = Vector2.new(centerX, centerY)
                esp.Line.To = Vector2.new(headPos2D.X, headPos2D.Y)
                esp.Line.Color = Color3.new(1, 1, 0)
                esp.Line.Visible = true
            else
                if esp.Line then esp.Line.Visible = false end
            end

            if ESPConfig.Antenna and esp.Antenna and headOnScreen then
                local antennaTopPos = headPos + Vector3.new(0, 5, 0)
                local antennaTop2D, topOnScreen = camera:WorldToViewportPoint(antennaTopPos)
                if topOnScreen then
                    esp.Antenna.From = Vector2.new(headPos2D.X, headPos2D.Y)
                    esp.Antenna.To = Vector2.new(antennaTop2D.X, antennaTop2D.Y)
                    esp.Antenna.Color = Color3.new(0, 1, 1)
                    esp.Antenna.Visible = true
                else
                    esp.Antenna.Visible = false
                end
            else
                if esp.Antenna then esp.Antenna.Visible = false end
            end
        else
            local esp = ESPObjects[player]
            if esp then
                if esp.Box then esp.Box.Visible = false end
                if esp.Line then esp.Line.Visible = false end
                if esp.Antenna then esp.Antenna.Visible = false end
            end
        end
    end
end

-- Main Loop
RunService.RenderStepped:Connect(function()
    UpdateESP()
    if AimbotEnabled then
        local target = GetClosestTarget()
        if target then
            AimAt(target)
        end
    end
end)

-- Cleanup on script unload
local function Cleanup()
    ScreenGui:Destroy()
    for _, esp in pairs(ESPObjects) do
        if esp.Box then esp.Box.Visible = false; esp.Box:Remove() end
        if esp.Line then esp.Line.Visible = false; esp.Line:Remove() end
        if esp.Antenna then esp.Antenna.Visible = false; esp.Antenna:Remove() end
    end
    ESPObjects = {}
end

print("Free Fire Max Aimbot & ESP script loaded. Press F to toggle GUI.")
