--[[
    Auto Parry + Discord Logger (iPad & Mobile Compatible - Undetectable GUI)
    Updated & Anti-Detection Protected
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local UserInputService = game:GetService("UserInputService")
local Stats = game:GetService("Stats")
local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer

-- Custom Executor HTTP function resolver
local http_request = (syn and syn.request) or (http and http.request) or http_request or request or (fluxus and fluxus.request)

-- Function to safely parent UI away from anti-cheat detection
local function GetSafeParent()
    if gethui then
        return gethui()
    elseif syn and syn.protect_gui then
        local sg = Instance.new("ScreenGui")
        syn.protect_gui(sg)
        sg.Parent = LocalPlayer:WaitForChild("PlayerGui")
        return sg
    else
        return LocalPlayer:WaitForChild("PlayerGui")
    end
end

-- Random string generator for UI instance obfuscation
local function RandomName()
    local str = ""
    for _ = 1, 16 do
        str = str .. string.char(math.random(97, 122))
    end
    return str
end

-- ======================
-- CONFIG
-- ======================
local Config = {
    -- Distance & Clash Parameters
    BaseReactionWindow = 0.18,   -- Base trigger window (seconds)
    PingScalar = 0.85,           -- Ping adjustment multiplier
    CloseClashDistance = 25,     -- Distance to enter instant-clash mode
    FarDistanceThreshold = 100,  -- Distance threshold for long-range scaling
    
    -- Speed Scaling
    SpeedScalingFactor = 0.0008,
    
    -- Lockout (Cooldown between parries)
    MinLockout = 0.005,          -- Ultra-low lockout for fast close clashes
    MaxLockout = 0.2,

    -- Discord
    WebhookURL = "https://discord.com/api/webhooks/1550682285664239738/d6iIio921QjeDsuXieisBVb1WfYLDJ3m7IDr75dod5hOeQdddI6OtvvfxkVPrmKx1KYU",
    LogEnabled = true,
    LogParries = true,
    LogToggles = true,
    LogDeaths = true,
}

-- Custom Colors for Mobile UI Button
local UIColors = {
    OffBackground = Color3.fromRGB(35, 35, 45),
    OffText = Color3.fromRGB(200, 200, 210),
    OnBackground = Color3.fromRGB(120, 45, 220),
    OnText = Color3.fromRGB(255, 255, 255),
    Border = Color3.fromRGB(0, 225, 255)
}

-- ======================
-- STATE
-- ======================
local AutoParryEnabled = false
local IsParried = false
local TargetConnection = nil
local LastLogTime = 0
local ParryCount = 0
local SessionStart = os.time()
local ToggleBtnRef = nil

-- ======================
-- UTILS
-- ======================
local function Notify(title, text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title,
            Text = text,
            Duration = 2.2
        })
    end)
end

local function GetPing()
    local success, ping = pcall(function()
        local item = Stats.Network.ServerStatsItem:FindFirstChild("Data Ping")
        if item then
            return item:GetValue() / 1000
        end
        return LocalPlayer:GetNetworkPing()
    end)
    return success and ping or 0.04
end

local function GetBall()
    local ballsFolder = workspace:FindFirstChild("Balls")
    if not ballsFolder then return nil end

    for _, ball in ipairs(ballsFolder:GetChildren()) do
        if ball:GetAttribute("realBall") or ball:FindFirstChild("zoomies") then
            return ball
        end
    end
    return nil
end

-- ======================
-- DISCORD LOGGER
-- ======================
local function SendDiscordLog(title, description, color, fields)
    if not Config.LogEnabled or Config.WebhookURL == "" or Config.WebhookURL == "YOUR_WEBHOOK_URL_HERE" then
        return
    end

    local now = os.clock()
    if now - LastLogTime < 0.5 then return end
    LastLogTime = now

    local embed = {
        title = title,
        description = description,
        color = color or 5793266,
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        footer = {
            text = "Auto Parry Logger • " .. LocalPlayer.Name
        },
        fields = fields or {}
    }

    local payload = HttpService:JSONEncode({
        username = "Blade Ball Logger",
        embeds = { embed }
    })

    task.spawn(function()
        pcall(function()
            if http_request then
                http_request({
                    Url = Config.WebhookURL,
                    Method = "POST",
                    Headers = {
                        ["Content-Type"] = "application/json"
                    },
                    Body = payload
                })
            else
                HttpService:PostAsync(
                    Config.WebhookURL,
                    payload,
                    Enum.HttpContentType.ApplicationJson
                )
            end
        end)
    end)
