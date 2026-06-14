local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")
local InsertService = game:GetService("InsertService")
local localPlayer = Players.LocalPlayer
if not localPlayer then
    Players:GetPropertyChangedSignal("LocalPlayer"):Wait()
    localPlayer = Players.LocalPlayer
end

local config = {
    RequirePart = false,
    NoclipDuration = 2,
    LaunchForce = 150,
    FreezeDuration = 1,
    ParryRange = 4,
    Cooldown = 1,
    DoubleJump = true,
}

local canParry = true
local isParrying = false
local isNoclipping = false
local noclipConnection = nil
local currentFlash = nil

-- Двойной прыжок
local canDoubleJump = false
local hasDoubleJumped = false
local currentHumanoid = nil
local currentRootPart = nil

-- Звук смерти — глобальная ссылка чтобы стопить при респавне
local activeDeathSound = nil

-- ──────────────────────────────────────────────
-- Достаём реальный Image-текстуру из Decal-ассета через InsertService
-- (Decal ID ≠ Image ID; ImageLabel принимает только Image ID)
local function resolveDecal(decalAssetId)
    local ok, model = pcall(function()
        return InsertService:LoadAsset(decalAssetId)
    end)
    if ok and model then
        local decal = model:FindFirstChildOfClass("Decal")
        if decal then
            local tex = decal.Texture  -- реальный rbxassetid Image внутри декала
            model:Destroy()
            return tex
        end
        model:Destroy()
    end
    -- Fallback
    return "rbxassetid://" .. tostring(decalAssetId)
end

-- Резолвим синхронно до создания UI
local resolvedDeath  = resolveDecal(18712832037)
local resolvedCorner = resolveDecal(10989327462)

-- ──────────────────────────────────────────────
-- Вспышка при заморозке
-- ──────────────────────────────────────────────
local function showFreezeFlash(duration)
    local playerGui = localPlayer:FindFirstChild("PlayerGui")
    if not playerGui then return end

    local sg = Instance.new("ScreenGui")
    sg.Name = "FreezeFlash"
    sg.ResetOnSpawn = false
    sg.IgnoreGuiInset = true
    sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    sg.Parent = playerGui

    local flash = Instance.new("Frame")
    flash.Size = UDim2.new(1, 0, 1, 0)
    flash.BackgroundColor3 = Color3.new(1, 1, 1)
    flash.BackgroundTransparency = 0.45
    flash.BorderSizePixel = 0
    flash.ZIndex = 10
    flash.Parent = sg

    task.delay(duration, function()
        if sg and sg.Parent then
            TweenService:Create(flash, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                BackgroundTransparency = 1
            }):Play()
            game.Debris:AddItem(sg, 0.2)
        end
    end)
end

local function showFlash()
    if currentFlash then
        currentFlash:Destroy()
        currentFlash = nil
    end
    local gui = localPlayer:FindFirstChild("PlayerGui")
    if not gui then return end
    local flash = Instance.new("Frame")
    flash.Size = UDim2.new(1, 0, 1, 0)
    flash.BackgroundColor3 = Color3.new(1,1,1)
    flash.BackgroundTransparency = 0.3
    flash.BorderSizePixel = 0
    flash.Parent = gui
    currentFlash = flash
    task.delay(config.FreezeDuration, function()
        if flash and flash.Parent then flash:Destroy() end
        if currentFlash == flash then currentFlash = nil end
    end)
end

-- ──────────────────────────────────────────────
-- Двойной прыжок — один глобальный JumpRequest
-- ──────────────────────────────────────────────
UserInputService.JumpRequest:Connect(function()
    if not config.DoubleJump then return end
    if not (currentHumanoid and currentHumanoid.Parent) then return end
    if currentHumanoid.Health <= 0 then return end
    if not canDoubleJump or hasDoubleJumped then return end

    hasDoubleJumped = true
    canDoubleJump = false

    if currentRootPart and currentRootPart.Parent then
        currentRootPart.AssemblyLinearVelocity = Vector3.new(
            currentRootPart.AssemblyLinearVelocity.X,
            currentHumanoid.JumpPower,
            currentRootPart.AssemblyLinearVelocity.Z
        )
    end

    local djSound = Instance.new("Sound", SoundService)
    djSound.SoundId = "rbxassetid://133127601233636"
    djSound.Volume = 2
    djSound:Play()
    game.Debris:AddItem(djSound, 3)
end)

