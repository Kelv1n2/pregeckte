--// AutoWalk v5.0 ULTRA — Cyberpunk Edition //--
--// Design: Matrix Rain · Waveform · Partículas · Glitch · Temas //--

local AutoWalk = {}

-- ══════════════════════════════════════════════
--   SERVIÇOS
-- ══════════════════════════════════════════════
local Players              = game:GetService("Players")
local RunService           = game:GetService("RunService")
local TweenService         = game:GetService("TweenService")
local ContextActionService = game:GetService("ContextActionService")
local UserInputService     = game:GetService("UserInputService")
local HttpService          = game:GetService("HttpService")

-- ══════════════════════════════════════════════
--   TEMAS (5 presets — igual aos swatches)
-- ══════════════════════════════════════════════
AutoWalk.Themes = {
    { name="CYAN",   accent=Color3.fromRGB(0,255,229),  accent2=Color3.fromRGB(0,170,255),  mod=Color3.fromRGB(123,47,255) },
    { name="PINK",   accent=Color3.fromRGB(255,45,107), accent2=Color3.fromRGB(255,107,0),  mod=Color3.fromRGB(255,45,107) },
    { name="PURPLE", accent=Color3.fromRGB(123,47,255), accent2=Color3.fromRGB(170,68,255), mod=Color3.fromRGB(34,0,255)   },
    { name="GOLD",   accent=Color3.fromRGB(255,230,0),  accent2=Color3.fromRGB(255,136,0),  mod=Color3.fromRGB(255,170,0)  },
    { name="BLUE",   accent=Color3.fromRGB(0,170,255),  accent2=Color3.fromRGB(0,85,255),   mod=Color3.fromRGB(0,221,255)  },
}
AutoWalk.CurrentTheme = 1  -- índice do tema ativo

-- ══════════════════════════════════════════════
--   CONFIGURAÇÃO
-- ══════════════════════════════════════════════
AutoWalk.Config = {
    ACTION_NAME    = "AutoWalkV5",
    TOGGLE_KEY     = Enum.KeyCode.R,
    DEBOUNCE_TIME  = 0.25,
    TWEEN_OPEN     = 0.45,
    TWEEN_CLOSE    = 0.22,
    DEFAULT_SPEED  = 16,
    MIN_SPEED      = 8,
    MAX_SPEED      = 100,
    HUMANIZE_MIN   = 0.8,
    HUMANIZE_MAX   = 2.2,
    SPEED_VARIANCE = 1.5,
    SAVE_KEY       = "AutoWalk_v5_ultra",
    GLITCH_INTERVAL = 4.2,  -- segundos entre glitches no speed
}

-- ══════════════════════════════════════════════
--   ESTADO
-- ══════════════════════════════════════════════
AutoWalk.State = {
    enabled       = false,
    minimized     = false,
    debounce      = false,
    jumpEnabled   = false,
    humanizeOn    = true,
    connections   = {},
    hrp           = nil,
    hum           = nil,
    originalSpeed = nil,
    currentSpeed  = 16,
    lastJumpTime  = 0,
    humanizeTimer = 0,
    glitchTimer   = 0,
    waveTimer     = 0,
    matrixTimer   = 0,
}

local player = Players.LocalPlayer

-- ══════════════════════════════════════════════
--   SAVE / LOAD
-- ══════════════════════════════════════════════
function AutoWalk:_save()
    local d = {
        speed   = self.State.currentSpeed,
        jump    = self.State.jumpEnabled,
        human   = self.State.humanizeOn,
        theme   = self.CurrentTheme,
    }
    local ok, enc = pcall(HttpService.JSONEncode, HttpService, d)
    if not ok then return end
    if writefile then pcall(writefile, self.Config.SAVE_KEY..".json", enc) end
    _G[self.Config.SAVE_KEY] = d
end

function AutoWalk:_load()
    local d
    if readfile then
        local ok, c = pcall(readfile, self.Config.SAVE_KEY..".json")
        if ok and c then
            local ok2, dec = pcall(HttpService.JSONDecode, HttpService, c)
            if ok2 then d = dec end
        end
    end
    d = d or _G[self.Config.SAVE_KEY]
    if d then
        self.State.currentSpeed  = d.speed  or self.Config.DEFAULT_SPEED
        self.State.jumpEnabled   = d.jump   or false
        self.State.humanizeOn    = d.human  ~= false
        self.CurrentTheme        = d.theme  or 1
    end
end

-- ══════════════════════════════════════════════
--   HELPERS
-- ══════════════════════════════════════════════
local function newCorner(p, r)
    local c = Instance.new("UICorner", p)
    c.CornerRadius = UDim.new(0, r or 8)
end

local function newStroke(p, col, thick, trans)
    local s = Instance.new("UIStroke", p)
    s.Color       = col or Color3.fromRGB(26,26,58)
    s.Thickness   = thick or 1
    s.Transparency = trans or 0
    return s
end

local function newLabel(p, text, size, color, font, xAlign)
    local l = Instance.new("TextLabel", p)
    l.Text               = text
    l.TextColor3         = color
    l.Font               = font or Enum.Font.GothamBold
    l.TextSize           = size
    l.TextScaled         = false
    l.BackgroundTransparency = 1
    l.TextXAlignment     = xAlign or Enum.TextXAlignment.Left
    l.RichText           = true
    return l
end

local function newMono(p, text, size, color, xAlign)
    return newLabel(p, text, size, color, Enum.Font.Code, xAlign or Enum.TextXAlignment.Left)
end