end

local function LogParry(distance, speed, timeToReach, method)
    if not Config.LogParries then return end
    ParryCount += 1

    SendDiscordLog(
        "✅ Parry Fired",
        "Successfully triggered a parry",
        5763719,
        {
            { name = "Distance", value = string.format("%.1f studs", distance), inline = true },
            { name = "Speed", value = string.format("%.1f", speed), inline = true },
            { name = "Time to Hit", value = string.format("%.3fs", timeToReach), inline = true },
            { name = "Method", value = method or "Unknown", inline = true },
            { name = "Total Parries", value = tostring(ParryCount), inline = true },
            { name = "Ping", value = string.format("%.0fms", GetPing() * 1000), inline = true },
        }
    )
end

local function LogToggle(state)
    if not Config.LogToggles then return end
    SendDiscordLog(
        state and "🟢 Auto Parry Enabled" or "🔴 Auto Parry Disabled",
        state and "Script is now active" or "Script has been turned off",
        state and 5763719 or 15548997
    )
end

local function LogDeath()
    if not Config.LogDeaths then return end
    SendDiscordLog(
        "💀 Player Died",
        "Character died / reset",
        15158332,
        {
            { name = "Parries this life", value = tostring(ParryCount), inline = true },
            { name = "Session Time", value = string.format("%ds", os.time() - SessionStart), inline = true },
        }
    )
    ParryCount = 0
end

-- ======================
-- PARRY INPUT
-- ======================
local function FireParryInput()
    local method = "None"

    if typeof(mouse1click) == "function" then
        task.spawn(mouse1click)
        method = "mouse1click"
    end

    if keypress and keyrelease then
        task.spawn(function()
            keypress(0x46)
            task.wait(0.005)
            keyrelease(0x46)
        end)
        method = method == "None" and "keypress F" or method .. " + keypress F"
    else
        pcall(function()
            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.F, false, game)
            task.wait(0.005)
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.F, false, game)
        end)
        method = method == "None" and "VirtualKey F" or method .. " + VirtualKey F"
    end

    local remote = ReplicatedStorage:FindFirstChild("ParryAttempt", true)
        or (ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("ParryButtonPress"))

    if remote and remote:IsA("RemoteEvent") then
        remote:FireServer()
        method = method == "None" and "Remote" or method .. " + Remote"
    end

    return method
end

-- ======================
-- TARGET LISTENER
-- ======================
local function ResetTargetListener()
    if TargetConnection then
        TargetConnection:Disconnect()
        TargetConnection = nil
    end

    local ball = GetBall()
    if ball then
        TargetConnection = ball:GetAttributeChangedSignal("target"):Connect(function()
            IsParried = false
        end)
    end
end

-- ======================
-- CHARACTER HANDLING
-- ======================
local function OnCharacterAdded(char)
    IsParried = false
    local humanoid = char:WaitForChild("Humanoid", 8)
    if humanoid then
        humanoid.Died:Connect(function()
            LogDeath()
        end)
    end
end

if LocalPlayer.Character then
    OnCharacterAdded(LocalPlayer.Character)
end
LocalPlayer.CharacterAdded:Connect(OnCharacterAdded)

-- ======================
-- TOGGLE & SAFE MOBILE UI
-- ======================
local function UpdateButtonVisuals(btn)
    if not btn then return end
    if AutoParryEnabled then
        btn.BackgroundColor3 = UIColors.OnBackground
        btn.TextColor3 = UIColors.OnText
        btn.Text = "Parry: ON"
    else
        btn.BackgroundColor3 = UIColors.OffBackground
        btn.TextColor3 = UIColors.OffText
        btn.Text = "Parry: OFF"
    end
end

local function SetAutoParry(state)
    AutoParryEnabled = state
    Notify("Auto Parry", AutoParryEnabled and "ENABLED" or "DISABLED")
    LogToggle(AutoParryEnabled)

    if ToggleBtnRef then
        UpdateButtonVisuals(ToggleBtnRef)
    end
end

