-- Dragon Ball Legends Avançado - Mobile
-- Goku Player + Freeza NPC atacante + VFX detalhado
-- Client-side

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local player = Players.LocalPlayer
local goku = player.Character
local freeza

-- Configuração personagens e habilidades
local Characters = {
    Goku = {
        Abilities = {
            Kamehameha = {Damage = 50, Speed = 150, Color = Color3.fromRGB(0,162,255), Size = Vector3.new(3,3,10)},
            GenkiDama = {Damage = 200, Speed = 100, Color = Color3.fromRGB(255,255,255), Size = Vector3.new(5,5,5)}
        },
        Transformations = {
            SuperSaiyajin = {Color = Color3.fromRGB(255,255,0), SizeMultiplier = 1.5}
        },
        CounterActive = false
    },
    Freeza = {
        Model = "FreezaModel",
        FollowDistance = 8,
        AttackInterval = 3 -- tempo entre ataques
    }
}

-- Spawn Freeza
local function spawnFreeza()
    local template = ReplicatedStorage:WaitForChild("FreezaModel")
    freeza = template:Clone()
    freeza.Parent = workspace
    freeza:SetPrimaryPartCFrame(goku.PrimaryPart.CFrame + Vector3.new(5,0,0))
end

-- Freeza segue Goku
local function followPlayer()
    RunService.Heartbeat:Connect(function()
        if freeza and freeza.PrimaryPart then
            local targetPos = goku.PrimaryPart.Position - (goku.PrimaryPart.CFrame.LookVector * Characters.Freeza.FollowDistance)
            freeza:SetPrimaryPartCFrame(CFrame.new(targetPos, goku.PrimaryPart.Position))
        end
    end)
end

-- Transformação Goku
local function transformGoku()
    for _, part in pairs(goku:GetDescendants()) do
        if part:IsA("BasePart") then
            local tweenInfo = TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
            local tween = TweenService:Create(part, tweenInfo, {Color = Characters.Goku.Transformations.SuperSaiyajin.Color})
            tween:Play()
        end
    end
    -- VFX de aura
    local aura = Instance.new("ParticleEmitter")
    aura.Color = ColorSequence.new(Characters.Goku.Transformations.SuperSaiyajin.Color)
    aura.Rate = 200
    aura.Lifetime = NumberRange.new(0.8)
    aura.Speed = NumberRange.new(2,5)
    aura.Parent = goku.PrimaryPart
    Debris:AddItem(aura, 3)
end

-- Função para criar projétil com VFX detalhado
local function createProjectile(originPart, abilityName, targetPos)
    local ability = Characters.Goku.Abilities[abilityName]
    if not ability then return end

    local proj = Instance.new("Part")
    proj.Size = ability.Size
    proj.Shape = Enum.PartType.Ball
    proj.Material = Enum.Material.Neon
    proj.Color = ability.Color
    proj.Anchored = false
    proj.CanCollide = false
    proj.Position = originPart.Position + Vector3.new(0,3,0)
    proj.Parent = workspace

    -- Luz do projétil
    local light = Instance.new("PointLight")
    light.Color = ability.Color
    light.Brightness = 5
    light.Range = 10
    light.Parent = proj

    -- Partículas do projétil
    local fx = Instance.new("ParticleEmitter")
    fx.Color = ColorSequence.new(ability.Color)
    fx.Speed = NumberRange.new(5,15)
    fx.Lifetime = NumberRange.new(0.3,0.6)
    fx.Rate = 200
    fx.Parent = proj

    local direction = (targetPos - proj.Position).Unit
    local speed = ability.Speed

    local conn
    conn = RunService.Heartbeat:Connect(function(dt)
        if proj.Parent == nil then conn:Disconnect() return end
        proj.CFrame = proj.CFrame + direction * speed * dt
        -- Impacto
        for _, obj in pairs(workspace:GetChildren()) do
            if obj:IsA("Model") and obj.PrimaryPart and obj ~= goku and obj ~= freeza then
                if (proj.Position - obj.PrimaryPart.Position).Magnitude < 5 then
                    -- Efeito de impacto
                    local impact = Instance.new("ParticleEmitter")
                    impact.Color = ColorSequence.new(Color3.fromRGB(255,200,50))
                    impact.Rate = 300
                    impact.Lifetime = NumberRange.new(0.3)
                    impact.Speed = NumberRange.new(5,10)
                    impact.Parent = obj.PrimaryPart
                    Debris:AddItem(impact,0.5)
                    proj:Destroy()
                    conn:Disconnect()
                end
            end
        end
    end)
end

-- Counter (reflete projéteis)
local function activateCounter(duration)
    Characters.Goku.CounterActive = true
    local shield = Instance.new("ParticleEmitter")
    shield.Color = ColorSequence.new(Color3.fromRGB(0,255,255))
    shield.Rate = 300
    shield.Lifetime = NumberRange.new(duration)
    shield.Speed = NumberRange.new(5,10)
    shield.Parent = goku.PrimaryPart
    Debris:AddItem(shield,duration)
    wait(duration)
    Characters.Goku.CounterActive = false
end

-- Freeza ataca inimigos automaticamente
local function freezaAttack()
    while freeza and freeza.Parent do
        local target
        for _, p in pairs(Players:GetPlayers()) do
            if p.Character and p.Character.PrimaryPart and p ~= player then
                target = p.Character.PrimaryPart
                break
            end
        end
        if target then
            createProjectile(freeza.PrimaryPart, "Kamehameha", target.Position)
        end
        wait(Characters.Freeza.AttackInterval)
    end
end

-- Mobile Controls (toque 1x = Kamehameha, 2x = Genki Dama, swipe = counter)
local lastTap = 0
local tapCount = 0

UserInputService.TouchTap:Connect(function(touches, isProcessed)
    if isProcessed then return end
    local currentTime = tick()
    if currentTime - lastTap < 0.5 then
        tapCount = tapCount + 1
    else
        tapCount = 1
    end
    lastTap = currentTime

    local touchPos = touches[1].Position
    local worldPos = workspace.CurrentCamera:ScreenPointToRay(touchPos.X,touchPos.Y).Origin

    if tapCount == 1 then
        createProjectile(goku.PrimaryPart, "Kamehameha", worldPos)
    elseif tapCount == 2 then
        createProjectile(goku.PrimaryPart, "GenkiDama", worldPos)
    end
end)

UserInputService.TouchMoved:Connect(function(touch, delta)
    if delta.Magnitude > 50 then
        activateCounter(2)
    end
end)

-- Inicialização
spawnFreeza()
followPlayer()
transformGoku()
freezaAttack()