local function newBtn(p, text, size, color, bg)
    local b = Instance.new("TextButton", p)
    b.Text              = text
    b.TextColor3        = color
    b.BackgroundColor3  = bg or Color3.fromRGB(15,15,25)
    b.Font              = Enum.Font.GothamBold
    b.TextSize          = size
    b.TextScaled        = false
    b.AutoButtonColor   = false
    b.BorderSizePixel   = 0
    return b
end

local function lerp(a, b, t) return a + (b - a) * t end
local function lerpColor(a, b, t)
    return Color3.new(
        lerp(a.R, b.R, t),
        lerp(a.G, b.G, t),
        lerp(a.B, b.B, t)
    )
end

-- ══════════════════════════════════════════════
--   BUILD GUI
-- ══════════════════════════════════════════════
function AutoWalk:_buildGui()
    local T  = self.Themes[self.CurrentTheme]
    local BG = {
        Color3.fromRGB(5,5,14),
        Color3.fromRGB(9,9,26),
        Color3.fromRGB(13,13,32),
        Color3.fromRGB(17,17,40),
    }
    local DIM    = Color3.fromRGB(60,60,100)
    local DIMMED = Color3.fromRGB(40,40,70)
    local OFF    = Color3.fromRGB(255,45,107)
    local WHITE  = Color3.fromRGB(200,205,240)

    -- ScreenGui
    local gui = Instance.new("ScreenGui")
    gui.Name           = "AutoWalkV5"
    gui.ResetOnSpawn   = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.IgnoreGuiInset = true
    local ok = pcall(function() gui.Parent = game:GetService("CoreGui") end)
    if not ok then gui.Parent = player:WaitForChild("PlayerGui") end

    -- Panel principal
    local panel = Instance.new("Frame", gui)
    panel.Name               = "Panel"
    panel.Size               = UDim2.new(0, 300, 0, 310)
    panel.Position           = UDim2.new(0.04, 0, 0.16, 0)
    panel.BackgroundColor3   = BG[2]
    panel.BackgroundTransparency = 1
    panel.Active             = true
    panel.Draggable          = true
    panel.ClipsDescendants   = true
    newCorner(panel, 18)
    local panelStroke = newStroke(panel, Color3.fromRGB(0,255,229), 1, 0.85)

    -- Gradiente sutil no fundo
    local grad = Instance.new("UIGradient", panel)
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(12,12,26)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(6,6,16)),
    })
    grad.Rotation = 135

    -- ── CORNER ACCENTS (4 cantos SVG-like com frames) ──
    local function makeCorner(xAnchor, yAnchor, flipX, flipY)
        local c = Instance.new("Frame", panel)
        c.Size             = UDim2.new(0, 18, 0, 18)
        c.AnchorPoint      = Vector2.new(xAnchor, yAnchor)
        c.Position         = UDim2.new(xAnchor, 0, yAnchor, 0)
        c.BackgroundTransparency = 1
        c.BorderSizePixel  = 0
        -- Borda superior
        local top = Instance.new("Frame", c)
        top.Size           = UDim2.new(1, 0, 0, 1)
        top.Position       = UDim2.new(0,0,0,0)
        top.BackgroundColor3 = T.accent
        top.BorderSizePixel = 0
        -- Borda lateral
        local side = Instance.new("Frame", c)
        side.Size          = UDim2.new(0, 1, 1, 0)
        side.Position      = UDim2.new(flipX and 1 or 0, flipX and -1 or 0, 0, 0)
        side.BackgroundColor3 = T.accent
        side.BorderSizePixel = 0
        return c, top, side
    end
    local _, ctlT, ctlS = makeCorner(0,0,false,false)
    local _, ctrT, ctrS = makeCorner(1,0,true, false)
    local _, cblT, cblS = makeCorner(0,1,false,true)
    local _, cbrT, cbrS = makeCorner(1,1,true, true)
    self._cornerParts = {ctlT,ctlS,ctrT,ctrS,cblT,cblS,cbrT,cbrS}

    -- ── TITLEBAR ──
    local tb = Instance.new("Frame", panel)
    tb.Size             = UDim2.new(1, 0, 0, 40)
    tb.BackgroundColor3 = Color3.fromRGB(7,7,16)
    tb.BorderSizePixel  = 0
    newCorner(tb, 18)
    -- filler para cobrir cantos inferiores arredondados da titlebar
    local tbFill = Instance.new("Frame", tb)
    tbFill.Size             = UDim2.new(1,0, 0, 10)
    tbFill.Position         = UDim2.new(0,0,1,-10)
    tbFill.BackgroundColor3 = Color3.fromRGB(7,7,16)
    tbFill.BorderSizePixel  = 0
    -- linha separadora
    local tbLine = Instance.new("Frame", tb)
    tbLine.Size             = UDim2.new(1,0,0,1)
    tbLine.Position         = UDim2.new(0,0,1,-1)
    tbLine.BackgroundColor3 = Color3.fromRGB(20,20,45)
    tbLine.BorderSizePixel  = 0

    -- Ícone girando (anel + dot)
    local iconWrap = Instance.new("Frame", tb)
    iconWrap.Size             = UDim2.new(0, 32, 0, 32)
    iconWrap.Position         = UDim2.new(0, 10, 0.5, -16)
    iconWrap.BackgroundColor3 = Color3.fromRGB(0, 35, 28)
    iconWrap.BorderSizePixel  = 0
    newCorner(iconWrap, 99)
    newStroke(iconWrap, T.accent, 1, 0.5)
    local iconDot = Instance.new("Frame", iconWrap)
    iconDot.Size             = UDim2.new(0, 10, 0, 10)
    iconDot.Position         = UDim2.new(0.5,-5, 0.5,-5)
    iconDot.BackgroundColor3 = OFF
    iconDot.BorderSizePixel  = 0
    newCorner(iconDot, 99)

    -- Título + subtítulo
    local titleLbl = newMono(tb, "AUTOWALK", 12, T.accent)
    titleLbl.Size     = UDim2.new(0, 150, 0, 15)
    titleLbl.Position = UDim2.new(0, 50, 0, 7)
    titleLbl.TextTransparency = 1

    local subLbl = newMono(tb, "SYS·v5.0·ULTRA·CYBR", 8, DIMMED)
    subLbl.Size     = UDim2.new(0, 170, 0, 11)
    subLbl.Position = UDim2.new(0, 50, 0, 23)
    subLbl.TextTransparency = 1

    -- Botões titlebar
    local minBtn = newBtn(tb, "─", 11, DIM, Color3.fromRGB(10,10,20))
    minBtn.Size     = UDim2.new(0, 22, 0, 22)
    minBtn.Position = UDim2.new(1, -48, 0.5, -11)
    minBtn.TextTransparency = 1
    newCorner(minBtn, 5)
    newStroke(minBtn, Color3.fromRGB(30,30,60), 1)

    local closeBtn = newBtn(tb, "✕", 10, Color3.fromRGB(160,40,60), Color3.fromRGB(10,10,20))
    closeBtn.Size     = UDim2.new(0, 22, 0, 22)
    closeBtn.Position = UDim2.new(1, -24, 0.5, -11)
    closeBtn.TextTransparency = 1
    newCorner(closeBtn, 5)
    newStroke(closeBtn, Color3.fromRGB(60,20,30), 1)

    -- ── BODY ──
    local body = Instance.new("Frame", panel)
    body.Name               = "Body"
    body.Size               = UDim2.new(1, 0, 1, -40)
    body.Position           = UDim2.new(0, 0, 0, 40)
    body.BackgroundTransparency = 1
    body.ClipsDescendants   = true

    local pad = Instance.new("UIPadding", body)
    pad.PaddingLeft   = UDim.new(0, 13)
    pad.PaddingRight  = UDim.new(0, 13)
    pad.PaddingTop    = UDim.new(0, 10)
    pad.PaddingBottom = UDim.new(0, 10)

    -- STATUS CARD
    local statCard = Instance.new("Frame", body)
    statCard.Size             = UDim2.new(1, 0, 0, 34)
    statCard.Position         = UDim2.new(0,0,0,0)
    statCard.BackgroundColor3 = BG[3]
    statCard.BorderSizePixel  = 0
    statCard.BackgroundTransparency = 1
    newCorner(statCard, 10)
    newStroke(statCard, Color3.fromRGB(20,20,50), 1)

    -- Hexágono status (frame hexagonal simulado)
    local hexFrame = Instance.new("Frame", statCard)
    hexFrame.Size             = UDim2.new(0, 30, 0, 30)
    hexFrame.Position         = UDim2.new(0, 8, 0.5, -15)
    hexFrame.BackgroundColor3 = Color3.fromRGB(0, 30, 24)
    hexFrame.BorderSizePixel  = 0
    newCorner(hexFrame, 99)
    newStroke(hexFrame, T.accent, 1, 0.6)

    local statusDot = Instance.new("Frame", hexFrame)
    statusDot.Size             = UDim2.new(0, 10, 0, 10)
    statusDot.Position         = UDim2.new(0.5,-5,0.5,-5)
    statusDot.BackgroundColor3 = OFF
    statusDot.BorderSizePixel  = 0
    newCorner(statusDot, 99)

    local statLbl = newMono(statCard, "SISTEMA", 8, DIMMED)
    statLbl.Size     = UDim2.new(0, 80, 0, 10)
    statLbl.Position = UDim2.new(0, 46, 0, 5)
    statLbl.TextTransparency = 1

    local statVal = newMono(statCard, "OFFLINE", 12, OFF)
    statVal.Size     = UDim2.new(0, 100, 0, 14)
    statVal.Position = UDim2.new(0, 46, 0, 17)
    statVal.TextTransparency = 1

    -- Waveform (frames simulando onda)
    local waveWrap = Instance.new("Frame", statCard)
    waveWrap.Size             = UDim2.new(0, 55, 0, 22)
    waveWrap.Position         = UDim2.new(1, -63, 0.5, -11)
    waveWrap.BackgroundTransparency = 1
    waveWrap.ClipsDescendants = true
    local waveBars = {}
    local waveHeights = {6,11,18,14,8,16,12,9,17,13,7,15,10}
    for i, h in ipairs(waveHeights) do
        local bar = Instance.new("Frame", waveWrap)
        bar.Size             = UDim2.new(0, 2, 0, h)
        bar.Position         = UDim2.new(0, (i-1)*4, 0.5, -h/2)
        bar.BackgroundColor3 = OFF
        bar.BorderSizePixel  = 0
        newCorner(bar, 1)
        bar.BackgroundTransparency = 0.4
        table.insert(waveBars, bar)
    end

    -- MAIN TOGGLE BUTTON
    local mainBtn = newBtn(body, "⬡  OFFLINE", 12, OFF, Color3.fromRGB(28,8,16))
    mainBtn.Name    = "MainBtn"
    mainBtn.Size    = UDim2.new(1, 0, 0, 42)
    mainBtn.Position = UDim2.new(0, 0, 0, 40)
    mainBtn.TextTransparency = 1
    mainBtn.BackgroundTransparency = 1
    newCorner(mainBtn, 11)
    local mainBtnStroke = newStroke(mainBtn, Color3.fromRGB(80,20,30), 1)

    -- ── SPEED ──
    local secSpd = newMono(body, "// VELOCIDADE", 8, DIMMED)
    secSpd.Size     = UDim2.new(1,0,0,10)
    secSpd.Position = UDim2.new(0,0,0,90)
    secSpd.TextTransparency = 1

    local speedNum = newMono(body, tostring(self.State.currentSpeed), 28, T.accent2)
    speedNum.Size     = UDim2.new(0, 80, 0, 30)
    speedNum.Position = UDim2.new(0, 0, 0, 103)
    speedNum.TextTransparency = 1
    speedNum.Font = Enum.Font.Code

    local speedUnit = newMono(body, "WLK/S", 8, DIMMED)
    speedUnit.Size     = UDim2.new(0, 50, 0, 10)
    speedUnit.Position = UDim2.new(0, 2, 0, 132)
    speedUnit.TextTransparency = 1

    local speedRange = newMono(body, "08 ──── 100", 8, Color3.fromRGB(35,35,65))
    speedRange.Size     = UDim2.new(0, 100, 0, 10)
    speedRange.Position = UDim2.new(1,-100, 0, 132)
    speedRange.TextXAlignment = Enum.TextXAlignment.Right
    speedRange.TextTransparency = 1

    -- Slider track
    local sliderBg = Instance.new("Frame", body)
    sliderBg.Size             = UDim2.new(1, 0, 0, 5)
    sliderBg.Position         = UDim2.new(0, 0, 0, 147)
    sliderBg.BackgroundColor3 = Color3.fromRGB(18,18,38)
    sliderBg.BorderSizePixel  = 0
    sliderBg.BackgroundTransparency = 1
    newCorner(sliderBg, 3)

    local sliderFill = Instance.new("Frame", sliderBg)
    sliderFill.Size             = UDim2.new(
        (self.State.currentSpeed - self.Config.MIN_SPEED) /
        (self.Config.MAX_SPEED   - self.Config.MIN_SPEED), 0, 1, 0)
    sliderFill.BackgroundColor3 = T.accent2
    sliderFill.BorderSizePixel  = 0
    newCorner(sliderFill, 3)

    local sliderThumb = Instance.new("TextButton", sliderBg)
    sliderThumb.Text             = ""
    sliderThumb.BackgroundColor3 = Color3.fromRGB(230,235,255)
    sliderThumb.Size             = UDim2.new(0, 16, 0, 16)
    sliderThumb.Position         = UDim2.new(sliderFill.Size.X.Scale, -8, 0.5, -8)
    sliderThumb.BorderSizePixel  = 0
    sliderThumb.BackgroundTransparency = 1
    newCorner(sliderThumb, 99)

    -- ── TEMA ──
    local secTheme = newMono(body, "// TEMA", 8, DIMMED)
    secTheme.Size     = UDim2.new(1,0,0,10)
    secTheme.Position = UDim2.new(0,0,0,170)
    secTheme.TextTransparency = 1

    local themeRow = Instance.new("Frame", body)
    themeRow.Size               = UDim2.new(1,0,0,18)
    themeRow.Position           = UDim2.new(0,0,0,183)
    themeRow.BackgroundTransparency = 1

    local themeDots = {}
    for i, th in ipairs(self.Themes) do
        local dot = Instance.new("TextButton", themeRow)
        dot.Size             = UDim2.new(0, 16, 0, 16)
        dot.Position         = UDim2.new(0, (i-1)*22, 0.5, -8)
        dot.BackgroundColor3 = th.accent
        dot.Text             = ""
        dot.BorderSizePixel  = 0
        dot.BackgroundTransparency = 1
        newCorner(dot, 99)
        if i == self.CurrentTheme then
            newStroke(dot, Color3.fromRGB(255,255,255), 2, 0.3)
        else
            newStroke(dot, Color3.fromRGB(255,255,255), 1, 0.85)
        end
        table.insert(themeDots, dot)
    end

    local themeAccentLbl = newMono(themeRow, "ACCENT", 8, DIMMED)
    themeAccentLbl.Size     = UDim2.new(0, 60, 1, 0)
    themeAccentLbl.Position = UDim2.new(1,-60,0,0)
    themeAccentLbl.TextXAlignment = Enum.TextXAlignment.Right
    themeAccentLbl.TextTransparency = 1

    -- ── MODULES ──
    local secMod = newMono(body, "// MÓDULOS", 8, DIMMED)
    secMod.Size     = UDim2.new(1,0,0,10)
    secMod.Position = UDim2.new(0,0,0,207)
    secMod.TextTransparency = 1

    local function makeModRow(yPos, iconChar, label, isOn, barColor)
        local row = Instance.new("Frame", body)
        row.Size             = UDim2.new(1, 0, 0, 28)
        row.Position         = UDim2.new(0, 0, 0, yPos)
        row.BackgroundColor3 = BG[3]
        row.BorderSizePixel  = 0
        row.BackgroundTransparency = 1
        newCorner(row, 9)
        newStroke(row, Color3.fromRGB(18,18,45), 1)

        local ic = newMono(row, iconChar, 13, barColor)
        ic.Size     = UDim2.new(0, 26, 0, 26)
        ic.Position = UDim2.new(0, 4, 0.5, -13)
        ic.TextXAlignment = Enum.TextXAlignment.Center
        ic.TextTransparency = 1

        local lbl = newLabel(row, label, 12, WHITE, Enum.Font.GothamBold)
        lbl.Size     = UDim2.new(0.5, 0, 1, 0)
        lbl.Position = UDim2.new(0, 34, 0, 0)
        lbl.TextTransparency = 1

        -- Mini barra de progresso
        local barBg = Instance.new("Frame", row)
        barBg.Size             = UDim2.new(0, 38, 0, 3)
        barBg.Position         = UDim2.new(1, -90, 0.5, -1)
        barBg.BackgroundColor3 = Color3.fromRGB(20,20,40)
        barBg.BorderSizePixel  = 0
        newCorner(barBg, 2)
        local barFill = Instance.new("Frame", barBg)
        barFill.Size             = UDim2.new(isOn and 1 or 0, 0, 1, 0)
        barFill.BackgroundColor3 = barColor
        barFill.BorderSizePixel  = 0
        newCorner(barFill, 2)

        -- Pill ON/OFF
        local pill = newBtn(row, isOn and "ON" or "OFF", 8,
            isOn and T.accent or DIMMED,
            isOn and Color3.fromRGB(0,25,20) or Color3.fromRGB(15,15,28))
        pill.Size     = UDim2.new(0, 36, 0, 16)
        pill.Position = UDim2.new(1, -42, 0.5, -8)
        pill.BackgroundTransparency = 1
        newCorner(pill, 4)
        newStroke(pill,
            isOn and Color3.fromRGB(0,70,55) or Color3.fromRGB(30,30,60), 1)

        return row, pill, barFill, ic
    end

    local jumpRow, jumpPill, jumpBarFill, jumpIc =
        makeModRow(220, "↑", "Auto-Jump",  self.State.jumpEnabled, T.accent)
    local humanRow, humanPill, humanBarFill, humanIc =
        makeModRow(253, "~", "Humanizer",  self.State.humanizeOn,  T.mod)

    -- BOTTOM BAR
    local bottomLine = Instance.new("Frame", body)
    bottomLine.Size             = UDim2.new(1,0,0,1)
    bottomLine.Position         = UDim2.new(0,0,1,-22)
    bottomLine.BackgroundColor3 = Color3.fromRGB(18,18,40)
    bottomLine.BorderSizePixel  = 0

    local hintLbl = newMono(body, "[R] TOGGLE_", 8, Color3.fromRGB(35,35,65))
    hintLbl.Size     = UDim2.new(0.5,0,0,12)
    hintLbl.Position = UDim2.new(0,0,1,-14)
    hintLbl.TextTransparency = 1

    local verLbl = newMono(body, "CYBR·v5·ULTRA", 7, Color3.fromRGB(70,40,120))
    verLbl.Size     = UDim2.new(0.5,0,0,12)
    verLbl.Position = UDim2.new(0.5,0,1,-14)
    verLbl.TextXAlignment = Enum.TextXAlignment.Right
    verLbl.TextTransparency = 1

    -- ── GUARDA REFERÊNCIAS ──
    self.Gui           = gui
    self.Panel         = panel
    self.PanelStroke   = panelStroke
    self.Body          = body
    self.IconDot       = iconDot
    self.IconWrap      = iconWrap
    self.StatusDot     = statusDot
    self.StatVal       = statVal
    self.StatCard      = statCard
    self.WaveBars      = waveBars
    self.MainBtn       = mainBtn
    self.MainBtnStroke = mainBtnStroke
    self.SpeedNum      = speedNum
    self.SliderBg      = sliderBg
    self.SliderFill    = sliderFill
    self.SliderThumb   = sliderThumb
    self.ThemeDots     = themeDots
    self.JumpRow       = jumpRow
    self.JumpPill      = jumpPill
    self.JumpBarFill   = jumpBarFill
    self.JumpIc        = jumpIc
    self.HumanRow      = humanRow
    self.HumanPill     = humanPill
    self.HumanBarFill  = humanBarFill
    self.HumanIc       = humanIc
    self.MinBtn        = minBtn
    self.CloseBtn      = closeBtn
    self.TitleLbl      = titleLbl
    self.SubLbl        = subLbl

    -- Lista de elementos de texto para fade
    self._fadeText = {
        titleLbl, subLbl, minBtn, closeBtn,
        statLbl, statVal, secSpd, speedNum,
        speedUnit, speedRange, secTheme, themeAccentLbl,
        secMod, hintLbl, verLbl
    }
    self._fadeBg = { statCard, mainBtn, sliderBg, jumpRow, humanRow }
    self._fadeThemeDots = themeDots