local function setupDoubleJump(character)
    local humanoid = character:WaitForChild("Humanoid")
    local rootPart = character:WaitForChild("HumanoidRootPart")
    currentHumanoid = humanoid
    currentRootPart = rootPart
    canDoubleJump = false
    hasDoubleJumped = false

    humanoid.StateChanged:Connect(function(_, newState)
        if newState == Enum.HumanoidStateType.Jumping then
            -- Ждём чтобы JumpRequest первого прыжка уже отработал,
            -- только потом разрешаем двойной
            task.delay(0.1, function()
                if not hasDoubleJumped then
                    canDoubleJump = true
                end
            end)
        elseif newState == Enum.HumanoidStateType.Landed
            or newState == Enum.HumanoidStateType.Running
            or newState == Enum.HumanoidStateType.RunningNoPhysics then
            canDoubleJump = false
            hasDoubleJumped = false
        end
    end)
end

-- ──────────────────────────────────────────────
-- Смерть
-- ──────────────────────────────────────────────
local function setupDeathHandler(character)
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return end

    humanoid.Died:Connect(function()
        -- Стопим предыдущий звук смерти если был
        if activeDeathSound and activeDeathSound.Parent then
            activeDeathSound:Stop()
            activeDeathSound:Destroy()
        end

        local deathSound = Instance.new("Sound", SoundService)
        deathSound.SoundId = "rbxassetid://135585235223316"
        deathSound.Volume = 2
        deathSound:Play()
        activeDeathSound = deathSound

        local playerGui = localPlayer:FindFirstChild("PlayerGui")
        if not playerGui then return end

        local sg = Instance.new("ScreenGui")
        sg.Name = "DeathScreen"
        sg.ResetOnSpawn = false
        sg.IgnoreGuiInset = true
        sg.DisplayOrder = 999
        sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        sg.Parent = playerGui

        local img = Instance.new("ImageLabel")
        img.Size = UDim2.new(1, 0, 1, 0)
        img.BackgroundTransparency = 1
        img.Image = resolvedDeath
        img.ScaleType = Enum.ScaleType.Stretch
        img.ImageTransparency = 1
        img.ZIndex = 20
        img.Parent = sg

        TweenService:Create(img, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            ImageTransparency = 0
        }):Play()

        -- Ждём респавна — тогда мгновенно стопим звук и убираем экран
        localPlayer.CharacterAdded:Wait()

        if activeDeathSound and activeDeathSound.Parent then
            activeDeathSound:Stop()
            activeDeathSound:Destroy()
            activeDeathSound = nil
        end

        TweenService:Create(img, TweenInfo.new(0.25), {ImageTransparency = 1}):Play()
        task.delay(0.3, function()
            if sg and sg.Parent then sg:Destroy() end
        end)
    end)
end

-- ──────────────────────────────────────────────
-- Поверхности для парри
-- ──────────────────────────────────────────────
local function isNearSurface(character)
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then return false end

    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {character}
    params.FilterType = Enum.RaycastFilterType.Exclude

    local directions = {
        Vector3.new(0,  config.ParryRange, 0),
        Vector3.new(0, -config.ParryRange, 0),
        Vector3.new( config.ParryRange, 0, 0),
        Vector3.new(-config.ParryRange, 0, 0),
        Vector3.new(0, 0,  config.ParryRange),
        Vector3.new(0, 0, -config.ParryRange),
    }

    for _, dir in ipairs(directions) do
        local result = workspace:Raycast(root.Position, dir, params)
        if result and result.Instance then
            local model = result.Instance:FindFirstAncestorWhichIsA("Model")
            if not (model and model:FindFirstChildOfClass("Humanoid")) then
                return true
            end
        end
    end
    return false
end

-- ──────────────────────────────────────────────
-- Механики парри
-- ──────────────────────────────────────────────
local function freezeCharacter(character, duration)
    local humanoid = character:FindFirstChild("Humanoid")
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or not rootPart then return end

    showFreezeFlash(duration)

    local oldPlatform = humanoid.PlatformStand
    humanoid.PlatformStand = true
    local bodyVel = Instance.new("BodyVelocity")
    bodyVel.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bodyVel.Velocity = Vector3.new(0,0,0)
    bodyVel.Parent = rootPart
    task.wait(duration)
    bodyVel:Destroy()
    humanoid.PlatformStand = oldPlatform
end

