--[[
================================================================================
    MECHA MILITAR GIGANTE — LocalScript Único
    Sistema de pilotagem com entrada cinematográfica
    
    COMO FUNCIONA:
    - Um mecha militar gigante (~35 studs de altura) é criado no mundo.
    - O jogador se aproxima, aparece um prompt "ENTRAR".
    - Ao ativar: a porta do cockpit abre, um cabo de energia sai do peito,
      agarra o jogador e o puxa para dentro. A porta fecha. Você pilota.
    - Para sair: aperta o botão SAIR. A porta abre, você é ejetado.
    
    CONTROLES:
    - Mobile: joystick padrão + botões na tela (SAIR, TURBO, BOOST)
    - PC: WASD move | Shift = turbo | Space = pulo | Y = sair
    
    LIMITAÇÃO: Client-side. Outros jogadores não veem o mecha.
================================================================================
]]

-- ============================================================================
-- SERVIÇOS
-- ============================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ContextActionService = game:GetService("ContextActionService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = workspace.CurrentCamera

-- ============================================================================
-- CONFIGURAÇÃO
-- ============================================================================
local CFG = {
    -- Dimensões do mecha (em studs). Escala gigante militar.
    Scale = {
        Foot       = Vector3.new(4.5, 1.8, 7.5),
        LowerLeg   = Vector3.new(4, 8.5, 4.5),
        Knee       = Vector3.new(4.5, 2.2, 5),
        UpperLeg   = Vector3.new(5, 9, 5.5),
        Hip        = Vector3.new(5.5, 3, 5.5),
        Pelvis     = Vector3.new(10, 4, 6.5),
        Abdomen    = Vector3.new(9, 3.5, 5.5),
        Chest      = Vector3.new(14, 11, 8.5),
        ChestPlate = Vector3.new(11, 7.5, 1),
        Neck       = Vector3.new(3.5, 2, 3.5),
        Head       = Vector3.new(5, 4.5, 5.5),
        SensorPod  = Vector3.new(3, 1.5, 3),
        Shoulder   = Vector3.new(6, 5, 6.5),
        UpperArm   = Vector3.new(3.8, 7.5, 4.3),
        Elbow      = Vector3.new(4.3, 2.2, 4.8),
        LowerArm   = Vector3.new(4.3, 8, 4.8),
        Hand       = Vector3.new(4, 4.5, 3.5),
        -- Cockpit
        CockpitInterior = Vector3.new(6.5, 6, 4),
        CockpitDoorL    = Vector3.new(3.2, 6, 0.5),
        CockpitDoorR    = Vector3.new(3.2, 6, 0.5),
        CockpitFrame    = Vector3.new(7, 0.6, 0.7),
    },
    
    -- Cores militares
    Colors = {
        Primary   = Color3.fromRGB(58, 62, 45),      -- Verde-oliva militar
        Secondary = Color3.fromRGB(45, 48, 55),       -- Cinza escuro
        Metal     = Color3.fromRGB(88, 92, 100),      -- Metal
        Dark      = Color3.fromRGB(22, 25, 30),       -- Preto
        Warning   = Color3.fromRGB(220, 160, 40),     -- Amarelo de aviso
        Danger    = Color3.fromRGB(180, 45, 40),      -- Vermelho
        Glow      = Color3.fromRGB(80, 220, 255),     -- Ciano do reator
        Cable     = Color3.fromRGB(255, 200, 60),     -- Cabo dourado
        Camo      = Color3.fromRGB(78, 84, 62),       -- Verde mais claro
    },
    
    -- Movimento
    Move = {
        WalkSpeed = 32,
        RunSpeed  = 55,
        JumpPower = 90,
        TurnRate  = 3.2,       -- rad/s de rotação
    },
    
    -- Entrada
    Entry = {
        DoorOpenTime    = 0.9,
        CableExtendTime = 0.35,
        PullInTime      = 0.9,
        DoorCloseTime   = 0.7,
        PromptRange     = 16,
    },
    
    -- Câmera
    Camera = {
        Fov = { Humanoid = 70, Piloting = 82, Boost = 96 },
        Offset = { Humanoid = Vector3.new(0,0,0), Piloting = Vector3.new(0, 3, 0) },
        ZoomOut = { Humanoid = 0, Piloting = 12 },
    },
    
    -- Turbo
    Boost = { Max = 100, Drain = 32, Regen = 18, RegenDelay = 1.0, MinToStart = 15 },
}

local C = CFG.Colors
local SZ = CFG.Scale

-- ============================================================================
-- UTILITÁRIOS
-- ============================================================================
local function makePart(props, parent)
    local p = Instance.new("Part")
    p.Anchored = false
    p.CanCollide = false
    p.CanTouch = false
    p.CanQuery = false
    p.Massless = true
    p.Material = Enum.Material.Metal
    p.TopSurface = Enum.SurfaceType.Smooth
    p.BottomSurface = Enum.SurfaceType.Smooth
    for k, v in pairs(props) do p[k] = v end
    p.Parent = parent
    return p
end

local function weld(p0, p1, c0, c1)
    local w = Instance.new("Weld")
    w.Part0 = p0
    w.Part1 = p1
    w.C0 = c0 or CFrame.new()
    w.C1 = c1 or CFrame.new()
    w.Parent = p0
    return w
end

local function motor(p0, p1, c0, c1, name)
    local m = Instance.new("Motor6D")
    m.Name = name or "Joint"
    m.Part0 = p0
    m.Part1 = p1
    m.C0 = c0 or CFrame.new()
    m.C1 = c1 or CFrame.new()
    m.Parent = p0
    return m
end

local function tween(inst, props, dur, style, dir)
    return TweenService:Create(inst, TweenInfo.new(
        dur, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out
    ), props)
end

-- ============================================================================
-- ESTADO GLOBAL
-- ============================================================================
local S = {
    mecha = nil,              -- Model
    root = nil,               -- HRP do mecha
    joints = {},              -- Referência às juntas principais
    cockpit = nil,            -- Attachment do assento
    cockpitDoorL = nil,
    cockpitDoorR = nil,
    cable = nil,              -- Part do cabo
    prompt = nil,             -- ProximityPrompt
    
    piloting = false,
    entering = false,
    exiting = false,
    
    savedHumanoid = nil,      -- Estado original do Humanoid do player
    savedRootCF = nil,
    
    speed = 0,
    boosting = false,
    energy = CFG.Boost.Max,
    lastBoostEnd = 0,
    
    camFov = 70,
    camZoom = 0,
    bobPhase = 0,
    lastStepIdx = 0,
    
    fx = {},                  -- Referências de efeitos
}

-- ============================================================================
-- CONSTRUÇÃO DO MECHA MILITAR
-- ============================================================================
local function buildMecha(position, facing)
    local model = Instance.new("Model")
    model.Name = "MilitaryMecha_Giant"
    
    -- Raiz (HRP)
    local root = makePart({
        Name = "HumanoidRootPart",
        Size = Vector3.new(4, 4, 4),
        Transparency = 1,
        CanCollide = false,
        CFrame = CFrame.new(position) * CFrame.Angles(0, facing, 0),
        Anchored = true,
    }, model)
    model.PrimaryPart = root
    
    -- Helper: cria uma peça e liga por Motor6D a um pai
    local function limb(name, size, cfOffset, parentPart, pivotOffset, color, material)
        local part = makePart({
            Name = name,
            Size = size,
            Color = color or C.Primary,
            Material = material or Enum.Material.Metal,
            CFrame = root.CFrame,
        }, model)
        
        local pivot = pivotOffset or Vector3.new(0, size.Y * 0.5, 0)
        local c1 = CFrame.new(-pivot)
        motor(parentPart, part, cfOffset, c1, name .. "_Joint")
        
        return part
    end
    
    -- ==== ESQUELETO / ORDEM HIERÁRQUICA ====
    -- Root -> Pelvis -> {Abdomen -> Chest -> {Head, ShoulderL/R -> Arms}, HipL/R -> Legs}
    
    -- PELVIS
    local pelvis = limb("Pelvis", SZ.Pelvis, CFrame.new(0, 22, 0), root, Vector3.new(0, 0, 0), C.Primary)
    
    -- ABDOMEN
    local abdomen = limb("Abdomen", SZ.Abdomen, CFrame.new(0, 2.8, 0), pelvis, Vector3.new(0, -SZ.Abdomen.Y * 0.5, 0), C.Secondary)
    
    -- CHEST (torso principal)
    local chest = limb("Chest", SZ.Chest, CFrame.new(0, 6.5, 0), abdomen, Vector3.new(0, -SZ.Chest.Y * 0.5, 0), C.Primary)
    
    -- HEAD (pequena, militar, com sensor)
    local head = limb("Head", SZ.Head, CFrame.new(0, SZ.Chest.Y * 0.5 + SZ.Head.Y * 0.5 + 0.5, 0), chest,
        Vector3.new(0, -SZ.Head.Y * 0.5, 0), C.Secondary)
    
    -- Sensor pod no topo da cabeça
    local sensor = makePart({
        Name = "SensorPod", Size = SZ.SensorPod, Color = C.Dark,
        Material = Enum.Material.SmoothPlastic,
        CFrame = root.CFrame,
    }, model)
    weld(head, sensor, CFrame.new(0, SZ.Head.Y * 0.5 + SZ.SensorPod.Y * 0.5, -1.2), CFrame.new())
    
    -- Visor / olhos (neon)
    local visor = makePart({
        Name = "Visor", Size = Vector3.new(3.5, 0.8, 0.2), Color = C.Glow,
        Material = Enum.Material.Neon, CFrame = root.CFrame,
    }, model)
    weld(head, visor, CFrame.new(0, 0.4, -SZ.Head.Z * 0.5 - 0.05), CFrame.new())
    
    -- Antena
    local antenna = makePart({
        Name = "Antenna", Size = Vector3.new(0.3, 4, 0.3), Color = C.Metal,
        Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    weld(head, antenna, CFrame.new(2, SZ.Head.Y * 0.5 + 2, 0), CFrame.new())
    
    -- ==== SHOULDERS + BRAÇOS ====
    for _, side in ipairs({ { -1, "Left" }, { 1, "Right" } }) do
        local sx, sn = side[1], side[2]
        
        -- Ombreira grande militar
        local shoulder = limb(sn .. "Shoulder", SZ.Shoulder,
            CFrame.new(sx * (SZ.Chest.X * 0.5 + SZ.Shoulder.X * 0.5 - 0.5), SZ.Chest.Y * 0.4, 0),
            chest, Vector3.new(0, 0, 0), C.Secondary)
        
        -- Detalhe amarelo na ombreira
        local stripe = makePart({
            Name = sn .. "ShoulderStripe", Size = Vector3.new(SZ.Shoulder.X + 0.2, 0.6, SZ.Shoulder.Z + 0.2),
            Color = C.Warning, Material = Enum.Material.Metal, CFrame = root.CFrame,
        }, model)
        weld(shoulder, stripe, CFrame.new(0, SZ.Shoulder.Y * 0.35, 0), CFrame.new())
        
        -- Braço superior
        local upperArm = limb(sn .. "UpperArm", SZ.UpperArm,
            CFrame.new(0, -SZ.Shoulder.Y * 0.5 - SZ.UpperArm.Y * 0.4, 0), shoulder,
            Vector3.new(0, SZ.UpperArm.Y * 0.5, 0), C.Primary)
        
        -- Cotovelo
        local elbow = limb(sn .. "Elbow", SZ.Elbow,
            CFrame.new(0, -SZ.UpperArm.Y * 0.5, 0), upperArm,
            Vector3.new(0, SZ.Elbow.Y * 0.5, 0), C.Dark)
        
        -- Antebraço
        local lowerArm = limb(sn .. "LowerArm", SZ.LowerArm,
            CFrame.new(0, -SZ.Elbow.Y * 0.5 - SZ.LowerArm.Y * 0.4, 0), elbow,
            Vector3.new(0, SZ.LowerArm.Y * 0.5, 0), C.Primary)
        
        -- Mão
        local hand = limb(sn .. "Hand", SZ.Hand,
            CFrame.new(0, -SZ.LowerArm.Y * 0.5 - SZ.Hand.Y * 0.3, 0), lowerArm,
            Vector3.new(0, SZ.Hand.Y * 0.5, 0), C.Secondary)
    end
    
    -- ==== COCKPIT NO PEITO ====
    -- Interior (fundo escuro)
    local cockpitInterior = makePart({
        Name = "CockpitInterior", Size = SZ.CockpitInterior,
        Color = Color3.fromRGB(12, 15, 20), Material = Enum.Material.SmoothPlastic,
        CFrame = root.CFrame,
    }, model)
    weld(chest, cockpitInterior,
        CFrame.new(0, -1, -SZ.Chest.Z * 0.5 + SZ.CockpitInterior.Z * 0.5 - 0.1),
        CFrame.new())
    
    -- Assento (onde o player fica)
    local seatAttachment = Instance.new("Attachment")
    seatAttachment.Name = "CockpitSeat"
    seatAttachment.Position = Vector3.new(0, -1.5, -SZ.Chest.Z * 0.5 + SZ.CockpitInterior.Z * 0.5 + 0.5)
    seatAttachment.Parent = chest
    
    -- Frame superior da porta
    local topFrame = makePart({
        Name = "CockpitTopFrame", Size = SZ.CockpitFrame,
        Color = C.Warning, Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    weld(chest, topFrame,
        CFrame.new(0, SZ.CockpitInterior.Y * 0.5 + 0.3, -SZ.Chest.Z * 0.5 - 0.2),
        CFrame.new())
    
    -- Portas (esquerda e direita)
    local doorL = makePart({
        Name = "CockpitDoorL", Size = SZ.CockpitDoorL, Color = C.Metal,
        Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    local doorLClosed = CFrame.new(
        -SZ.CockpitDoorL.X * 0.5, -1,
        -SZ.Chest.Z * 0.5 - SZ.CockpitDoorL.Z * 0.5
    )
    weld(chest, doorL, doorLClosed, CFrame.new())
    
    local doorR = makePart({
        Name = "CockpitDoorR", Size = SZ.CockpitDoorR, Color = C.Metal,
        Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    local doorRClosed = CFrame.new(
        SZ.CockpitDoorR.X * 0.5, -1,
        -SZ.Chest.Z * 0.5 - SZ.CockpitDoorR.Z * 0.5
    )
    weld(chest, doorR, doorRClosed, CFrame.new())
    
    -- Detalhes de painéis na porta (warning stripes)
    local stripeL = makePart({
        Name = "DoorStripeL", Size = Vector3.new(SZ.CockpitDoorL.X - 0.4, 0.4, SZ.CockpitDoorL.Z + 0.1),
        Color = C.Warning, Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    weld(doorL, stripeL, CFrame.new(0, SZ.CockpitDoorL.Y * 0.35, 0), CFrame.new())
    
    local stripeR = makePart({
        Name = "DoorStripeR", Size = Vector3.new(SZ.CockpitDoorR.X - 0.4, 0.4, SZ.CockpitDoorR.Z + 0.1),
        Color = C.Warning, Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    weld(doorR, stripeR, CFrame.new(0, SZ.CockpitDoorR.Y * 0.35, 0), CFrame.new())
    
    -- ==== PERNAS ====
    for _, side in ipairs({ { -1, "Left" }, { 1, "Right" } }) do
        local sx, sn = side[1], side[2]
        
        -- Quadril
        local hip = limb(sn .. "Hip", SZ.Hip,
            CFrame.new(sx * (SZ.Pelvis.X * 0.5 - SZ.Hip.X * 0.3), -SZ.Pelvis.Y * 0.4, 0),
            pelvis, Vector3.new(0, 0, 0), C.Secondary)
        
        -- Coxa
        local upperLeg = limb(sn .. "UpperLeg", SZ.UpperLeg,
            CFrame.new(0, -SZ.Hip.Y * 0.5 - SZ.UpperLeg.Y * 0.4, 0), hip,
            Vector3.new(0, SZ.UpperLeg.Y * 0.5, 0), C.Primary)
        
        -- Joelho (armadura)
        local knee = limb(sn .. "Knee", SZ.Knee,
            CFrame.new(0, -SZ.UpperLeg.Y * 0.5, 0), upperLeg,
            Vector3.new(0, SZ.Knee.Y * 0.5, 0), C.Dark)
        
        -- Canela
        local lowerLeg = limb(sn .. "LowerLeg", SZ.LowerLeg,
            CFrame.new(0, -SZ.Knee.Y * 0.5 - SZ.LowerLeg.Y * 0.4, 0), knee,
            Vector3.new(0, SZ.LowerLeg.Y * 0.5, 0), C.Primary)
        
        -- Pé grande e pesado
        local foot = limb(sn .. "Foot", SZ.Foot,
            CFrame.new(0, -SZ.LowerLeg.Y * 0.5 - SZ.Foot.Y * 0.3, -SZ.Foot.Z * 0.15), lowerLeg,
            Vector3.new(0, SZ.Foot.Y * 0.5, 0), C.Secondary)
        
        -- Detalhe no pé (sola)
        local sole = makePart({
            Name = sn .. "Sole", Size = Vector3.new(SZ.Foot.X + 0.3, 0.4, SZ.Foot.Z + 0.3),
            Color = C.Dark, Material = Enum.Material.Metal, CFrame = root.CFrame,
        }, model)
        weld(foot, sole, CFrame.new(0, -SZ.Foot.Y * 0.5 + 0.2, 0), CFrame.new())
    end
    
    -- ==== DETALHES EXTRAS MILITARES ====
    -- Reator central no peito
    local reactor = makePart({
        Name = "ReactorCore", Size = Vector3.new(2.2, 2.2, 0.4),
        Color = C.Glow, Material = Enum.Material.Neon, CFrame = root.CFrame,
    }, model)
    weld(chest, reactor, CFrame.new(0, 1.5, -SZ.Chest.Z * 0.5 - 0.15), CFrame.new())
    
    -- Luz do reator
    local reactorLight = Instance.new("PointLight")
    reactorLight.Color = C.Glow
    reactorLight.Brightness = 2
    reactorLight.Range = 30
    reactorLight.Parent = reactor
    
    -- Mochila traseira (fuel tank / equipment)
    local backpack = makePart({
        Name = "Backpack", Size = Vector3.new(SZ.Chest.X * 0.7, SZ.Chest.Y * 0.7, 2.5),
        Color = C.Secondary, Material = Enum.Material.Metal, CFrame = root.CFrame,
    }, model)
    weld(chest, backpack, CFrame.new(0, 1, SZ.Chest.Z * 0.5 + 1.25), CFrame.new())
    
    -- Vents (grade de ventilação)
    for i = -2, 2 do
        local vent = makePart({
            Name = "Vent" .. i, Size = Vector3.new(0.5, 3, 0.3),
            Color = C.Dark, Material = Enum.Material.Metal, CFrame = root.CFrame,
        }, model)
        weld(backpack, vent, CFrame.new(i * 1, 0, 1.35), CFrame.new())
    end
    
    -- Cabo de entrada (escondido inicialmente, sai do cockpit)
    local cableBase = Instance.new("Attachment")
    cableBase.Name = "CableOrigin"
    cableBase.Position = Vector3.new(0, 0, -SZ.Chest.Z * 0.5 - 0.5)
    cableBase.Parent = chest
    
    local cable = makePart({
        Name = "EntryCable", Size = Vector3.new(0.35, 0.35, 0.1),
        Color = C.Cable, Material = Enum.Material.Neon, CFrame = root.CFrame,
        Transparency = 1,
    }, model)
    cable.Transparency = 1
    
    model.Parent = workspace
    
    -- Guardar referências
    S.joints = {
        pelvis = pelvis, abdomen = abdomen, chest = chest, head = head,
        doorL = doorL, doorR = doorR,
        pelvisJoint = pelvis:FindFirstChild("Pelvis_Joint"),
        chestJoint = chest:FindFirstChild("Chest_Joint"),
        headJoint = head:FindFirstChild("Head_Joint"),
    }
    S.cockpit = seatAttachment
    S.cockpitDoorL = doorL
    S.cockpitDoorR = doorR
    S.cable = cable
    S.cableBase = cableBase
    S.fx.reactor = reactorLight
    S.fx.visor = visor
    S.fx.doorClosedL = doorLClosed
    S.fx.doorClosedR = doorRClosed
    S.root = root
    S.mecha = model
    
    return model
end

-- ============================================================================
-- PROXIMITY PROMPT
-- ============================================================================
local function attachPrompt(mecha)
    local chest = S.joints.chest
    local prompt = Instance.new("ProximityPrompt")
    prompt.Name = "MechaEntryPrompt"
    prompt.ActionText = "ENTRAR NO MECHA"
    prompt.ObjectText = "Unidade Militar-01"
    prompt.KeyboardKeyCode = Enum.KeyCode.E
    prompt.GamepadKeyCode = Enum.KeyCode.ButtonX
    prompt.HoldDuration = 0.3
    prompt.MaxActivationDistance = CFG.Entry.PromptRange
    prompt.RequiresLineOfSight = false
    prompt.Parent = chest
    S.prompt = prompt
    
    prompt.Triggered:Connect(function()
        if S.piloting or S.entering or S.exiting then return end
        if not player.Character then return end
        task.spawn(enterMecha)
    end)
end

-- ============================================================================
-- ENTRADA NO MECHA (cinematográfica)
-- ============================================================================
function enterMecha()
    if S.entering or S.piloting then return end
    S.entering = true
    
    local char = player.Character
    if not char then S.entering = false; return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hum or not hrp then S.entering = false; return end
    
    S.prompt.Enabled = false
    
    -- Salvar estado do player
    S.savedHumanoid = {
        WalkSpeed = hum.WalkSpeed,
        JumpPower = hum.JumpPower,
        JumpHeight = hum.JumpHeight,
        AutoRotate = hum.AutoRotate,
        PlatformStand = hum.PlatformStand,
    }
    S.savedRootCF = hrp.CFrame
    
    -- Travar player
    hum.WalkSpeed = 0
    hum.JumpPower = 0
    hum.AutoRotate = false
    hum.PlatformStand = true
    
    -- Ancorar todas as partes do player
    for _, p in ipairs(char:GetDescendants()) do
        if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then
            p.Anchored = true
        end
    end
    
    -- Camera para o mecha
    camera.CameraSubject = S.root
    
    -- ===== FASE 1: PORTA ABRE =====
    local chest = S.joints.chest
    local dL = S.joints.doorL
    local dR = S.joints.doorR
    local dLJoint = dL:FindFirstChildOfClass("Motor6D") or dL:FindFirstChild("CockpitDoorL_Joint")
    local dRJoint = dR:FindFirstChildOfClass("Motor6D") or dR:FindFirstChild("CockpitDoorR_Joint")
    
    -- Portas deslizam lateralmente (aumentando X do C0)
    local openOffset = 3.5
    local closedC0L, closedC0R
    -- Encontrar o Motor6D que liga cada porta ao chest
    for _, w in ipairs(chest:GetChildren()) do
        if w:IsA("Motor6D") and w.Part1 == dL then closedC0L = w end
        if w:IsA("Motor6D") and w.Part1 == dR then closedC0R = w end
    end
    
    if closedC0L and closedC0R then
        local targetL = closedC0L.C0 * CFrame.new(-openOffset, 0, 0)
        local targetR = closedC0R.C0 * CFrame.new(openOffset, 0, 0)
        tween(closedC0L, { C0 = targetL }, CFG.Entry.DoorOpenTime, Enum.EasingStyle.Quad, Enum.EasingDirection.Out):Play()
        tween(closedC0R, { C0 = targetR }, CFG.Entry.DoorOpenTime, Enum.EasingStyle.Quad, Enum.EasingDirection.Out):Play()
    end
    
    task.wait(CFG.Entry.DoorOpenTime)
    
    -- ===== FASE 2: CABO SE ESTENDE =====
    local cable = S.cable
    local originPos = S.cableBase.WorldPosition
    local targetPos = hrp.Position
    
    cable.Transparency = 0
    cable.Anchored = true
    
    local delta = targetPos - originPos
    local dist = delta.Magnitude
    
    -- Cabo inicialmente no cockpit
    cable.Size = Vector3.new(0.35, 0.35, 0.1)
    cable.CFrame = CFrame.new(originPos, originPos + delta.Unit * 0.1)
    
    local extendTween = tween(cable, {
        Size = Vector3.new(0.35, 0.35, dist),
        CFrame = CFrame.new(originPos + delta * 0.5, targetPos),
    }, CFG.Entry.CableExtendTime, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    extendTween:Play()
    
    task.wait(CFG.Entry.CableExtendTime)
    
    -- ===== FASE 3: PLAYER É PUXADO =====
    local startCF = hrp.CFrame
    local seatWorld = S.cockpit.WorldCFrame
    local endCF = seatWorld * CFrame.new(0, -2.5, 0) * CFrame.Angles(0, math.rad(180), 0)
    
    -- Corrigir orientação: player deve olhar para frente do mecha (-Z do mecha)
    local mechaLook = S.root.CFrame.LookVector
    local playerFacing = CFrame.lookAt(seatWorld.Position, seatWorld.Position - mechaLook)
    endCF = playerFacing * CFrame.new(0, -1, -1)
    
    local t0 = tick()
    local dur = CFG.Entry.PullInTime
    local conn
    conn = RunService.RenderStepped:Connect(function()
        local a = math.clamp((tick() - t0) / dur, 0, 1)
        local ease = a * a * (3 - 2 * a)
        
        -- Mover HRP
        hrp.CFrame = startCF:Lerp(endCF, ease)
        
        -- Mover todas as partes do player junto (elas estão ancoradas)
        for _, p in ipairs(player.Character:GetDescendants()) do
            if p:IsA("BasePart") and p ~= hrp and p.Name ~= "HumanoidRootPart" then
                local rel = startCF:ToObjectSpace(p.CFrame)
                p.CFrame = hrp.CFrame * rel
            end
        end
        
        -- Cabo encurta junto
        local newDist = (1 - ease) * dist
        cable.Size = Vector3.new(0.35, 0.35, math.max(0.1, newDist))
        cable.CFrame = CFrame.new(originPos + (targetPos - originPos) * (1 - ease) * 0.5, originPos + (targetPos - originPos) * (1 - ease))
        
        if a >= 1 then conn:Disconnect() end
    end)
    
    task.wait(dur + 0.05)
    
    -- Esconder cabo
    cable.Transparency = 1
    cable.Anchored = false
    
    -- ===== FASE 4: PORTA FECHA =====
    if closedC0L and closedC0R then
        tween(closedC0L, { C0 = closedC0L.C0 * CFrame.new(openOffset, 0, 0) }, 0, Enum.EasingStyle.Linear):Play()
        tween(closedC0R, { C0 = closedC0R.C0 * CFrame.new(-openOffset, 0, 0) }, 0, Enum.EasingStyle.Linear):Play()
    end
    
    -- Usar os C0 originais (antes da abertura)
    if closedC0L then
        closedC0L.C0 = closedC0L.C0 * CFrame.new(openOffset, 0, 0)
        tween(closedC0L, { C0 = closedC0L.C0 * CFrame.new(-openOffset, 0, 0) }, CFG.Entry.DoorCloseTime):Play()
    end
    if closedC0R then
        closedC0R.C0 = closedC0R.C0 * CFrame.new(-openOffset, 0, 0)
        tween(closedC0R, { C0 = closedC0R.C0 * CFrame.new(openOffset, 0, 0) }, CFG.Entry.DoorCloseTime):Play()
    end
    
    -- Esconder avatar (mas manter HRP)
    for _, p in ipairs(player.Character:GetDescendants()) do
        if p:IsA("BasePart") then
            p.LocalTransparencyModifier = 1
        elseif p:IsA("Decal") then
            p.Transparency = 1
        end
    end
    
    task.wait(CFG.Entry.DoorCloseTime)
    
    -- ===== ENTRADA COMPLETA =====
    S.piloting = true
    S.entering = false
    S.prompt.Enabled = false
    S.lastStepIdx = 0
    
    -- Atualizar câmera
    camera.CameraSubject = S.root
    S.camFov = CFG.Camera.Fov.Piloting
    
    print("[Mecha] Pilotagem ativa. Use o joystick / WASD para mover.")
end

-- ============================================================================
-- SAÍDA DO MECHA
-- ============================================================================
function exitMecha()
    if not S.piloting or S.exiting then return end
    S.exiting = true
    
    local char = player.Character
    if not char then S.exiting = false; return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hum or not hrp then S.exiting = false; return end
    
    -- Parar movimento
    S.speed = 0
    S.boosting = false
    
    -- Abrir porta
    local chest = S.joints.chest
    local dL = S.joints.doorL
    local dR = S.joints.doorR
    local openOffset = 3.5
    local cL, cR
    for _, w in ipairs(chest:GetChildren()) do
        if w:IsA("Motor6D") and w.Part1 == dL then cL = w end
        if w:IsA("Motor6D") and w.Part1 == dR then cR = w end
    end
    
    if cL then tween(cL, { C0 = cL.C0 * CFrame.new(-openOffset, 0, 0) }, 0.7):Play() end
    if cR then tween(cR, { C0 = cR.C0 * CFrame.new(openOffset, 0, 0) }, 0.7):Play() end
    
    task.wait(0.7)
    
    -- Desancorar player e posicionar à frente
    local mechaCF = S.root.CFrame
    local dropPos = mechaCF * CFrame.new(0, 2, -12)
    
    hrp.CFrame = dropPos
    for _, p in ipairs(char:GetDescendants()) do
        if p:IsA("BasePart") then
            p.LocalTransparencyModifier = 0
            if p.Name ~= "HumanoidRootPart" then
                p.Anchored = false
            end
        elseif p:IsA("Decal") then
            p.Transparency = 0
        end
    end
    hrp.Anchored = false
    
    -- Restaurar Humanoid
    if S.savedHumanoid then
        hum.WalkSpeed = S.savedHumanoid.WalkSpeed
        hum.JumpPower = S.savedHumanoid.JumpPower
        hum.JumpHeight = S.savedHumanoid.JumpHeight
        hum.AutoRotate = S.savedHumanoid.AutoRotate
        hum.PlatformStand = false
    end
    
    -- Câmera volta para o player
    camera.CameraSubject = hum
    S.camFov = CFG.Camera.Fov.Humanoid
    
    -- Fechar porta
    task.wait(0.3)
    if cL then tween(cL, { C0 = cL.C0 * CFrame.new(openOffset, 0, 0) }, 0.7):Play() end
    if cR then tween(cR, { C0 = cR.C0 * CFrame.new(-openOffset, 0, 0) }, 0.7):Play() end
    
    S.piloting = false
    S.exiting = false
    S.prompt.Enabled = true
    
    print("[Mecha] Você saiu do mecha.")
end

-- ============================================================================
-- CONTROLE DE MOVIMENTO (durante pilotagem)
-- ============================================================================
local function updatePilot(dt)
    if not S.piloting then return end
    if not S.root or not S.root.Parent then return end
    
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    
    -- Turbo
    if S.boosting then
        S.energy = math.max(0, S.energy - CFG.Boost.Drain * dt)
        if S.energy <= 0 then S.boosting = false; S.lastBoostEnd = tick() end
    elseif S.energy < CFG.Boost.Max and tick() - S.lastBoostEnd > CFG.Boost.RegenDelay then
        S.energy = math.min(CFG.Boost.Max, S.energy + CFG.Boost.Regen * dt)
    end
    
    local targetSpeed = S.boosting and CFG.Move.RunSpeed or CFG.Move.WalkSpeed
    local moveDir = Vector3.zero
    if hum then
        moveDir = hum.MoveDirection
    end
    
    local wantsMove = moveDir.Magnitude > 0.1
    if wantsMove then
        S.speed = S.speed + (targetSpeed - S.speed) * math.clamp(dt * 4, 0, 1)
    else
        S.speed = S.speed + (0 - S.speed) * math.clamp(dt * 6, 0, 1)
    end
    
    -- Rotação suave na direção do movimento
    if wantsMove then
        local currentLook = S.root.CFrame.LookVector
        local targetYaw = math.atan2(moveDir.X, moveDir.Z)
        local currentYaw = math.atan2(currentLook.X, currentLook.Z)
        local diff = (targetYaw - currentYaw + math.pi) % (2 * math.pi) - math.pi
        local step = math.clamp(diff, -CFG.Move.TurnRate * dt, CFG.Move.TurnRate * dt)
        S.root.CFrame = CFrame.new(S.root.Position) * CFrame.Angles(0, currentYaw + step, 0)
    end
    
    -- Mover
    if S.speed > 0.5 then
        local look = S.root.CFrame.LookVector
        local newPos = S.root.Position + look * S.speed * dt
        
        -- Verificar chão
        local ray = RaycastParams.new()
        ray.FilterType = Enum.RaycastFilterType.Exclude
        ray.FilterDescendantsInstances = { S.mecha, player.Character }
        local hit = workspace:Raycast(newPos + Vector3.new(0, 8, 0), Vector3.new(0, -20, 0), ray)
        if hit then
            newPos = Vector3.new(newPos.X, hit.Position.Y + 0.5, newPos.Z)
        end
        S.root.CFrame = CFrame.new(newPos) * CFrame.Angles(0, select(2, S.root.CFrame:ToOrientation()), 0)
    end
    
    -- Animação de caminhada (pernas)
    if S.speed > 1 then
        S.bobPhase = S.bobPhase + S.speed * dt * 0.25
        local stepIdx = math.floor(S.bobPhase / math.pi)
        if stepIdx ~= S.lastStepIdx then
            S.lastStepIdx = stepIdx
            -- Passo: pequeno tremor na câmera
            local shakeAmp = 0.25
            camera.CFrame = camera.CFrame * CFrame.new(
                (math.random() - 0.5) * shakeAmp,
                (math.random() - 0.5) * shakeAmp,
                0
            )
        end
        
        -- Rotacionar coxas e joelhos
        local phase = S.bobPhase
        local swing = math.sin(phase) * 0.45
        local swing2 = math.sin(phase + math.pi) * 0.45
        
        local joints = S.joints
        for _, p in ipairs(S.mecha:GetDescendants()) do
            if p:IsA("BasePart") then
                if p.Name == "LeftUpperLeg" then
                    local j = p:FindFirstChildOfClass("Motor6D")
                    if j then j.Transform = CFrame.Angles(swing, 0, 0) end
                elseif p.Name == "RightUpperLeg" then
                    local j = p:FindFirstChildOfClass("Motor6D")
                    if j then j.Transform = CFrame.Angles(swing2, 0, 0) end
                elseif p.Name == "LeftKnee" then
                    local j = p:FindFirstChildOfClass("Motor6D")
                    if j then j.Transform = CFrame.Angles(math.max(0, -math.sin(phase)) * 0.5, 0, 0) end
                elseif p.Name == "RightKnee" then
                    local j = p:FindFirstChildOfClass("Motor6D")
                    if j then j.Transform = CFrame.Angles(math.max(0, -math.sin(phase + math.pi)) * 0.5, 0, 0) end
                end
            end
        end
    else
        -- Parar animação (reset)
        for _, p in ipairs(S.mecha:GetDescendants()) do
            if p:IsA("BasePart") and (p.Name:find("UpperLeg") or p.Name:find("Knee")) then
                local j = p:FindFirstChildOfClass("Motor6D")
                if j then j.Transform = j.Transform:Lerp(CFrame.new(), 0.15) end
            end
        end
    end
    
    -- Posicionar player dentro do cockpit
    if hrp and S.cockpit then
        local seatCF = S.cockpit.WorldCFrame
        hrp.CFrame = seatCF * CFrame.new(0, -1, -0.5)
    end
    
    -- Câmera
    if camera.CameraSubject ~= S.root then
        camera.CameraSubject = S.root
    end
end

-- ============================================================================
-- CÂMERA (FOV dinâmico)
-- ============================================================================
local function updateCamera(dt)
    if not camera then camera = workspace.CurrentCamera end
    if not camera then return end
    
    local targetFov = S.piloting and (S.boosting and CFG.Camera.Fov.Boost or CFG.Camera.Fov.Piloting)
        or CFG.Camera.Fov.Humanoid
    
    S.camFov = S.camFov + (targetFov - S.camFov) * math.clamp(dt * 4, 0, 1)
    camera.FieldOfView = S.camFov
end

-- ============================================================================
-- UI MOBILE
-- ============================================================================
local function buildUI()
    local old = playerGui:FindFirstChild("MechaPilotUI")
    if old then old:Destroy() end
    
    local gui = Instance.new("ScreenGui")
    gui.Name = "MechaPilotUI"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 50
    gui.Parent = playerGui
    
    -- Botão SAIR
    local exitBtn = Instance.new("TextButton")
    exitBtn.Name = "ExitBtn"
    exitBtn.AnchorPoint = Vector2.new(1, 1)
    exitBtn.Position = UDim2.new(1, -16, 1, -160)
    exitBtn.Size = UDim2.new(0, 90, 0, 60)
    exitBtn.BackgroundColor3 = Color3.fromRGB(160, 45, 45)
    exitBtn.BackgroundTransparency = 0.15
    exitBtn.Text = "SAIR"
    exitBtn.TextColor3 = Color3.new(1, 1, 1)
    exitBtn.Font = Enum.Font.GothamBlack
    exitBtn.TextSize = 18
    exitBtn.Visible = false
    exitBtn.Parent = gui
    local c1 = Instance.new("UICorner"); c1.CornerRadius = UDim.new(0, 12); c1.Parent = exitBtn
    local s1 = Instance.new("UIStroke"); s1.Color = Color3.new(1, 1, 1); s1.Transparency = 0.4; s1.Parent = exitBtn
    
    -- Botão TURBO
    local boostBtn = Instance.new("TextButton")
    boostBtn.Name = "BoostBtn"
    boostBtn.AnchorPoint = Vector2.new(1, 1)
    boostBtn.Position = UDim2.new(1, -16, 1, -240)
    boostBtn.Size = UDim2.new(0, 90, 0, 60)
    boostBtn.BackgroundColor3 = Color3.fromRGB(220, 160, 40)
    boostBtn.BackgroundTransparency = 0.15
    boostBtn.Text = "TURBO"
    boostBtn.TextColor3 = Color3.new(1, 1, 1)
    boostBtn.Font = Enum.Font.GothamBlack
    boostBtn.TextSize = 16
    boostBtn.Visible = false
    boostBtn.Parent = gui
    local c2 = Instance.new("UICorner"); c2.CornerRadius = UDim.new(0, 12); c2.Parent = boostBtn
    local s2 = Instance.new("UIStroke"); s2.Color = Color3.new(1, 1, 1); s2.Transparency = 0.4; s2.Parent = boostBtn
    
    -- Barra de energia
    local energyBg = Instance.new("Frame")
    energyBg.Name = "EnergyBg"
    energyBg.AnchorPoint = Vector2.new(0.5, 0)
    energyBg.Position = UDim2.new(0.5, 0, 0, 14)
    energyBg.Size = UDim2.new(0, 220, 0, 10)
    energyBg.BackgroundColor3 = Color3.fromRGB(25, 30, 40)
    energyBg.BorderSizePixel = 0
    energyBg.Visible = false
    energyBg.Parent = gui
    local c3 = Instance.new("UICorner"); c3.CornerRadius = UDim.new(1, 0); c3.Parent = energyBg
    
    local energyFill = Instance.new("Frame")
    energyFill.Name = "Fill"
    energyFill.Size = UDim2.new(1, 0, 1, 0)
    energyFill.BackgroundColor3 = Color3.fromRGB(220, 160, 40)
    energyFill.BorderSizePixel = 0
    energyFill.Parent = energyBg
    local c4 = Instance.new("UICorner"); c4.CornerRadius = UDim.new(1, 0); c4.Parent = energyFill
    
    -- Label de status
    local label = Instance.new("TextLabel")
    label.Name = "Status"
    label.AnchorPoint = Vector2.new(0.5, 0)
    label.Position = UDim2.new(0.5, 0, 0, 30)
    label.Size = UDim2.new(0, 300, 0, 20)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(80, 220, 255)
    label.Font = Enum.Font.GothamBold
    label.TextSize = 12
    label.Text = ""
    label.Parent = gui
    
    exitBtn.Activated:Connect(function()
        if S.piloting and not S.exiting then task.spawn(exitMecha) end
    end)
    
    local boostInput
    boostBtn.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
            boostInput = i
            if S.piloting and S.energy >= CFG.Boost.MinToStart then S.boosting = true end
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if boostInput and i == boostInput then
            boostInput = nil
            S.boosting = false
            S.lastBoostEnd = tick()
        end
    end)
    
    S.ui = {
        exit = exitBtn,
        boost = boostBtn,
        energyBg = energyBg,
        energyFill = energyFill,
        label = label,
    }
end

local function updateUI()
    if not S.ui then return end
    local visible = S.piloting
    S.ui.exit.Visible = visible
    S.ui.boost.Visible = visible
    S.ui.energyBg.Visible = visible
    
    if visible then
        S.ui.energyFill.Size = UDim2.new(S.energy / CFG.Boost.Max, 0, 1, 0)
        S.ui.energyFill.BackgroundColor3 = S.energy < CFG.Boost.MinToStart
            and Color3.fromRGB(200, 60, 60)
            or Color3.fromRGB(220, 160, 40)
        S.ui.label.Text = S.boosting and "MODO TURBO ATIVO" or "UNIDADE MILITAR-01 // ONLINE"
    end
end

-- ============================================================================
-- LOOP PRINCIPAL
-- ============================================================================
RunService.RenderStepped:Connect(function(dt)
    dt = math.min(dt, 0.1)
    updatePilot(dt)
    updateCamera(dt)
    updateUI()
end)

-- ============================================================================
-- INPUT (PC)
-- ============================================================================
ContextActionService:BindAction("MechaExit", function(_, state)
    if state == Enum.UserInputState.Begin and S.piloting and not S.exiting then
        task.spawn(exitMecha)
    end
    return Enum.ContextActionResult.Pass
end, false, Enum.KeyCode.Y)

ContextActionService:BindAction("MechaBoost", function(_, state)
    if not S.piloting then return Enum.ContextActionResult.Pass end
    if state == Enum.UserInputState.Begin and S.energy >= CFG.Boost.MinToStart then
        S.boosting = true
    elseif state == Enum.UserInputState.End or state == Enum.UserInputState.Cancel then
        S.boosting = false
        S.lastBoostEnd = tick()
    end
    return Enum.ContextActionResult.Pass
end, false, Enum.KeyCode.LeftShift, Enum.KeyCode.RightShift)

-- ============================================================================
-- INICIALIZAÇÃO
-- ============================================================================
local function initialize()
    -- Aguardar player
    if not player.Character then player.CharacterAdded:Wait() end
    local char = player.Character
    if not char then return end
    char:WaitForChild("Humanoid", 10)
    local hrp = char:WaitForChild("HumanoidRootPart", 10)
    if not hrp then return end
    
    -- Posicionar mecha à frente do player
    local spawnPos = hrp.Position + hrp.CFrame.LookVector * 25
    -- Colocar no chão
    local ray = RaycastParams.new()
    ray.FilterType = Enum.RaycastFilterType.Exclude
    ray.FilterDescendantsInstances = { char }
    local hit = workspace:Raycast(spawnPos + Vector3.new(0, 30, 0), Vector3.new(0, -60, 0), ray)
    if hit then
        spawnPos = Vector3.new(spawnPos.X, hit.Position.Y + 1, spawnPos.Z)
    end
    
    -- Orientação: de frente para o player
    local facing = math.atan2(hrp.Position.X - spawnPos.X, hrp.Position.Z - spawnPos.Z)
    
    buildMecha(spawnPos, facing)
    attachPrompt(S.mecha)
    buildUI()
    
    print("[Mecha] Unidade Militar-01 posicionada. Aproxime-se para entrar.")
end

task.spawn(initialize)

-- Reconstruir se o player morrer
player.CharacterAdded:Connect(function()
    if S.piloting then
        S.piloting = false
        S.entering = false
        S.exiting = false
    end
    task.wait(1)
    task.spawn(initialize)
end)