end

-- ══════════════════════════════════════════════
--   ANIMAÇÕES DE ABERTURA / FECHAMENTO
-- ══════════════════════════════════════════════
function AutoWalk:_animOpen()
    local t  = self.Config.TWEEN_OPEN
    local e  = TweenInfo.new(t, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)
    local e2 = TweenInfo.new(t*.65, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

    self.Panel.Size = UDim2.new(0, 275, 0, 285)
    TweenService:Create(self.Panel, e, {
        BackgroundTransparency = 0,
        Size = UDim2.new(0, 300, 0, 310),
    }):Play()
    TweenService:Create(self.PanelStroke, e2, { Transparency = 0.78 }):Play()

    task.delay(t * 0.2, function()
        for _, el in ipairs(self._fadeText) do
            TweenService:Create(el, e2, { TextTransparency = 0 }):Play()
        end
        for _, el in ipairs(self._fadeBg) do
            TweenService:Create(el, e2, { BackgroundTransparency = 0 }):Play()
        end
        for _, dot in ipairs(self._fadeThemeDots) do
            TweenService:Create(dot, e2, { BackgroundTransparency = 0 }):Play()
        end
        -- Thumb e fills do slider
        TweenService:Create(self.SliderThumb, e2, { BackgroundTransparency = 0 }):Play()
        TweenService:Create(self.SliderBg, e2, { BackgroundTransparency = 0 }):Play()
        -- Pills
        TweenService:Create(self.JumpPill, e2, {
            BackgroundTransparency = 0, TextTransparency = 0
        }):Play()
        TweenService:Create(self.HumanPill, e2, {
            BackgroundTransparency = 0, TextTransparency = 0
        }):Play()
        -- Ícones dos mods
        TweenService:Create(self.JumpIc, e2, { TextTransparency = 0 }):Play()
        TweenService:Create(self.HumanIc, e2, { TextTransparency = 0 }):Play()
        -- Wave bars
        for _, bar in ipairs(self.WaveBars) do
            TweenService:Create(bar, e2, { BackgroundTransparency = 0.4 }):Play()
        end
    end)
end

function AutoWalk:_animClose(cb)
    local t = self.Config.TWEEN_CLOSE
    local e = TweenInfo.new(t, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
    TweenService:Create(self.Panel, e, {
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 275, 0, 285),
    }):Play()
    TweenService:Create(self.PanelStroke, e, { Transparency = 1 }):Play()
    task.delay(t + 0.05, function() if cb then cb() end end)
end

-- ══════════════════════════════════════════════
--   MINIMIZAR
-- ══════════════════════════════════════════════
function AutoWalk:_minimize()
    self.State.minimized = not self.State.minimized
    local ease = TweenInfo.new(0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)
    if self.State.minimized then
        self.MinBtn.Text = "□"
        TweenService:Create(self.Panel, ease, { Size = UDim2.new(0,300,0,40) }):Play()
    else
        self.MinBtn.Text = "─"
        TweenService:Create(self.Panel, ease, { Size = UDim2.new(0,300,0,310) }):Play()
    end
end

-- ══════════════════════════════════════════════
--   TOGGLE VISUAL PRINCIPAL
-- ══════════════════════════════════════════════
function AutoWalk:_updateMainVisual()
    local T    = self.Themes[self.CurrentTheme]
    local ease = TweenInfo.new(0.22, Enum.EasingStyle.Quad)
    local OFF  = Color3.fromRGB(255,45,107)

    if self.State.enabled then
        self.MainBtn.Text = "⬡  ONLINE"
        TweenService:Create(self.MainBtn, ease, {
            TextColor3       = T.accent,
            BackgroundColor3 = Color3.fromRGB(0, 28, 22),
        }):Play()
        TweenService:Create(self.MainBtnStroke, ease,
            { Color = Color3.fromRGB(0,80,65) }):Play()
        TweenService:Create(self.StatusDot, ease, { BackgroundColor3 = T.accent }):Play()
        TweenService:Create(self.IconDot,   ease, { BackgroundColor3 = T.accent }):Play()
        TweenService:Create(self.StatVal,   ease, { TextColor3 = T.accent }):Play()
        self.StatVal.Text = "RUNNING"
        -- Waveform: muda cor para accent
        for _, bar in ipairs(self.WaveBars) do
            TweenService:Create(bar, ease, { BackgroundColor3 = T.accent }):Play()
        end
    else
        self.MainBtn.Text = "⬡  OFFLINE"
        TweenService:Create(self.MainBtn, ease, {
            TextColor3       = OFF,
            BackgroundColor3 = Color3.fromRGB(28, 8, 16),
        }):Play()
        TweenService:Create(self.MainBtnStroke, ease,
            { Color = Color3.fromRGB(80,20,30) }):Play()
        TweenService:Create(self.StatusDot, ease, { BackgroundColor3 = OFF }):Play()
        TweenService:Create(self.IconDot,   ease, { BackgroundColor3 = OFF }):Play()
        TweenService:Create(self.StatVal,   ease, { TextColor3 = OFF }):Play()
        self.StatVal.Text = "OFFLINE"
        for _, bar in ipairs(self.WaveBars) do
            TweenService:Create(bar, ease, { BackgroundColor3 = OFF }):Play()
        end
    end
end

-- ══════════════════════════════════════════════
--   PILL TOGGLE
-- ══════════════════════════════════════════════
function AutoWalk:_togglePill(pill, barFill, state, barColor)
    local T    = self.Themes[self.CurrentTheme]
    local ease = TweenInfo.new(0.18, Enum.EasingStyle.Quad)
    local DIMMED = Color3.fromRGB(40,40,70)
    pill.Text = state and "ON" or "OFF"
    TweenService:Create(pill, ease, {
        TextColor3       = state and T.accent or DIMMED,
        BackgroundColor3 = state and Color3.fromRGB(0,25,20) or Color3.fromRGB(15,15,28),
    }):Play()
    TweenService:Create(barFill, ease, {
        Size = UDim2.new(state and 1 or 0, 0, 1, 0),
        BackgroundColor3 = barColor,
    }):Play()
end

-- ══════════════════════════════════════════════
--   APLICAR TEMA
-- ══════════════════════════════════════════════
function AutoWalk:_applyTheme(idx)
    self.CurrentTheme = idx
    local T    = self.Themes[idx]
    local ease = TweenInfo.new(0.3, Enum.EasingStyle.Quad)

    -- Corners
    for _, part in ipairs(self._cornerParts) do
        TweenService:Create(part, ease, { BackgroundColor3 = T.accent }):Play()
    end
    -- Anel do ícone
    TweenService:Create(self.IconWrap, ease,
        { BackgroundColor3 = Color3.fromRGB(0,35,28) }):Play()

    -- Speed number
    TweenService:Create(self.SpeedNum, ease, { TextColor3 = T.accent2 }):Play()
    TweenService:Create(self.SliderFill, ease, { BackgroundColor3 = T.accent2 }):Play()

    -- Atualiza visual se ativo
    if self.State.enabled then
        self:_updateMainVisual()
    end

    -- Swatches: destaca o selecionado
    for i, dot in ipairs(self.ThemeDots) do
        local s = dot:FindFirstChildOfClass("UIStroke")
        if s then
            TweenService:Create(s, ease, {
                Transparency = (i == idx) and 0.3 or 0.85,
                Thickness    = (i == idx) and 2 or 1,
            }):Play()
        end
    end

    -- Re-aplica pills
    self:_togglePill(self.JumpPill,  self.JumpBarFill,  self.State.jumpEnabled,  T.accent)
    self:_togglePill(self.HumanPill, self.HumanBarFill, self.State.humanizeOn,   T.mod)

    self:_save()
end

-- ══════════════════════════════════════════════
--   TOGGLE WALK
-- ══════════════════════════════════════════════
function AutoWalk:_toggle()
    if self.State.debounce then return end
    self.State.debounce = true
    self.State.enabled  = not self.State.enabled
    self:_updateMainVisual()

    if self.State.hum then
        if self.State.enabled then
            self.State.originalSpeed  = self.State.hum.WalkSpeed
            self.State.hum.WalkSpeed  = self.State.currentSpeed
            self.State.hum.AutoRotate = true
        else
            if self.State.originalSpeed then
                self.State.hum.WalkSpeed = self.State.originalSpeed
            end
        end
    end

    task.delay(self.Config.DEBOUNCE_TIME, function()
        self.State.debounce = false
    end)
end

-- ══════════════════════════════════════════════
--   WAVEFORM ANIMADA (loop visual)
-- ══════════════════════════════════════════════
function AutoWalk:_updateWave(dt)
    self.State.waveTimer = self.State.waveTimer + dt * 6
    local t = self.State.waveTimer
    for i, bar in ipairs(self.WaveBars) do
        local phase  = i * 0.5
        local height = self.State.enabled
            and (8 + math.abs(math.sin(t + phase)) * 14)
            or  (3 + math.abs(math.sin(t * 0.3 + phase)) * 4)
        bar.Size     = UDim2.new(0, 2, 0, height)
        bar.Position = UDim2.new(0, (i-1)*4, 0.5, -height/2)
    end
end

-- ══════════════════════════════════════════════
--   GLITCH NO SPEED NUM
-- ══════════════════════════════════════════════
function AutoWalk:_glitchSpeed(dt)
    if not self.State.enabled then return end
    self.State.glitchTimer = self.State.glitchTimer + dt
    if self.State.glitchTimer >= self.Config.GLITCH_INTERVAL then
        self.State.glitchTimer = 0
        local orig = self.SpeedNum.Text
        local T    = self.Themes[self.CurrentTheme]
        TweenService:Create(self.SpeedNum,
            TweenInfo.new(0.05), { TextColor3 = Color3.fromRGB(255,45,107) }
        ):Play()
        self.SpeedNum.Text = tostring(math.random(10,99))
        task.delay(0.09, function()
            self.SpeedNum.Text = orig
            TweenService:Create(self.SpeedNum,
                TweenInfo.new(0.1), { TextColor3 = T.accent2 }
            ):Play()
        end)
    end
end

-- ══════════════════════════════════════════════
--   SLIDER DRAG
-- ══════════════════════════════════════════════
function AutoWalk:_setupSlider()
    local dragging = false
    local cfg = self.Config

    local function apply(clientX)
        local abs  = self.SliderBg.AbsolutePosition
        local size = self.SliderBg.AbsoluteSize
        local rat  = math.clamp((clientX - abs.X) / size.X, 0, 1)
        local spd  = math.floor(cfg.MIN_SPEED + rat * (cfg.MAX_SPEED - cfg.MIN_SPEED))
        self.State.currentSpeed    = spd
        self.SpeedNum.Text         = tostring(spd)
        self.SliderFill.Size       = UDim2.new(rat, 0, 1, 0)
        self.SliderThumb.Position  = UDim2.new(rat, -8, 0.5, -8)
        if self.State.hum and self.State.enabled then
            self.State.hum.WalkSpeed = spd
        end
        -- Flash rápido no número
        local T = self.Themes[self.CurrentTheme]
        self.SpeedNum.TextColor3 = Color3.fromRGB(255,45,107)
        task.delay(0.07, function()
            self.SpeedNum.TextColor3 = T.accent2
        end)
        self:_save()
    end

    local c1 = self.SliderThumb.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true end
    end)
    local c2 = self.SliderBg.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true; apply(i.Position.X)
        end
    end)
    local c3 = UserInputService.InputChanged:Connect(function(i)
        if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then
            apply(i.Position.X)
        end
    end)
    local c4 = UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    for _, c in ipairs({c1,c2,c3,c4}) do
        table.insert(self.State.connections, c)
    end
