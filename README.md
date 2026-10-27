--[[
ASTAR-MENU | v1.0 | @astar.mmgh1000
Tema: roxo neon, azul escuro e preto. Interface com mouse e toque.

LEIA ANTES DE USAR
Este arquivo precisa executar no contexto CLIENTE do Roblox. Em um projeto
editavel no Studio: crie um LocalScript em StarterPlayer > StarterPlayerScripts,
cole o arquivo e inicie Play. Um arquivo .lua nao se instala sozinho no app Roblox.
O menu foi solicitado para Counter Blox, mas nao foi testado numa partida desse
jogo. Personagens, camera, movimento e dados replicados podem variar por jogo.
Nenhum carregador externo, executor ou bypass esta incluido.

CONTROLES
RightShift ou botao A: abrir/fechar. Arraste o titulo para mover o painel.
Aimbot: habilite Liberar aimbot, escolha Safe/Full e feche o painel. Segure o
botao direito do mouse ou AIM no celular. Safe significa suavizacao, nao uma
garantia contra deteccao. Aothers permite ativacao continua e filtros.
Fly: WASD + Espaco/Ctrl no PC; joystick + SUBIR/DESCER no celular.
Configs: cores #RRGGBB, exportacao/importacao manual de JSON e descarregamento.

LIMITES EXPLICITOS
SkinChanger consulta assets replicados e faz PREVIA LOCAL. Nao concede skins,
nao escreve no inventario oficial e nao equipa itens no servidor. A pesquisa
publica nao forneceu um catalogo oficial completo. A deteccao em tempo de
execucao nao garante encontrar todas as skins: schemas privados nao sao adivinhados.
Colete/arma aparecem como N/D quando os valores nao estao disponiveis ao cliente.
Movimento e camera podem ser corrigidos pelos scripts do jogo ou pelo servidor.

DOCUMENTACAO CONSULTADA EM 2026-10-27
https://create.roblox.com/docs/reference/engine/classes/Camera
https://create.roblox.com/docs/reference/engine/classes/Highlight
https://create.roblox.com/docs/reference/engine/classes/RunService
https://create.roblox.com/docs/reference/engine/classes/LinearVelocity
https://create.roblox.com/docs/reference/engine/classes/ColorCorrectionEffect
https://create.roblox.com/docs/scripting/security/client-server-boundary
https://www.roblox.com/games/301549746/Counter-Blox
]]

local S = {
    Players = game:GetService("Players"),
    Run = game:GetService("RunService"),
    Input = game:GetService("UserInputService"),
    Storage = game:GetService("ReplicatedStorage"),
    Lighting = game:GetService("Lighting"),
    Http = game:GetService("HttpService"),
}
local player = S.Players.LocalPlayer
if not player or not S.Run:IsClient() then
    warn("Astar-Menu: execute este arquivo em um contexto cliente (LocalScript).")
    return
end
local playerGui = player:WaitForChild("PlayerGui", 15)
if not playerGui then
    warn("Astar-Menu: PlayerGui indisponivel.")
    return
end
local previous = playerGui:FindFirstChild("AstarMenu_UI")
if previous then
    local stop = previous:FindFirstChild("Shutdown")
    if stop and stop:IsA("BindableEvent") then stop:Fire() end
    if previous.Parent then previous:Destroy() end
end

local defaults = {
    AimMaster = false, AimMode = "Safe", AimFov = true, AimRadius = 120,
    AimStrength = 8, AimAlways = false, TeamCheck = true, WallCheck = true,
    EspMaster = false, EspBox = false, EspLine = false, EspName = false,
    EspHealth = false, EspArmor = false, EspWeapon = false, Chams = false,
    EspAllies = false, MaxDistance = 2000, ChamsOpacity = 0.40,
    Fly = false, FlySpeed = 50, Noclip = false,
    Speed = false, WalkSpeed = 24, HighJump = false, JumpPower = 65,
    AutoJump = false, Fullbright = false, NoFog = false,
    CameraFov = false, CameraFovValue = 90, Crosshair = false,
    Zoom = false, ZoomDistance = 30, ShowStats = true,
    Tint = false, TintColor = "FFFFFF", TintSaturation = 0,
    AccentColor = "B75CFF", BackgroundColor = "080A12", CardColor = "101A32",
    EspColor = "B75CFF", ChamsColor = "9E55FF", FovColor = "AE73FF",
}
local bounds = {
    AimRadius = {20, 600, 1}, AimStrength = {1, 25, 0.5},
    MaxDistance = {50, 5000, 50}, ChamsOpacity = {0, 1, 0.05},
    FlySpeed = {5, 150, 1}, WalkSpeed = {8, 80, 1},
    JumpPower = {20, 120, 1}, CameraFovValue = {40, 110, 1},
    ZoomDistance = {5, 80, 1}, TintSaturation = {-1, 1, 0.05},
}
local state = {}
for key, value in pairs(defaults) do state[key] = value end
local A = {
    alive = true, connections = {}, refreshers = {}, theme = {}, pages = {}, tabs = {},
    held = {}, touches = {}, ledger = {}, records = {}, activePage = "Aimbot",
    menuOpen = true, aimHeld = false, flying = nil, drag = nil, lastError = -math.huge,
    frameCount = 0, frameTime = 0, fps = 0, statTime = 0, tickTime = 0,
    skinToken = 0, skinItems = {}, skinPage = 1, skinQuery = "", scanBusy = false,
}