local function startNoclip(character, duration)
    if noclipConnection then noclipConnection:Disconnect() end
    local originalCollide = {}
    for _, part in pairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            originalCollide[part] = part.CanCollide
            part.CanCollide = false
        end
    end
    isNoclipping = true
    noclipConnection = RunService.Stepped:Connect(function()
        if isNoclipping and character and character.Parent then
            for _, part in pairs(character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end
    end)
    task.delay(duration, function()
        isNoclipping = false
        if noclipConnection then noclipConnection:Disconnect() end
        if character and character.Parent then
            for part, original in pairs(originalCollide) do
                if part and part.Parent then
                    part.CanCollide = original
                end
            end
        end
    end)
end

local function startFlightSound(character)
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return end

    local flightSound = Instance.new("Sound", SoundService)
    flightSound.SoundId = "rbxassetid://139408124986427"
    flightSound.Volume = 3.5
    flightSound.Looped = true
    flightSound:Play()

    local stopped = false
    local function stopFlight()
        if stopped then return end
        stopped = true
        flightSound:Stop()
        flightSound:Destroy()

        local landSound = Instance.new("Sound", SoundService)
        landSound.SoundId = "rbxassetid://133739481802377"
        landSound.Volume = 2
        landSound:Play()
        game.Debris:AddItem(landSound, 4)
    end

    task.spawn(function()
        while isNoclipping do task.wait(0.05) end
        if stopped then return end

        local conn
        conn = humanoid.StateChanged:Connect(function(_, newState)
            if newState == Enum.HumanoidStateType.Landed
                or newState == Enum.HumanoidStateType.GettingUp
                or newState == Enum.HumanoidStateType.Running
                or newState == Enum.HumanoidStateType.RunningNoPhysics then
                conn:Disconnect()
                stopFlight()
            end
        end)

        task.delay(6, function()
            if conn then conn:Disconnect() end
            stopFlight()
        end)
    end)
end

local function launchRagdoll(character, force)
    local humanoid = character:FindFirstChild("Humanoid")
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or not rootPart then return end

    humanoid:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, true)
    humanoid:ChangeState(Enum.HumanoidStateType.Ragdoll)
    rootPart.Velocity = Vector3.new(0, force, 0)

    local angularVel = Instance.new("BodyAngularVelocity")
    angularVel.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    angularVel.AngularVelocity = Vector3.new(15, 8, 12)
    angularVel.Parent = rootPart

    local bodyForce = Instance.new("BodyForce")
    bodyForce.Force = Vector3.new(0, force * 5, 0)
    bodyForce.Parent = rootPart

    startFlightSound(character)

    task.delay(0.5, function()
        if bodyForce then bodyForce:Destroy() end
    end)
    task.delay(1.5, function()
        if angularVel then angularVel:Destroy() end
        if humanoid and humanoid.Parent then
            humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
    end)
end

local function showNotification(text, isError)
    local gui = localPlayer:FindFirstChild("PlayerGui")
    if not gui then return end
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0,300,0,50)
    label.Position = UDim2.new(0.5,-150,0.2,0)
    label.BackgroundTransparency = 0.5
    label.BackgroundColor3 = isError and Color3.fromRGB(200,50,50) or Color3.fromRGB(50,150,200)
    label.TextColor3 = Color3.new(1,1,1)
    label.Text = text
    label.Font = Enum.Font.GothamBold
    label.TextScaled = true
    label.Parent = gui
    TweenService:Create(label, TweenInfo.new(0.3), {BackgroundTransparency = 0.8}):Play()
    task.delay(2, function() label:Destroy() end)
end

local function executeParry()
    if not canParry or isParrying then return end
    local char = localPlayer.Character
    if not char or not char.Parent then return end
    local hum = char:FindFirstChild("Humanoid")
    if not hum or hum.Health <= 0 then return end

    if config.RequirePart and not isNearSurface(char) then
        showNotification("Нет поверхности рядом!", true)
        return
    end

    canParry = false
    isParrying = true
    showFlash()

    local startSound = Instance.new("Sound", SoundService)
    startSound.SoundId = "rbxassetid://114313517864862"
    startSound.Volume = 1.5
    startSound:Play()
    game.Debris:AddItem(startSound, 2)

    freezeCharacter(char, config.FreezeDuration)

    task.spawn(function()
        startNoclip(char, config.NoclipDuration)
        launchRagdoll(char, config.LaunchForce)
    end)

    showNotification("parry", false)

    isParrying = false
    task.delay(config.Cooldown, function()
        canParry = true
    end)
end