end

-- ══════════════════════════════════════════════
--   HUMANIZER
-- ══════════════════════════════════════════════
function AutoWalk:_humanize(dt)
    if not self.State.humanizeOn or not self.State.hum then return end
    self.State.humanizeTimer = self.State.humanizeTimer - dt
    if self.State.humanizeTimer <= 0 then
        local cfg = self.Config
        self.State.humanizeTimer = cfg.HUMANIZE_MIN +
            math.random() * (cfg.HUMANIZE_MAX - cfg.HUMANIZE_MIN)
        local v = (math.random() * 2 - 1) * cfg.SPEED_VARIANCE
        self.State.hum.WalkSpeed = math.clamp(
            self.State.currentSpeed + v, cfg.MIN_SPEED, cfg.MAX_SPEED
        )
    end
end

-- ══════════════════════════════════════════════
--   AUTO JUMP
-- ══════════════════════════════════════════════
function AutoWalk:_autoJump()
    if not self.State.jumpEnabled then return end
    local hrp = self.State.hrp
    local hum = self.State.hum
    if not hrp or not hum or hum.Health <= 0 then return end
    if tick() - self.State.lastJumpTime < 0.6 then return end

    local fwd    = hrp.CFrame.LookVector
    local origin = hrp.Position
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = { player.Character }
    params.FilterType = Enum.RaycastFilterType.Exclude

    local hitLow  = workspace:Raycast(origin + Vector3.new(0,-1.5,0), fwd*3.5, params)
    local hitMid  = workspace:Raycast(origin + Vector3.new(0, 0.5,0), fwd*3.5, params)
    local hitHigh = workspace:Raycast(origin + Vector3.new(0, 3.0,0), fwd*3.5, params)

    if (hitLow or hitMid) and not hitHigh then
        hum.Jump = true
        self.State.lastJumpTime = tick()
    end