local function CreateMobileUI()
    local container = GetSafeParent()
    
    local ScreenGui
    if container:IsA("ScreenGui") then
        ScreenGui = container
    else
        ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name = RandomName()
        ScreenGui.ResetOnSpawn = false
        ScreenGui.Parent = container
    end

    local ToggleBtn = Instance.new("TextButton")
    ToggleBtn.Name = RandomName()
    ToggleBtn.Size = UDim2.new(0, 110, 0, 45)
    ToggleBtn.Position = UDim2.new(0.02, 0, 0.4, 0)
    ToggleBtn.TextScaled = true
    ToggleBtn.Font = Enum.Font.SourceSansBold
    ToggleBtn.Parent = ScreenGui
    ToggleBtnRef = ToggleBtn

    local UICorner = Instance.new("UICorner")
    UICorner.Name = RandomName()
    UICorner.CornerRadius = UDim.new(0, 10)
    UICorner.Parent = ToggleBtn

    local UIStroke = Instance.new("UIStroke")
    UIStroke.Name = RandomName()
    UIStroke.Color = UIColors.Border
    UIStroke.Thickness = 2
    UIStroke.Parent = ToggleBtn

    UpdateButtonVisuals(ToggleBtn)

    ToggleBtn.MouseButton1Click:Connect(function()
        SetAutoParry(not AutoParryEnabled)
    end)

    -- Touch/Mouse dragging
    local dragging, dragInput, dragStart, startPos
    ToggleBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = ToggleBtn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    ToggleBtn.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            ToggleBtn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- Create Obfuscated UI
CreateMobileUI()

-- Keycode Listener (For Hardware Keyboard Users)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.K then
        SetAutoParry(not AutoParryEnabled)
    end
end)

-- ======================
-- BALL FOLDER LISTENER
-- ======================
local ballsFolder = workspace:WaitForChild("Balls", 8)
if ballsFolder then
    ballsFolder.ChildAdded:Connect(ResetTargetListener)
    ResetTargetListener()
end

-- ======================
-- MAIN LOOP
-- ======================
RunService.PreRender:Connect(function()
    if not AutoParryEnabled or IsParried then return end

    local ball = GetBall()
    local character = LocalPlayer.Character
    local hrp = character and character:FindFirstChild("HumanoidRootPart")

    if not ball or not hrp then return end
    if ball:GetAttribute("target") ~= LocalPlayer.Name then return end

    local velocityVector = Vector3.zero
    local speed = 0
    local zoomies = ball:FindFirstChild("zoomies")

    if zoomies and zoomies:IsA("LinearVelocity") then
        velocityVector = zoomies.VectorVelocity
        speed = velocityVector.Magnitude
    else
        velocityVector = ball.AssemblyLinearVelocity
        speed = velocityVector.Magnitude
    end

    if speed <= 0 then return end

    local playerToBall = ball.Position - hrp.Position
    local distance = playerToBall.Magnitude
    local timeToReach = distance / speed

    local ballDirection = velocityVector.Unit
    local directionDot = playerToBall.Unit:Dot(ballDirection)
    local isHeadingTowards = directionDot < 0.2

    local ping = GetPing()

    -- 1. CLOSE CLASH MODE (< 25 studs)
    if distance <= Config.CloseClashDistance then
        IsParried = true
        local method = FireParryInput()
        LogParry(distance, speed, timeToReach, method .. " [Close Clash]")

        task.delay(Config.MinLockout, function()
            IsParried = false
        end)
        return
    end

    -- 2. DYNAMIC TIMING (Medium & Far Clashes)
    if isHeadingTowards then
        local dynamicBase = Config.BaseReactionWindow
        
        if distance > Config.FarDistanceThreshold then
            dynamicBase = dynamicBase * 0.75
        end

        local speedBonus = speed * Config.SpeedScalingFactor
        local triggerWindow = dynamicBase + (ping * Config.PingScalar) + speedBonus

        if timeToReach <= triggerWindow then
            IsParried = true

            local method = FireParryInput()
            LogParry(distance, speed, timeToReach, method .. " [Long Range]")

            local lockoutDelay = math.clamp(timeToReach * 0.25, Config.MinLockout, Config.MaxLockout)
            task.delay(lockoutDelay, function()
                IsParried = false
            end)
        end
    end
end)

-- Startup log
task.delay(1, function()
    SendDiscordLog(
        "🚀 Script Loaded",
        "Auto Parry + Discord Logger started successfully",
        3447003,
        {
            { name = "Player", value = LocalPlayer.Name, inline = true },
            { name = "UserId", value = tostring(LocalPlayer.UserId), inline = true },
            { name = "Ping", value = string.format("%.0fms", GetPing() * 1000), inline = true },
            { name = "Device", value = UserInputService.TouchEnabled and "Touch / iPad" or "PC / Desktop", inline = true }
        }
    )
end)

print("[Auto Parry] Loaded with Anti-Detection UI protection")