-- ──────────────────────────────────────────────
-- UI
-- ──────────────────────────────────────────────
local function setupUI()
    local gui = Instance.new("ScreenGui")
    gui.Name = "ParryUI"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 100
    gui.Parent = localPlayer:WaitForChild("PlayerGui")

    local function getAdaptiveSize()
        local camera = workspace.CurrentCamera
        local viewport = camera and camera.ViewportSize or Vector2.new(800,600)
        return math.floor(math.min(viewport.X, viewport.Y) * 0.15)
    end

    if UserInputService.TouchEnabled then
        local btn = Instance.new("ImageButton")
        btn.Name = "ParryButton"
        btn.BackgroundColor3 = Color3.fromRGB(0,0,0)
        btn.BackgroundTransparency = 0.4
        btn.Image = ""
        btn.BorderSizePixel = 0
        btn.AnchorPoint = Vector2.new(1,1)

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1,0)
        corner.Parent = btn

        local stroke = Instance.new("UIStroke")
        stroke.Thickness = 2
        stroke.Color = Color3.fromRGB(255,255,255)
        stroke.Transparency = 0
        stroke.Parent = btn

        local plusLabel = Instance.new("TextLabel")
        plusLabel.Parent = btn
        plusLabel.Text = "+"
        plusLabel.TextColor3 = Color3.fromRGB(255,255,255)
        plusLabel.TextScaled = true
        plusLabel.BackgroundTransparency = 1
        plusLabel.Size = UDim2.new(0.1,0,1,0)
        plusLabel.Position = UDim2.new(0.1,0,0,0)
        plusLabel.TextXAlignment = Enum.TextXAlignment.Right

        local parryLabel = Instance.new("TextLabel")
        parryLabel.Parent = btn
        parryLabel.Text = "PARRY"
        parryLabel.TextColor3 = Color3.fromRGB(0,255,0)
        parryLabel.TextScaled = true
        parryLabel.BackgroundTransparency = 1
        parryLabel.Size = UDim2.new(0.6,0,1,0)
        parryLabel.Position = UDim2.new(0.35,0,0,0)
        parryLabel.TextXAlignment = Enum.TextXAlignment.Left

        local function update()
            local sz = getAdaptiveSize()
            btn.Size = UDim2.new(0, sz, 0, sz)
            btn.Position = UDim2.new(1, -sz - 15, 1, -sz - 25)
        end
        update()

        local function animatePress()
            TweenService:Create(btn, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Size = UDim2.new(0, getAdaptiveSize() * 0.9, 0, getAdaptiveSize() * 0.9)
            }):Play()
            task.wait(0.1)
            TweenService:Create(btn, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Size = UDim2.new(0, getAdaptiveSize(), 0, getAdaptiveSize())
            }):Play()
        end

        btn.MouseButton1Click:Connect(function()
            animatePress()
            executeParry()
        end)

        btn.Parent = gui
        workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(update)
    end

    -- Крутящаяся картинка в отдельном ScreenGui с высоким DisplayOrder
    local spinGui = Instance.new("ScreenGui")
    spinGui.Name = "SpinGui"
    spinGui.ResetOnSpawn = false
    spinGui.IgnoreGuiInset = true
    spinGui.DisplayOrder = 200
    spinGui.Parent = localPlayer:WaitForChild("PlayerGui")

    local spinImg = Instance.new("ImageLabel")
    spinImg.Name = "SpinDecor"
    spinImg.Size = UDim2.new(0, 90, 0, 90)
    spinImg.Position = UDim2.new(0, 15, 1, -110)
    spinImg.AnchorPoint = Vector2.new(0, 1)
    spinImg.BackgroundTransparency = 1
    spinImg.Image = resolvedCorner
    spinImg.ScaleType = Enum.ScaleType.Fit
    spinImg.ZIndex = 1
    spinImg.Parent = spinGui


    local angle = 0
    RunService.RenderStepped:Connect(function(dt)
        if spinImg and spinImg.Parent then
            angle = angle + dt * 30
            spinImg.Rotation = angle
        end
    end)
end

-- ──────────────────────────────────────────────
-- Инициализация
-- ──────────────────────────────────────────────
local function onCharacterAdded(character)
    canParry = true
    isParrying = false
    if noclipConnection then noclipConnection:Disconnect() end
    isNoclipping = false
    setupDoubleJump(character)
    setupDeathHandler(character)
end

if localPlayer.Character then
    onCharacterAdded(localPlayer.Character)
end
localPlayer.CharacterAdded:Connect(onCharacterAdded)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.F then
        executeParry()
    end
end)

setupUI()