end

-- ══════════════════════════════════════════════
--   CACHE PERSONAGEM
-- ══════════════════════════════════════════════
function AutoWalk:_cacheChar(char)
    task.defer(function()
        self.State.hrp          = char:WaitForChild("HumanoidRootPart", 8)
        self.State.hum          = char:WaitForChild("Humanoid", 8)
        self.State.originalSpeed = self.State.hum and self.State.hum.WalkSpeed or 16
        if self.State.enabled and self.State.hum then
            self.State.hum.WalkSpeed = self.State.currentSpeed
        end
    end)
end

-- ══════════════════════════════════════════════
--   CLEANUP
-- ══════════════════════════════════════════════
function AutoWalk:_cleanup()
    self.State.enabled = false
    if self.State.hum and self.State.originalSpeed then
        self.State.hum.WalkSpeed = self.State.originalSpeed
    end
    for _, c in ipairs(self.State.connections) do
        if typeof(c) == "RBXScriptConnection" and c.Connected then
            c:Disconnect()
        end
    end
    self.State.connections = {}
    ContextActionService:UnbindAction(self.Config.ACTION_NAME)
    if self.Gui and self.Gui.Parent then self.Gui:Destroy() end
end

-- ══════════════════════════════════════════════
--   INIT
-- ══════════════════════════════════════════════
function AutoWalk:Init()
    self:_load()
    self:_buildGui()
    self:_animOpen()
    self:_setupSlider()

    -- Personagem
    if player.Character then self:_cacheChar(player.Character) end
    local c1 = player.CharacterAdded:Connect(function(char)
        self.State.hrp = nil; self.State.hum = nil
        self:_cacheChar(char)
    end)
    table.insert(self.State.connections, c1)

    -- ContextActionService (PC + Mobile)
    ContextActionService:BindAction(
        self.Config.ACTION_NAME,
        function(_, s, _)
            if s == Enum.UserInputState.Begin then self:_toggle() end
        end,
        true, self.Config.TOGGLE_KEY
    )
    ContextActionService:SetTitle(self.Config.ACTION_NAME, "Walk")
    ContextActionService:SetPosition(
        self.Config.ACTION_NAME, UDim2.new(0.5,0,0.85,0))

    -- Conexões GUI
    local function conn(signal, fn)
        table.insert(self.State.connections, signal:Connect(fn))
    end

    conn(self.MainBtn.MouseButton1Click,  function() self:_toggle() end)
    conn(self.MinBtn.MouseButton1Click,   function() self:_minimize() end)
    conn(self.CloseBtn.MouseButton1Click, function()
        self:_animClose(function() self:_cleanup() end)
    end)
    conn(self.JumpRow.InputBegan, function(i)
        if i.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
        self.State.jumpEnabled = not self.State.jumpEnabled
        self:_togglePill(self.JumpPill, self.JumpBarFill,
            self.State.jumpEnabled, self.Themes[self.CurrentTheme].accent)
        self:_save()
    end)
    conn(self.HumanRow.InputBegan, function(i)
        if i.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
        self.State.humanizeOn = not self.State.humanizeOn
        self:_togglePill(self.HumanPill, self.HumanBarFill,
            self.State.humanizeOn, self.Themes[self.CurrentTheme].mod)
        self:_save()
    end)

    -- Swatches de tema
    for i, dot in ipairs(self.ThemeDots) do
        conn(dot.MouseButton1Click, function() self:_applyTheme(i) end)
    end

    -- ── LOOP PRINCIPAL ──
    conn(RunService.RenderStepped, function(dt)
        -- Waveform (sempre anima, mesmo quando off — mais suave)
        self:_updateWave(dt)
        -- Glitch no speed
        self:_glitchSpeed(dt)

        -- Early return se inativo
        if not self.State.enabled then return end
        local hrp = self.State.hrp
        local hum = self.State.hum
        if not hrp or not hum or hum.Health <= 0 then return end

        -- Movimento
        local fwd = hrp.CFrame.LookVector
        hum:Move(Vector3.new(fwd.X, 0, fwd.Z), false)

        -- Módulos
        self:_humanize(dt)
        self:_autoJump()
    end)

    print("[AutoWalk v5.0 ULTRA] ✅ Online — Tema: "..self.Themes[self.CurrentTheme].name)
end

-- ══════════════════════════════════════════════
--   START
-- ══════════════════════════════════════════════
AutoWalk:Init()