local function clamp(x, lo, hi) return math.max(lo, math.min(hi, x)) end
local function finite(x) return type(x) == "number" and x == x and math.abs(x) < math.huge end
local function validHex(v)
    if type(v) ~= "string" then return nil end
    v = v:gsub("^#", ""):upper()
    return (#v == 6 and v:match("^%x+$")) and v or nil
end
local function color(v)
    v = validHex(v) or "B75CFF"
    return Color3.fromRGB(tonumber(v:sub(1, 2), 16), tonumber(v:sub(3, 4), 16), tonumber(v:sub(5, 6), 16))
end
local function make(kind, props, parent)
    local object = Instance.new(kind)
    for k, v in pairs(props or {}) do object[k] = v end
    object.Parent = parent
    return object
end
local function connect(signal, callback)
    local c = signal:Connect(callback)
    table.insert(A.connections, c)
    return c
end
local function rounded(object, pixels)
    make("UICorner", {CornerRadius = UDim.new(0, pixels or 10)}, object)
end
local function themed(object, prop, key)
    object[prop] = color(state[key])
    table.insert(A.theme, {object, prop, key})
end
local function refreshTheme()
    for _, item in ipairs(A.theme) do
        if item[1].Parent then item[1][item[2]] = color(state[item[3]]) end
    end
end
local function label(parent, text, props)
    local p = {
        Text = text, BackgroundTransparency = 1, BorderSizePixel = 0,
        TextColor3 = Color3.fromRGB(231, 232, 245), Font = Enum.Font.Gotham,
        TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, TextTruncate = Enum.TextTruncate.AtEnd,
        Size = UDim2.new(1, 0, 0, 24), ZIndex = 12,
    }
    for k, v in pairs(props or {}) do p[k] = v end
    return make("TextLabel", p, parent)
end
local function press(parent, text, props)
    local p = {
        Text = text, BorderSizePixel = 0, BackgroundColor3 = color(state.CardColor),
        TextColor3 = Color3.fromRGB(242, 240, 255), Font = Enum.Font.GothamMedium,
        TextSize = 12, AutoButtonColor = true, Size = UDim2.new(1, 0, 0, 38), ZIndex = 14,
    }
    for k, v in pairs(props or {}) do p[k] = v end
    local b = make("TextButton", p, parent)
    rounded(b, 8)
    return b
end
local function textInput(parent, props)
    local p = {
        BackgroundColor3 = Color3.fromRGB(6, 9, 18), BorderSizePixel = 0,
        TextColor3 = Color3.fromRGB(236, 237, 255), Font = Enum.Font.Code,
        TextSize = 12, ClearTextOnFocus = false, Text = "", ZIndex = 15,
    }
    for k, v in pairs(props or {}) do p[k] = v end
    local field = make("TextBox", p, parent)
    rounded(field, 6)
    return field
end
local function cornerStroke(object, thickness)
    local stroke = make("UIStroke", {Thickness = thickness or 1, Transparency = 0.30}, object)
    themed(stroke, "Color", "AccentColor")
    return stroke
end

-- A journal restores only values this script still owns. Server/game updates
-- observed while an override is active become the new restoration baseline.
local function override(feature, object, prop, value)
    if not object then return end
    local ok, old = pcall(function() return object[prop] end)
    if not ok then return end
    local journal = A.ledger[feature]
    if not journal then journal = {}; A.ledger[feature] = journal end
    local entry = journal[object]
    if not entry then entry = {}; journal[object] = entry end
    local record = entry[prop]
    if not record then
        record = {before = old, last = old}
        entry[prop] = record
    elseif old ~= record.last then
        record.before = old
    end
    if pcall(function() object[prop] = value end) then record.last = value end
end
local function restore(feature)
    local journal = A.ledger[feature]
    if not journal then return end
    for object, entry in pairs(journal) do
        for prop, record in pairs(entry) do
            pcall(function()
                if object[prop] == record.last then object[prop] = record.before end
            end)
        end
    end
    A.ledger[feature] = nil
end
local function restoreAll()
    local keys = {}
    for key in pairs(A.ledger) do table.insert(keys, key) end
    for _, key in ipairs(keys) do restore(key) end
end
local function stopFly()
    if A.flying then
        for _, obj in ipairs(A.flying.objects) do obj:Destroy() end
        A.flying = nil
    end
    restore("flight")
end
local function cleanPreview()
    A.previewModel = nil
    if A.previewWorld then A.previewWorld:ClearAllChildren() end
    if A.texturePreview then A.texturePreview.Image = ""; A.texturePreview.Visible = false end
end

local gui = make("ScreenGui", {
    Name = "AstarMenu_UI", ResetOnSpawn = false, IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling, DisplayOrder = 120,
}, playerGui)
local shutdown = make("BindableEvent", {Name = "Shutdown"}, gui)
local effects = make("Folder", {Name = "AstarMenuEffects_" .. tostring(player.UserId)}, workspace)
local tint = make("ColorCorrectionEffect", {Name = "AstarMenuTint", Enabled = false}, S.Lighting)

local panel = make("Frame", {
    Name = "Panel", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(740, 550), BorderSizePixel = 0, ZIndex = 10,
    Active = true, ClipsDescendants = true,
}, gui)
themed(panel, "BackgroundColor3", "BackgroundColor")
rounded(panel, 14); cornerStroke(panel, 1.6)
local top = make("Frame", {Size = UDim2.new(1, 0, 0, 64), BackgroundTransparency = 1, Active = true, ZIndex = 11}, panel)
local monogram = press(top, "A", {Position = UDim2.fromOffset(14, 13), Size = UDim2.fromOffset(38, 38), TextSize = 24, Font = Enum.Font.GothamBold})
themed(monogram, "BackgroundColor3", "AccentColor")
label(top, "Astar-Menu", {Position = UDim2.fromOffset(64, 10), Size = UDim2.new(1, -155, 0, 27), TextSize = 22, Font = Enum.Font.GothamBold})
label(top, "COUNTER BLOX  /  @astar.mmgh1000", {Position = UDim2.fromOffset(65, 36), Size = UDim2.new(1, -154, 0, 16), TextSize = 9, TextColor3 = Color3.fromRGB(155, 162, 190)})
local minimize = press(top, "—", {Position = UDim2.new(1, -52, 0, 14), Size = UDim2.fromOffset(36, 34), TextSize = 20})
local tabsBar = make("ScrollingFrame", {
    Position = UDim2.fromOffset(14, 66), Size = UDim2.new(1, -28, 0, 42),
    CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.X,
    ScrollingDirection = Enum.ScrollingDirection.X, ScrollBarThickness = 2,
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 12,
}, panel)
make("UIListLayout", {FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder}, tabsBar)
local footer = label(panel, "Tudo desligado ao iniciar  •  RightShift / A abre o menu", {
    Position = UDim2.new(0, 16, 1, -28), Size = UDim2.new(1, -32, 0, 20),
    TextSize = 10, TextColor3 = Color3.fromRGB(160, 165, 187),
})
local bubble = press(gui, "A", {
    Name = "OpenMenu", Position = UDim2.fromOffset(14, 80), Size = UDim2.fromOffset(44, 44),
    TextSize = 23, Font = Enum.Font.GothamBold, ZIndex = 30, Visible = false,
})
themed(bubble, "BackgroundColor3", "AccentColor"); cornerStroke(bubble, 1)
local toast = label(gui, "", {
    Position = UDim2.new(0.5, 0, 0, 36), AnchorPoint = Vector2.new(0.5, 0),
    Size = UDim2.new(0.9, 0, 0, 54), BackgroundTransparency = 0.05,
    BackgroundColor3 = Color3.fromRGB(13, 18, 33), TextWrapped = true,
    TextTruncate = Enum.TextTruncate.None, TextXAlignment = Enum.TextXAlignment.Center,
    Visible = false, ZIndex = 50, TextSize = 12,
})
rounded(toast, 8)
local function notify(message)
    A.toastUntil = os.clock() + 5
    toast.Text = message
    toast.Visible = true
end
local function refresh()
    refreshTheme()
    for _, fn in ipairs(A.refreshers) do fn() end
end
local function set(key, value)
    local base = defaults[key]
    if base == nil then return false end
    if type(base) == "boolean" then
        if type(value) ~= "boolean" then return false end
    elseif type(base) == "number" then
        if not finite(value) then return false end
        local b = bounds[key]
        value = clamp(value, b[1], b[2])
        value = math.floor(value / b[3] + 0.5) * b[3]
    elseif key == "AimMode" then
        if value ~= "Safe" and value ~= "Full" then return false end
    else
        value = validHex(value)
        if not value then return false end
    end
    state[key] = value
    if key == "Fly" and not value then stopFly() end
    local feature = ({Noclip = "noclip", Speed = "speed", HighJump = "jump", Fullbright = "light", NoFog = "fog", CameraFov = "fov", Zoom = "zoom"})[key]
    if feature and not value then restore(feature) end
    if key == "AimMaster" and not value then A.aimHeld = false end
    refresh()
    return true
end
local function setOpen(open)
    A.menuOpen = open
    panel.Visible = open
    bubble.Visible = not open
    A.aimHeld = false
    A.drag = nil
    for _, page in pairs(A.pages) do page.ScrollingEnabled = true end
    if open then A.held = {}; A.touches = {} end
end
connect(minimize.Activated, function() setOpen(false) end)
connect(bubble.Activated, function() setOpen(true) end)

local function resize()
    local camera = workspace.CurrentCamera
    if not camera then return end
    local v = camera.ViewportSize
    if v.X < 1 or v.Y < 1 then return end
    local w, h = math.min(760, math.max(260, v.X - 24)), math.min(570, math.max(210, v.Y - 72))
    panel.Size = UDim2.fromOffset(w, h)
    panel.Position = UDim2.fromOffset(v.X / 2, v.Y / 2)
    A.viewport = v
end
resize()

local tabNames = {"Aimbot", "esp", "SkinChanger", "main", "aothers", "configs", "credts"}
for index, name in ipairs(tabNames) do
    local page = make("ScrollingFrame", {
        Name = name, Position = UDim2.fromOffset(14, 118), Size = UDim2.new(1, -28, 1, -154),
        BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
        CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ScrollingDirection = Enum.ScrollingDirection.Y, Visible = index == 1, ZIndex = 12,
    }, panel)
    make("UIPadding", {PaddingRight = UDim.new(0, 6), PaddingBottom = UDim.new(0, 8)}, page)
    make("UIListLayout", {Padding = UDim.new(0, 7), SortOrder = Enum.SortOrder.LayoutOrder}, page)
    local tab = press(tabsBar, name, {Size = UDim2.fromOffset(name == "SkinChanger" and 112 or 84, 34), LayoutOrder = index})
    A.pages[name] = page; A.tabs[name] = tab
    connect(tab.Activated, function()
        A.activePage = name
        for key, p in pairs(A.pages) do p.Visible = key == name end
        refresh()
    end)
end
table.insert(A.refreshers, function()
    for name, tab in pairs(A.tabs) do
        tab.BackgroundColor3 = color(name == A.activePage and state.AccentColor or state.CardColor)
    end
end)

local orders = {}
local function row(pageName, height)
    local page = A.pages[pageName]
    orders[pageName] = (orders[pageName] or 0) + 1
    local r = make("Frame", {Size = UDim2.new(1, 0, 0, height or 62), BorderSizePixel = 0, LayoutOrder = orders[pageName], ZIndex = 12}, page)
    rounded(r, 9); themed(r, "BackgroundColor3", "CardColor")
    return r
end
local function section(page, title, detail, height)
    local r = row(page, height or 65)
    label(r, title, {Position = UDim2.fromOffset(12, 7), Size = UDim2.new(1, -24, 0, 22), TextSize = 13, Font = Enum.Font.GothamBold})
    label(r, detail, {Position = UDim2.fromOffset(12, 31), Size = UDim2.new(1, -24, 1, -36), TextSize = 11, TextWrapped = true, TextTruncate = Enum.TextTruncate.None, TextYAlignment = Enum.TextYAlignment.Top, TextColor3 = Color3.fromRGB(167, 176, 201)})
    return r
end
local function toggle(page, title, description, key, mode)
    local r = row(page, 68)
    label(r, title, {Position = UDim2.fromOffset(12, 6), Size = UDim2.new(1, -92, 0, 23), Font = Enum.Font.GothamMedium})
    label(r, description, {Position = UDim2.fromOffset(12, 30), Size = UDim2.new(1, -92, 0, 31), TextSize = 10, TextWrapped = true, TextTruncate = Enum.TextTruncate.None, TextYAlignment = Enum.TextYAlignment.Top, TextColor3 = Color3.fromRGB(161, 170, 196)})
    local button = press(r, "OFF", {Position = UDim2.new(1, -68, 0, 19), Size = UDim2.fromOffset(54, 30), TextSize = 10})
    local function paint()
        local active = mode and state[key] == mode or (not mode and state[key])
        button.Text = active and "ON" or "OFF"
        button.BackgroundColor3 = active and color(state.AccentColor) or Color3.fromRGB(31, 38, 57)
    end
    table.insert(A.refreshers, paint)
    connect(button.Activated, function() set(key, mode or not state[key]) end)
    paint()
end
local function slider(page, title, key, description)
    local b = bounds[key]
    local r = row(page, 91)
    label(r, title, {Position = UDim2.fromOffset(12, 5), Size = UDim2.new(1, -24, 0, 22), Font = Enum.Font.GothamMedium})
    label(r, description or "", {Position = UDim2.fromOffset(12, 28), Size = UDim2.new(1, -24, 0, 17), TextSize = 10, TextColor3 = Color3.fromRGB(161, 170, 196)})
    local track = make("TextButton", {Text = "", AutoButtonColor = false, Position = UDim2.fromOffset(13, 52), Size = UDim2.new(1, -94, 0, 28), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 14}, r)
    local rail = make("Frame", {Position = UDim2.new(0, 0, 0.5, -3), Size = UDim2.new(1, 0, 0, 6), BorderSizePixel = 0, BackgroundColor3 = Color3.fromRGB(35, 45, 68), ZIndex = 14}, track)
    rounded(rail, 3)
    local fill = make("Frame", {Size = UDim2.new(0, 0, 1, 0), BorderSizePixel = 0, ZIndex = 15}, rail)
    themed(fill, "BackgroundColor3", "AccentColor"); rounded(fill, 3)
    local dot = make("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 0, 0.5, 0), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 16}, rail)
    rounded(dot, 7)
    local input = textInput(r, {Position = UDim2.new(1, -68, 0, 52), Size = UDim2.fromOffset(55, 28)})
    local function paint()
        local f = (state[key] - b[1]) / (b[2] - b[1])
        fill.Size = UDim2.new(f, 0, 1, 0); dot.Position = UDim2.new(f, 0, 0.5, 0)
        if not input:IsFocused() then input.Text = string.format(b[3] < 1 and "%.2f" or "%.0f", state[key]) end
    end
    local function move(position)
        local f = clamp((position.X - track.AbsolutePosition.X) / math.max(1, track.AbsoluteSize.X), 0, 1)
        set(key, b[1] + f * (b[2] - b[1]))
    end
    connect(track.InputBegan, function(inputObject)
        if inputObject.UserInputType == Enum.UserInputType.MouseButton1 or inputObject.UserInputType == Enum.UserInputType.Touch then
            A.pages[page].ScrollingEnabled = false
            A.drag = {input = inputObject, kind = "slider", move = move, page = A.pages[page]}
            move(inputObject.Position)
        end
    end)
    connect(input.FocusLost, function()
        if not set(key, tonumber(input.Text)) then notify("Valor invalido: " .. title) end
        paint()
    end)
    table.insert(A.refreshers, paint); paint()
end
local function colorField(page, title, key)
    local r = row(page, 62)
    label(r, title, {Position = UDim2.fromOffset(12, 6), Size = UDim2.new(1, -24, 0, 19), TextSize = 12})
    local swatch = make("Frame", {Position = UDim2.fromOffset(13, 31), Size = UDim2.fromOffset(20, 20), BorderSizePixel = 0, ZIndex = 14}, r)
    rounded(swatch, 5)
    local input = textInput(r, {Position = UDim2.fromOffset(43, 28), Size = UDim2.new(1, -56, 0, 27)})
    local function paint() swatch.BackgroundColor3 = color(state[key]); if not input:IsFocused() then input.Text = "#" .. state[key] end end
    connect(input.FocusLost, function()
        if not set(key, input.Text) then notify("Use seis digitos hexadecimais, por exemplo #B75CFF.") end
        paint()
    end)
    table.insert(A.refreshers, paint); paint()
end
local function action(page, title, callback)
    local r = row(page, 45)
    local b = press(r, title, {Position = UDim2.fromOffset(5, 4), Size = UDim2.new(1, -10, 1, -8)})
    themed(b, "BackgroundColor3", "CardColor")
    connect(b.Activated, callback)
    return b
end

section("Aimbot", "CONTROLE DE MIRA", "Feche o painel e segure o botao direito ou AIM. FOV medido em pixels.", 77)
toggle("Aimbot", "Liberar aimbot", "Liga o controle da camera. Nao dispara automaticamente.", "AimMaster")
toggle("Aimbot", "Aimbot safe", "Transicao suave. Safe descreve a suavizacao.", "AimMode", "Safe")
toggle("Aimbot", "Aimbot Full", "Alinhamento imediato. Substitui o modo Safe.", "AimMode", "Full")
toggle("Aimbot", "Aimfov", "Mostra o limite circular usado para escolher o alvo.", "AimFov")
slider("Aimbot", "Raio do Aimfov", "AimRadius", "O limite continua valendo com o circulo oculto.")
slider("Aimbot", "Forca do modo Safe", "AimStrength", "Menor = mais suave; maior = resposta mais rapida.")

section("esp", "LEITURA VISUAL", "Colete e arma usam apenas dados disponiveis ao cliente. N/D significa indisponivel.", 78)
toggle("esp", "Esp actve", "Controle geral das opcoes abaixo.", "EspMaster")
toggle("esp", "Esp box", "Caixa projetada ao redor do personagem.", "EspBox")
toggle("esp", "Esp line", "Linha da base da tela ate o personagem.", "EspLine")
toggle("esp", "Esp nome", "Nome do jogador acima da caixa.", "EspName")
toggle("esp", "Esp vida", "Barra e valor de vida do Humanoid.", "EspHealth")
toggle("esp", "Esp colete", "Valor replicado de Armor, Kevlar ou equivalente.", "EspArmor")
toggle("esp", "Esp arma", "Tool equipada ou nome replicado da arma.", "EspWeapon")
toggle("esp", "Chams", "Preenchimento e contorno do personagem.", "Chams")

section("main", "MOVIMENTO E VISAO", "Fly e Noclip + oito extras. O jogo ou servidor pode corrigir estas alteracoes.", 78)
toggle("main", "Fly", "PC: WASD + Espaco/Ctrl. Celular: joystick + botoes.", "Fly")
slider("main", "Velocidade do Fly", "FlySpeed", "Velocidade em studs por segundo.")
toggle("main", "Noclip", "Desativa localmente a colisao do personagem.", "Noclip")
toggle("main", "1. Velocidade", "Ajusta a velocidade de caminhada.", "Speed")
slider("main", "Velocidade de caminhada", "WalkSpeed", "Pode ser reescrita pelo controlador do jogo.")
toggle("main", "2. Pulo alto", "Ajusta JumpPower ou a altura equivalente.", "HighJump")
slider("main", "Forca do pulo", "JumpPower", "Converte para JumpHeight quando necessario.")
toggle("main", "3. Pulo automatico", "Pula ao tocar o chao enquanto voce se move.", "AutoJump")
toggle("main", "4. Iluminacao", "Clareia a cena e reduz sombras locais.", "Fullbright")
toggle("main", "5. Remover neblina", "Reduz Fog e Atmosphere localmente.", "NoFog")
toggle("main", "6. FOV da camera", "Altera o campo de visao da camera.", "CameraFov")
slider("main", "Campo de visao", "CameraFovValue", "Em graus. Diferente do raio do Aimfov.")
toggle("main", "7. Mira central", "Desenha uma cruz no centro da tela.", "Crosshair")
toggle("main", "8. Zoom ajustavel", "Libera o limite de zoom da camera padrao.", "Zoom")
slider("main", "Distancia maxima do zoom", "ZoomDistance", "Cameras proprias do jogo podem ignorar este limite.")

section("aothers", "FILTROS E CONTROLES", "Ajustes adicionais de alvo, alcance e informacoes.")
toggle("aothers", "Ignorar aliados no aimbot", "Compara equipes quando o jogo as informa.", "TeamCheck")
toggle("aothers", "Verificar linha de visao", "Usa raycast para evitar selecionar alvos obstruidos.", "WallCheck")
toggle("aothers", "Aim continuo", "Dispensa segurar AIM ou o botao direito.", "AimAlways")
toggle("aothers", "ESP em aliados", "Inclui jogadores da mesma equipe.", "EspAllies")
slider("aothers", "Distancia maxima", "MaxDistance", "Limite para selecao de alvo e ESP, em studs.")
toggle("aothers", "Mostrar FPS", "Contador de quadros e numero de jogadores.", "ShowStats")
action("aothers", "Ocultar painel", function() setOpen(false) end)

section("configs", "CORES E PERFIL", "Digite qualquer cor em #RRGGBB. Exporte o JSON para guardar seu perfil.", 76)
colorField("configs", "Cor do ESP", "EspColor")
colorField("configs", "Cor dos Chams", "ChamsColor")
slider("configs", "Opacidade dos Chams", "ChamsOpacity", "0 = transparente; 1 = preenchimento solido.")
colorField("configs", "Cor do Aimfov / mira", "FovColor")
colorField("configs", "Cor de destaque do painel", "AccentColor")
colorField("configs", "Cor de fundo do painel", "BackgroundColor")
colorField("configs", "Cor dos cards / abas", "CardColor")
toggle("configs", "Filtro de visao", "Aplica uma tonalidade somente na sua imagem.", "Tint")
colorField("configs", "Cor da visao do personagem", "TintColor")
slider("configs", "Saturacao da visao", "TintSaturation", "-1 = cinza; 0 = normal; 1 = cores intensas.")
local configRow = row("configs", 141)
label(configRow, "Perfil JSON — copie/cole manualmente", {Position = UDim2.fromOffset(12, 5), Size = UDim2.new(1, -24, 0, 22), TextSize = 11})
local configText = textInput(configRow, {
    Position = UDim2.fromOffset(12, 33), Size = UDim2.new(1, -24, 1, -44), MultiLine = true,
    TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
    PlaceholderText = "Exporte um perfil ou cole um JSON aqui.", TextSize = 10,
})
action("configs", "Exportar configuracoes", function()
    configText.Text = S.Http:JSONEncode({version = 1, settings = state})
    notify("Perfil exportado no campo JSON. Copie o texto para guardar.")
end)
action("configs", "Importar configuracoes", function()
    if #configText.Text > 24000 then notify("Perfil grande demais. Limite: 24 KB."); return end
    local ok, data = pcall(function() return S.Http:JSONDecode(configText.Text) end)
    if not ok or type(data) ~= "table" or data.version ~= 1 or type(data.settings) ~= "table" then
        notify("Perfil invalido. Use um JSON exportado pelo Astar-Menu v1."); return
    end
    local valid = {}
    for key, value in pairs(data.settings) do
        local base = defaults[key]
        if base == nil or type(value) ~= type(base) then notify("Campo invalido: " .. tostring(key)); return end
        if type(base) == "number" and not finite(value) then notify("Numero invalido no perfil."); return end
        if type(base) == "string" then
            if key == "AimMode" then
                if value ~= "Safe" and value ~= "Full" then notify("Modo de aimbot invalido."); return end
            elseif not validHex(value) then notify("Cor invalida no perfil."); return end
        end
        valid[key] = value
    end
    for key, value in pairs(valid) do set(key, value) end
    notify("Perfil importado.")
end)
action("configs", "Restaurar configuracoes padrao", function()
    stopFly(); restoreAll(); A.aimHeld = false
    for key, value in pairs(defaults) do state[key] = value end
    refresh(); notify("Configuracoes restauradas. Recursos de jogo desligados.")
end)

section("credts", "ASTAR-MENU", "Criado conforme o visual e a organizacao pedidos.", 64)
section("credts", "CREDITOS", "@astar.mmgh1000", 70)
section("credts", "VERSAO 1.0", "Interface cliente para mouse e toque. Compatibilidade ao vivo com Counter Blox ainda nao validada.", 92)

-- Skin catalog: read-only asset discovery, with no remote calls or module execution.
-- Paths listed here are discovered, not asserted to exist in Counter Blox.
do
    section("SkinChanger", "CATALOGO / PREVIA LOCAL", "Esta aba nao adiciona skins ao inventario oficial nem equipa itens no servidor. Lista somente assets replicados reconhecidos.", 103)
    local r = row("SkinChanger", 84)
    label(r, "Pasta de assets (opcional)", {Position = UDim2.fromOffset(12, 5), Size = UDim2.new(1, -24, 0, 22)})
    A.skinPath = textInput(r, {Position = UDim2.fromOffset(12, 35), Size = UDim2.new(1, -24, 0, 34), PlaceholderText = "ReplicatedStorage/Pasta/Subpasta"})
    local searchRow = row("SkinChanger", 50)
    A.skinSearch = textInput(searchRow, {Position = UDim2.fromOffset(10, 7), Size = UDim2.new(1, -20, 0, 34), PlaceholderText = "Filtrar assets pelo nome..."})
    A.skinStatus = section("SkinChanger", "Aguardando leitura", "Clique em Ler catalogo. Um schema privado ou ModuleScript nao sera interpretado.", 90)
    -- Keep explicit label references; child ordering is not an API contract.
    for _, child in ipairs(A.skinStatus:GetChildren()) do
        if child:IsA("TextLabel") then
            if child.Text == "Aguardando leitura" then A.skinStatusTitle = child else A.skinStatusDetail = child end
        end
    end
    local listRow = row("SkinChanger", 220)
    A.skinList = make("ScrollingFrame", {Position = UDim2.fromOffset(8, 8), Size = UDim2.new(1, -16, 1, -16), BackgroundTransparency = 1, BorderSizePixel = 0, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarThickness = 3, ZIndex = 13}, listRow)
    make("UIListLayout", {Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder}, A.skinList)
    local previewRow = row("SkinChanger", 230)
    A.previewCaption = label(previewRow, "Escolha um asset para visualizar", {Position = UDim2.fromOffset(12, 4), Size = UDim2.new(1, -24, 0, 23), TextSize = 11})
    A.preview = make("ViewportFrame", {Position = UDim2.fromOffset(8, 33), Size = UDim2.new(1, -16, 1, -41), BackgroundColor3 = Color3.fromRGB(6, 9, 18), BorderSizePixel = 0, Ambient = Color3.fromRGB(190, 190, 210), LightColor = Color3.new(1, 1, 1), ZIndex = 14}, previewRow)
    rounded(A.preview, 8)
    A.previewWorld = make("WorldModel", {}, A.preview)
    A.previewCamera = make("Camera", {FieldOfView = 45}, A.preview)
    A.preview.CurrentCamera = A.previewCamera
    A.texturePreview = make("ImageLabel", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ScaleType = Enum.ScaleType.Fit, Visible = false, ZIndex = 15}, A.preview)
end

local function assetId(v)
    if type(v) ~= "string" then return nil end
    local n = v:match("^rbxassetid://(%d+)$") or v:match("^https?://[%w%.]*roblox%.com/asset/%?id=(%d+)$") or v:match("^(%d+)$")
    return n and ("rbxassetid://" .. n) or nil
end
local function assetTexture(obj)
    if obj:IsA("StringValue") then return assetId(obj.Value) end
    if obj:IsA("Decal") or obj:IsA("Texture") then return assetId(obj.Texture) end
    if obj:IsA("SurfaceAppearance") then return assetId(obj.ColorMap) end
    return nil
end
local function previewAsset(item)
    cleanPreview()
    if not item.object.Parent then notify("O asset nao esta mais disponivel. Releia o catalogo."); return end
    A.previewCaption.Text = item.name
    local texture = assetTexture(item.object)
    if texture then A.texturePreview.Image = texture; A.texturePreview.Visible = true; return end
    local ok, cloned = pcall(function() return item.object:Clone() end)
    if not ok or not cloned then notify("Este asset nao permite clonagem para previa."); return end
    for _, obj in ipairs(cloned:GetDescendants()) do
        if obj:IsA("LuaSourceContainer") or obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") or obj:IsA("Humanoid") or obj:IsA("Sound") then obj:Destroy() end
    end
    local model
    if cloned:IsA("Model") then model = cloned else model = Instance.new("Model"); cloned.Parent = model end
    local count = 0
    for _, part in ipairs(model:GetDescendants()) do
        if part:IsA("BasePart") then part.Anchored = true; part.CanCollide = false; count = count + 1 end
    end
    if count == 0 then model:Destroy(); notify("Asset sem geometria ou textura reconhecida para previa."); return end
    model.Parent = A.previewWorld
    local box, size = model:GetBoundingBox()
    model:PivotTo(CFrame.new(-box.Position) * model:GetPivot())
    A.previewModel = model; A.previewPivot = model:GetPivot(); A.previewAngle = 0
    local radius = math.max(size.Magnitude / 2, 0.1)
    local aspect = A.preview.AbsoluteSize.X / math.max(A.preview.AbsoluteSize.Y, 1)
    local half = math.atan(math.tan(math.rad(45) / 2) * math.min(1, aspect))
    local distance = radius / math.sin(half) * 1.15
    A.previewCamera.CFrame = CFrame.lookAt(Vector3.new(distance * 0.7, distance * 0.3, distance), Vector3.new())
end
local function renderCatalog()
    for _, child in ipairs(A.skinList:GetChildren()) do if child:IsA("GuiObject") then child:Destroy() end end
    local matches = {}
    local query = A.skinQuery:lower()
    for _, item in ipairs(A.skinItems) do if item.name:lower():find(query, 1, true) then table.insert(matches, item) end end
    local pages = math.max(1, math.ceil(#matches / 40))
    A.skinPage = clamp(A.skinPage, 1, pages)
    for i = (A.skinPage - 1) * 40 + 1, math.min(A.skinPage * 40, #matches) do
        local item = matches[i]
        local button = press(A.skinList, item.name, {Size = UDim2.new(1, -5, 0, 32), TextSize = 10, TextTruncate = Enum.TextTruncate.AtEnd, LayoutOrder = i})
        button.Activated:Connect(function() previewAsset(item) end)
    end
    A.skinList.CanvasPosition = Vector2.new()
    A.skinStatusTitle.Text = string.format("%d assets  /  pagina %d de %d", #matches, A.skinPage, pages)
end
local function scanCatalog()
    if A.scanBusy then notify("A leitura ja esta em andamento."); return end
    A.scanBusy = true; A.skinToken = A.skinToken + 1
    local token = A.skinToken
    A.skinStatusTitle.Text = "Lendo assets replicados..."
    task.spawn(function()
        local ok, message = pcall(function()
            local roots, items, seen, seenItems = {}, {}, {}, {}
            local requested = A.skinPath.Text:match("^%s*(.-)%s*$")
            if requested ~= "" then
                local node = S.Storage
                local first = true
                for segment in requested:gmatch("[^/%.]+") do
                    if first and segment == "ReplicatedStorage" then first = false
                    else node = node and node:FindFirstChild(segment); first = false end
                end
                if not node or node == S.Storage then error("Informe uma subpasta existente de ReplicatedStorage.", 0) end
                roots[1] = node
            else
                local accepted = {skins = true, weaponskins = true, gunskins = true, skincatalog = true}
                local queue, head = {{S.Storage, 0}}, 1
                while head <= #queue and head <= 10000 do
                    if not A.alive or A.skinToken ~= token then return end
                    local pair = queue[head]; head = head + 1
                    local normalized = pair[1].Name:lower():gsub("[^%w]", "")
                    if accepted[normalized] then table.insert(roots, pair[1])
                    elseif pair[2] < 7 then
                        for _, child in ipairs(pair[1]:GetChildren()) do table.insert(queue, {child, pair[2] + 1}) end
                    end
                    if head % 250 == 0 then task.wait() end
                end
            end
            local queue, head, examined = {}, 1, 0
            for _, root in ipairs(roots) do table.insert(queue, root) end
            while head <= #queue and examined < 30000 do
                if not A.alive or A.skinToken ~= token then return end
                local obj = queue[head]; head = head + 1
                if not seen[obj] then
                    seen[obj] = true; examined = examined + 1
                    local previewable = obj:IsA("BasePart") or (obj:IsA("Model") and obj:FindFirstChildWhichIsA("BasePart", true)) or assetTexture(obj)
                    if previewable and not seenItems[obj] then
                        seenItems[obj] = true
                        table.insert(items, {name = obj:GetFullName():gsub("^ReplicatedStorage[%.]", ""), object = obj})
                    else
                        for _, child in ipairs(obj:GetChildren()) do table.insert(queue, child) end
                    end
                    if examined % 200 == 0 then task.wait() end
                end
            end
            if not A.alive or A.skinToken ~= token then return end
            table.sort(items, function(x, y) return x.name:lower() < y.name:lower() end)
            A.skinItems = items; A.skinPage = 1
            A.skinStatusDetail.Text = (#roots == 0 and "Nenhuma pasta reconhecida. Informe o caminho de assets, se disponivel.")
                or (#items == 0 and "Pasta encontrada, mas sem modelos/texturas legiveis. Nao e um catalogo de inventario.")
                or ((examined >= 30000 and "Leitura parcial: limite de 30 mil objetos. ") or "") .. "Assets locais encontrados; nomes e estrutura nao comprovam posse ou disponibilidade de skins."
            renderCatalog()
        end)
        if A.alive and A.skinToken == token then
            A.scanBusy = false
            if not ok then A.skinStatusTitle.Text = "Leitura indisponivel"; A.skinStatusDetail.Text = tostring(message) end
        end
    end)
end
action("SkinChanger", "Ler catalogo de assets disponiveis", scanCatalog).Parent.LayoutOrder = 4
A.skinStatus.LayoutOrder = 5
A.skinList.Parent.LayoutOrder = 6
action("SkinChanger", "Pagina anterior", function() A.skinPage = A.skinPage - 1; renderCatalog() end).Parent.LayoutOrder = 7
action("SkinChanger", "Proxima pagina", function() A.skinPage = A.skinPage + 1; renderCatalog() end).Parent.LayoutOrder = 8
A.preview.Parent.LayoutOrder = 9
connect(A.skinSearch.FocusLost, function() A.skinQuery = A.skinSearch.Text; A.skinPage = 1; renderCatalog() end)

local circle = make("Frame", {Name = "AimFov", AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1, BorderSizePixel = 0, Visible = false, ZIndex = 2}, gui)
make("UICorner", {CornerRadius = UDim.new(1, 0)}, circle)
local circleStroke = make("UIStroke", {Thickness = 1.5, Transparency = 0.1}, circle)
local cross = make("Frame", {Name = "Crosshair", Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(1, 1), BackgroundTransparency = 1, ZIndex = 3, Visible = false}, gui)
for _, props in ipairs({{-9, -1, 6, 2}, {4, -1, 6, 2}, {-1, -9, 2, 6}, {-1, 4, 2, 6}}) do
    make("Frame", {Position = UDim2.fromOffset(props[1], props[2]), Size = UDim2.fromOffset(props[3], props[4]), BorderSizePixel = 0, ZIndex = 3}, cross)
end
local stats = label(gui, "", {Position = UDim2.new(1, -195, 0, 8), Size = UDim2.fromOffset(180, 21), TextXAlignment = Enum.TextXAlignment.Right, TextSize = 11, ZIndex = 5, TextStrokeTransparency = 0.4})
local aimButton = press(gui, "AIM", {Position = UDim2.new(1, -100, 0.5, -30), Size = UDim2.fromOffset(74, 54), Visible = false, ZIndex = 25})
local upButton = press(gui, "SUBIR", {Position = UDim2.new(1, -102, 0.5, 34), Size = UDim2.fromOffset(78, 42), Visible = false, ZIndex = 25})
local downButton = press(gui, "DESCER", {Position = UDim2.new(1, -102, 0.5, 84), Size = UDim2.fromOffset(78, 42), Visible = false, ZIndex = 25})
for _, button in ipairs({aimButton, upButton, downButton}) do themed(button, "BackgroundColor3", "AccentColor") end
local function holdButton(button, key)
    connect(button.InputBegan, function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            A.touches[input] = key
            if key == "aim" then A.aimHeld = true else A.held[key] = true end
        end
    end)
end
holdButton(aimButton, "aim"); holdButton(upButton, "up"); holdButton(downButton, "down")

connect(top.InputBegan, function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
    local start = Vector2.new(input.Position.X, input.Position.Y)
    local center = panel.AbsolutePosition + panel.AbsoluteSize / 2
    A.drag = {input = input, kind = "panel", move = function(position)
        local camera = workspace.CurrentCamera
        if not camera then return end
        local p = center + Vector2.new(position.X, position.Y) - start
        local half = panel.AbsoluteSize / 2
        local v = camera.ViewportSize
        panel.Position = UDim2.fromOffset(clamp(p.X, half.X + 4, v.X - half.X - 4), clamp(p.Y, half.Y + 4, v.Y - half.Y - 4))
    end}
end)
connect(S.Input.InputChanged, function(input)
    local drag = A.drag
    if drag and ((drag.input.UserInputType == Enum.UserInputType.MouseButton1 and input.UserInputType == Enum.UserInputType.MouseMovement) or input == drag.input) then drag.move(input.Position) end
end)
connect(S.Input.InputBegan, function(input, processed)
    if S.Input:GetFocusedTextBox() then return end
    if input.KeyCode == Enum.KeyCode.RightShift then setOpen(not A.menuOpen); return end
    if processed or A.menuOpen then return end
    A.held[input.KeyCode] = true
    if input.UserInputType == Enum.UserInputType.MouseButton2 then A.aimHeld = true end
end)
connect(S.Input.InputEnded, function(input)
    A.held[input.KeyCode] = nil
    if input.UserInputType == Enum.UserInputType.MouseButton2 then A.aimHeld = false end
    local key = A.touches[input]
    if key then
        A.touches[input] = nil
        if key == "aim" then A.aimHeld = false else A.held[key] = nil end
    end
    local drag = A.drag
    if drag and (input == drag.input or (drag.input.UserInputType == Enum.UserInputType.MouseButton1 and input.UserInputType == Enum.UserInputType.MouseButton1)) then
        if drag.page then drag.page.ScrollingEnabled = true end
        A.drag = nil
    end
end)
connect(S.Input.WindowFocusReleased, function()
    A.held = {}; A.touches = {}; A.aimHeld = false; A.drag = nil
    for _, page in pairs(A.pages) do page.ScrollingEnabled = true end
end)

local function living(p)
    local character = p.Character
    if not character or not character.Parent then return nil end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local root = character:FindFirstChild("HumanoidRootPart") or character.PrimaryPart
    if not humanoid or humanoid.Health <= 0 or not root or not root:IsA("BasePart") then return nil end
    return character, humanoid, root
end
local function allies(p)
    return p.Team ~= nil and player.Team ~= nil and not p.Neutral and not player.Neutral and p.Team == player.Team
end
local function visible(camera, target, character, ownCharacter)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local exclude = {camera, effects}
    if ownCharacter then table.insert(exclude, ownCharacter) end
    params.FilterDescendantsInstances = exclude
    params.IgnoreWater = true
    local result = workspace:Raycast(camera.CFrame.Position, target.Position - camera.CFrame.Position, params)
    return not result or result.Instance:IsDescendantOf(character)
end
local function aimStep(camera, dt)
    if not state.AimMaster or A.menuOpen or S.Input:GetFocusedTextBox() or (not A.aimHeld and not state.AimAlways) then return end
    local own = living(player)
    if not own then return end
    local center = camera.ViewportSize / 2
    local nearest, best = nil, state.AimRadius
    for _, other in ipairs(S.Players:GetPlayers()) do
        if other ~= player and not (state.TeamCheck and allies(other)) then
            local character, _, root = living(other)
            if character then
                local target = character:FindFirstChild("Head") or root
                if target:IsA("BasePart") and (target.Position - camera.CFrame.Position).Magnitude <= state.MaxDistance then
                    local p, onScreen = camera:WorldToViewportPoint(target.Position)
                    if onScreen and p.Z > 0 then
                        local distance = (Vector2.new(p.X, p.Y) - center).Magnitude
                        if distance < best and (not state.WallCheck or visible(camera, target, character, own)) then nearest = target; best = distance end
                    end
                end
            end
        end
    end
    if nearest and (nearest.Position - camera.CFrame.Position).Magnitude > 0.01 then
        local goal = CFrame.lookAt(camera.CFrame.Position, nearest.Position)
        local alpha = state.AimMode == "Full" and 1 or (1 - math.exp(-state.AimStrength * math.min(dt, 0.1)))
        camera.CFrame = camera.CFrame:Lerp(goal, alpha)
    end
end

local function scalarStat(p, character, names)
    local containers = {character, p}
    for _, root in ipairs({character, p}) do
        for _, folderName in ipairs({"Status", "Stats", "leaderstats"}) do
            local folder = root:FindFirstChild(folderName)
            if folder then table.insert(containers, folder) end
        end
    end
    for _, container in ipairs(containers) do
        for _, name in ipairs(names) do
            local v = container:GetAttribute(name)
            if type(v) == "string" or finite(v) then return v end
            local obj = container:FindFirstChild(name)
            if obj then
                if obj:IsA("StringValue") or obj:IsA("IntValue") or obj:IsA("NumberValue") then return obj.Value end
                if obj:IsA("ObjectValue") and obj.Value then return obj.Value.Name end
            end
        end
    end
    return nil
end
local function weaponName(p, character)
    local tool = character:FindFirstChildOfClass("Tool")
    if tool then return tool.Name end
    return scalarStat(p, character, {"EquippedTool", "EquippedWeapon", "CurrentWeapon", "Weapon", "Equipped", "Gun"})
end
local function newEsp(other)
    local box = make("Frame", {BackgroundTransparency = 1, BorderSizePixel = 0, Visible = false, ZIndex = 2}, gui)
    local stroke = make("UIStroke", {Thickness = 1.3}, box)
    local name = label(gui, "", {TextXAlignment = Enum.TextXAlignment.Center, TextSize = 11, Visible = false, TextStrokeTransparency = 0.2, ZIndex = 3})
    local info = label(gui, "", {TextXAlignment = Enum.TextXAlignment.Center, TextYAlignment = Enum.TextYAlignment.Top, TextTruncate = Enum.TextTruncate.None, TextSize = 10, Visible = false, TextStrokeTransparency = 0.2, ZIndex = 3})
    local line = make("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), BorderSizePixel = 0, Visible = false, ZIndex = 1}, gui)
    local bar = make("Frame", {BorderSizePixel = 0, BackgroundColor3 = Color3.fromRGB(23, 28, 35), Visible = false, ZIndex = 3}, gui)
    local fill = make("Frame", {AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1), BorderSizePixel = 0, ZIndex = 4}, bar)
    local highlight = make("Highlight", {Adornee = other.Character, Enabled = false, DepthMode = Enum.HighlightDepthMode.AlwaysOnTop, OutlineTransparency = 0.1}, effects)
    return {box = box, stroke = stroke, name = name, info = info, line = line, bar = bar, fill = fill, highlight = highlight, metadataAt = 0}
end
local function hideEsp(record)
    record.box.Visible = false; record.name.Visible = false; record.info.Visible = false
    record.line.Visible = false; record.bar.Visible = false; record.highlight.Enabled = false
end
local function removeEsp(other)
    local r = A.records[other]
    if not r then return end
    for _, obj in ipairs({r.box, r.name, r.info, r.line, r.bar, r.highlight}) do obj:Destroy() end
    A.records[other] = nil
end
local function screenBounds(camera, character)
    local cframe, size = character:GetBoundingBox()
    local loX, loY, hiX, hiY = math.huge, math.huge, -math.huge, -math.huge
    for x = -1, 1, 2 do for y = -1, 1, 2 do for z = -1, 1, 2 do
        local v = camera:WorldToViewportPoint(cframe * Vector3.new(size.X * x / 2, size.Y * y / 2, size.Z * z / 2))
        if v.Z <= 0.05 then return nil end
        loX = math.min(loX, v.X); loY = math.min(loY, v.Y); hiX = math.max(hiX, v.X); hiY = math.max(hiY, v.Y)
    end end end
    local viewport = camera.ViewportSize
    if hiX < 0 or hiY < 0 or loX > viewport.X or loY > viewport.Y then return nil end
    return clamp(loX, 0, viewport.X), clamp(loY, 0, viewport.Y), clamp(hiX, 0, viewport.X), clamp(hiY, 0, viewport.Y)
end
local function espStep(camera)
    for _, other in ipairs(S.Players:GetPlayers()) do
        if other ~= player then
            local r = A.records[other]
            if state.EspMaster and not r then r = newEsp(other); A.records[other] = r end
            if r then
                hideEsp(r)
                local character, humanoid, root = living(other)
                if state.EspMaster and character and (state.EspAllies or not allies(other)) and (root.Position - camera.CFrame.Position).Magnitude <= state.MaxDistance then
                    r.highlight.Adornee = character; r.highlight.Enabled = state.Chams
                    r.highlight.FillColor = color(state.ChamsColor); r.highlight.OutlineColor = color(state.EspColor)
                    r.highlight.FillTransparency = 1 - state.ChamsOpacity
                    local left, topY, right, bottom = screenBounds(camera, character)
                    if left then
                        local width, height = math.max(1, right - left), math.max(1, bottom - topY)
                        local centerX, c = (left + right) / 2, color(state.EspColor)
                        r.box.Position = UDim2.fromOffset(left, topY); r.box.Size = UDim2.fromOffset(width, height)
                        r.box.Visible = state.EspBox; r.stroke.Color = c
                        r.name.Text = other.DisplayName .. " (@" .. other.Name .. ")"
                        r.name.Position = UDim2.fromOffset(centerX - 110, math.max(0, topY - 20)); r.name.Size = UDim2.fromOffset(220, 18)
                        r.name.Visible = state.EspName; r.name.TextColor3 = c
                        if os.clock() >= r.metadataAt or r.character ~= character then
                            r.armor = scalarStat(other, character, {"Armor", "Kevlar", "Vest", "Colete", "armor"})
                            r.weapon = weaponName(other, character); r.metadataAt = os.clock() + 0.25; r.character = character
                        end
                        local texts = {}
                        if state.EspHealth then table.insert(texts, "Vida: " .. tostring(math.floor(humanoid.Health + 0.5))) end
                        if state.EspArmor then table.insert(texts, "Colete: " .. tostring(r.armor or "N/D")) end
                        if state.EspWeapon then table.insert(texts, "Arma: " .. tostring(r.weapon or "N/D")) end
                        r.info.Text = table.concat(texts, "\n"); r.info.TextColor3 = c
                        r.info.Position = UDim2.fromOffset(centerX - 110, bottom + 3); r.info.Size = UDim2.fromOffset(220, #texts * 13 + 2)
                        r.info.Visible = #texts > 0
                        local ratio = clamp(humanoid.Health / math.max(humanoid.MaxHealth, 1), 0, 1)
                        r.bar.Position = UDim2.fromOffset(math.max(0, left - 6), topY); r.bar.Size = UDim2.fromOffset(3, height); r.bar.Visible = state.EspHealth
                        r.fill.Size = UDim2.new(1, 0, ratio, 0); r.fill.BackgroundColor3 = Color3.fromHSV(ratio / 3, 0.85, 1)
                        local start = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y - 4)
                        local finish = Vector2.new(centerX, bottom); local delta = finish - start
                        r.line.Position = UDim2.fromOffset((start.X + finish.X) / 2, (start.Y + finish.Y) / 2)
                        r.line.Size = UDim2.fromOffset(delta.Magnitude, 1); r.line.Rotation = math.deg(math.atan2(delta.Y, delta.X))
                        r.line.BackgroundColor3 = c; r.line.Visible = state.EspLine
                    end
                end
            end
        end
    end
end
connect(S.Players.PlayerRemoving, removeEsp)

local function startFly(humanoid, root)
    stopFly()
    local attachment = make("Attachment", {Name = "AstarFlightAttachment"}, root)
    local velocity = make("LinearVelocity", {Name = "AstarFlightVelocity", Attachment0 = attachment, RelativeTo = Enum.ActuatorRelativeTo.World, VelocityConstraintMode = Enum.VelocityConstraintMode.Vector, ForceLimitsEnabled = false, VectorVelocity = Vector3.new()}, root)
    local orientation = make("AlignOrientation", {Name = "AstarFlightOrientation", Attachment0 = attachment, Mode = Enum.OrientationAlignmentMode.OneAttachment, MaxTorque = 1000000, Responsiveness = 30}, root)
    A.flying = {root = root, humanoid = humanoid, velocity = velocity, orientation = orientation, objects = {velocity, orientation, attachment}}
end
local function movementStep()
    local character, humanoid, root = living(player)
    if not character then
        stopFly(); restore("noclip"); restore("speed"); restore("jump"); return
    end
    if state.Noclip then
        for _, part in ipairs(character:GetDescendants()) do if part:IsA("BasePart") then override("noclip", part, "CanCollide", false) end end
    else restore("noclip") end
    if state.Speed then override("speed", humanoid, "WalkSpeed", state.WalkSpeed) else restore("speed") end
    if state.HighJump then
        if humanoid.UseJumpPower then override("jump", humanoid, "JumpPower", state.JumpPower)
        else override("jump", humanoid, "JumpHeight", state.JumpPower ^ 2 / (2 * math.max(workspace.Gravity, 1))) end
    else restore("jump") end
    if state.AutoJump and not state.Fly and not A.menuOpen and humanoid.MoveDirection.Magnitude > 0.1 and humanoid.FloorMaterial ~= Enum.Material.Air then humanoid.Jump = true end
    if state.Fly then
        if not A.flying or A.flying.root ~= root or not A.flying.velocity.Parent then startFly(humanoid, root) end
        local camera = workspace.CurrentCamera
        if camera then
            override("flight", humanoid, "AutoRotate", false); override("flight", humanoid, "PlatformStand", true)
            local move = Vector3.new()
            if not A.menuOpen and not S.Input:GetFocusedTextBox() then
                local right = (A.held[Enum.KeyCode.D] and 1 or 0) - (A.held[Enum.KeyCode.A] and 1 or 0)
                local forward = (A.held[Enum.KeyCode.W] and 1 or 0) - (A.held[Enum.KeyCode.S] and 1 or 0)
                move = camera.CFrame.RightVector * right + camera.CFrame.LookVector * forward
                if S.Input.TouchEnabled and right == 0 and forward == 0 then move = humanoid.MoveDirection end
                local vertical = ((A.held[Enum.KeyCode.Space] or A.held.up) and 1 or 0) - ((A.held[Enum.KeyCode.LeftControl] or A.held.down) and 1 or 0)
                move = move + Vector3.new(0, vertical, 0)
            end
            if move.Magnitude > 1 then move = move.Unit end
            A.flying.velocity.VectorVelocity = move * state.FlySpeed
            local look = camera.CFrame.LookVector
            local yaw = Vector3.new(look.X, 0, look.Z)
            if yaw.Magnitude > 0.001 then A.flying.orientation.CFrame = CFrame.lookAt(Vector3.new(), yaw.Unit) end
        end
    else stopFly() end
end
connect(player.CharacterRemoving, function()
    stopFly(); restore("noclip"); restore("speed"); restore("jump")
    A.held = {}; A.touches = {}; A.aimHeld = false
end)

local function environmentStep()
    if state.Fullbright then
        override("light", S.Lighting, "Brightness", 2)
        override("light", S.Lighting, "Ambient", Color3.fromRGB(190, 190, 190))
        override("light", S.Lighting, "OutdoorAmbient", Color3.fromRGB(190, 190, 190))
        override("light", S.Lighting, "GlobalShadows", false)
        override("light", S.Lighting, "ClockTime", 14)
    else restore("light") end
    if state.NoFog then
        override("fog", S.Lighting, "FogStart", 0); override("fog", S.Lighting, "FogEnd", 1000000)
        for _, atmosphere in ipairs(S.Lighting:GetChildren()) do
            if atmosphere:IsA("Atmosphere") then override("fog", atmosphere, "Density", 0); override("fog", atmosphere, "Haze", 0) end
        end
    else restore("fog") end
    if state.Zoom then
        override("zoom", player, "CameraMode", Enum.CameraMode.Classic)
        override("zoom", player, "CameraMinZoomDistance", math.min(0.5, player.CameraMaxZoomDistance))
        override("zoom", player, "CameraMaxZoomDistance", state.ZoomDistance)
    else restore("zoom") end
    tint.Enabled = state.Tint; tint.TintColor = color(state.TintColor); tint.Saturation = state.TintSaturation
end

local function reportError(where, message)
    if os.clock() - A.lastError > 5 then
        A.lastError = os.clock()
        warn("Astar-Menu / " .. where .. ": " .. tostring(message))
        notify("Falha em " .. where .. ". O console mostra o detalhe; compatibilidade pode exigir ajuste.")
    end
end
local bindName = "AstarMenu_Render_" .. tostring(player.UserId) .. "_" .. S.Http:GenerateGUID(false)
local function cleanup()
    if not A.alive then return end
    A.alive = false; A.skinToken = A.skinToken + 1
    S.Run:UnbindFromRenderStep(bindName)
    stopFly(); restoreAll()
    for _, connection in ipairs(A.connections) do connection:Disconnect() end
    A.connections = {}
    effects:Destroy(); tint:Destroy()
    if gui.Parent then gui:Destroy() end
end
connect(shutdown.Event, cleanup)
connect(gui.Destroying, cleanup)
action("configs", "Descarregar menu e restaurar alteracoes", cleanup)

table.insert(A.refreshers, function()
    circleStroke.Color = color(state.FovColor)
    for _, part in ipairs(cross:GetChildren()) do if part:IsA("Frame") then part.BackgroundColor3 = color(state.FovColor) end end
end)
refresh()
connect(S.Run.PreSimulation, function()
    if not A.alive then return end
    local ok, err = pcall(movementStep)
    if not ok then
        stopFly(); restore("noclip"); restore("speed"); restore("jump")
        reportError("movimento", err)
    end
end)
connect(S.Run.Heartbeat, function(dt)
    if not A.alive then return end
    A.tickTime = A.tickTime + dt
    if A.tickTime >= 0.2 then
        A.tickTime = 0
        local ok, err = pcall(environmentStep)
        if not ok then reportError("ambiente", err) end
    end
end)
S.Run:BindToRenderStep(bindName, Enum.RenderPriority.Camera.Value + 1, function(dt)
    if not A.alive then return end
    local camera = workspace.CurrentCamera
    if not camera then return end
    if A.lastCamera ~= camera then restore("fov"); A.lastCamera = camera end
    if not A.viewport or A.viewport ~= camera.ViewportSize then resize() end
    if state.CameraFov then override("fov", camera, "FieldOfView", state.CameraFovValue) else restore("fov") end
    circle.Visible = state.AimMaster and state.AimFov
    circle.Position = UDim2.fromOffset(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    circle.Size = UDim2.fromOffset(state.AimRadius * 2, state.AimRadius * 2)
    cross.Visible = state.Crosshair
    aimButton.Visible = S.Input.TouchEnabled and state.AimMaster and not state.AimAlways and not A.menuOpen
    upButton.Visible = S.Input.TouchEnabled and state.Fly and not A.menuOpen; downButton.Visible = upButton.Visible
    A.frameCount = A.frameCount + 1; A.frameTime = A.frameTime + dt
    if A.frameTime >= 0.5 then
        A.fps = math.floor(A.frameCount / A.frameTime + 0.5); A.frameCount = 0; A.frameTime = 0
        stats.Text = tostring(A.fps) .. " FPS  /  " .. tostring(#S.Players:GetPlayers()) .. " jogadores"
    end
    stats.Visible = state.ShowStats
    if toast.Visible and os.clock() >= (A.toastUntil or 0) then toast.Visible = false end
    if A.previewModel and A.menuOpen and A.activePage == "SkinChanger" then
        A.previewAngle = A.previewAngle + dt * 0.45
        A.previewModel:PivotTo(CFrame.Angles(0, A.previewAngle, 0) * A.previewPivot)
    end
    local ok, err = pcall(aimStep, camera, dt)
    if not ok then reportError("aimbot", err) end
    ok, err = pcall(espStep, camera)
    if not ok then reportError("ESP", err) end
end)
notify("Astar-Menu pronto. Escolha os recursos e feche o painel para us
