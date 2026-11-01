-- Potasium.lua — minimal executor UI library for Potassium
-- Load: local Potasium = loadstring(game:HttpGet("<your-url>"))()

local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local HttpService      = game:GetService("HttpService")
local Players          = game:GetService("Players")

local Potasium = {}
local Tab      = {}
Tab.__index = Tab

local Theme = {
    Background = Color3.fromRGB(20, 20, 25),
    Sidebar    = Color3.fromRGB(15, 15, 20),
    Accent     = Color3.fromRGB(80, 200, 120),   -- "potassium" green
    Text       = Color3.fromRGB(240, 240, 240),
    SubText    = Color3.fromRGB(150, 150, 160),
    Element    = Color3.fromRGB(35, 35, 45),
    Stroke     = Color3.fromRGB(50, 50, 60),
}
Potasium.Theme = Theme

-- ---------- helpers ----------

local function new(class, props)
    local inst = Instance.new(class)
    for k, v in pairs(props or {}) do inst[k] = v end
    return inst
end

local function corner(parent, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 6)
    c.Parent = parent
    return c
end

local function stroke(parent, color, thickness)
    local s = Instance.new("UIStroke")
    s.Color = color or Theme.Stroke
    s.Thickness = thickness or 1
    s.Parent = parent
    return s
end

local function safeCallback(fn)
    if type(fn) == "function" then return fn end
    return function() end
end

-- ---------- Window ----------

function Potasium:CreateWindow(config)
    config = config or {}
    local self = setmetatable({}, Potasium)
    self.Title        = config.Title or "Potasium"
    self.Subtitle     = config.Subtitle or ""
    self.ConfigFolder = config.ConfigFolder or "PotasiumConfig"
    self.Flags        = {}
    self.FlagObjects  = {}
    self.Tabs         = {}
    self.SelectedTab  = nil
    self._order       = 0
    self._conns       = {}

    function self._track(conn)
        if conn then table.insert(self._conns, conn) end
        return conn
    end

    local gui = new("ScreenGui", {
        Name = "PotasiumUI",
        ResetOnSpawn = false,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true,
    })

    -- Prefer gethui() (Potassium / UNC), then CoreGui, then PlayerGui.
    local parented = false
    if type(gethui) == "function" then
        local ok, res = pcall(gethui)
        if ok and res then
            gui.Parent = res
            parented = true
        end
    end
    if not parented then
        local ok = pcall(function() gui.Parent = game:GetService("CoreGui") end)
        if not ok then
            gui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui")
        end
    end

    local main = new("Frame", {
        Name = "MainFrame",
        Parent = gui,
        Size = UDim2.fromOffset(560, 380),
        Position = UDim2.fromOffset(200, 200),
        BackgroundColor3 = Theme.Background,
        BorderSizePixel = 0,
    })
    corner(main, 10); stroke(main)
    self.Gui, self.MainFrame = gui, main

    -- Title bar
    local titleBar = new("Frame", {
        Name = "TitleBar",
        Parent = main,
        Size = UDim2.new(1, 0, 0, 48),
        BackgroundTransparency = 1,
    })
    corner(titleBar, 10)

    new("TextLabel", {
        Parent = titleBar, Size = UDim2.new(1, -80, 0, 18),
        Position = UDim2.fromOffset(15, 8), BackgroundTransparency = 1,
        Text = self.Title, TextColor3 = Theme.Text, TextSize = 16,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
    })
    if self.Subtitle ~= "" then
        new("TextLabel", {
            Parent = titleBar, Size = UDim2.new(1, -80, 0, 14),
            Position = UDim2.fromOffset(15, 26), BackgroundTransparency = 1,
            Text = self.Subtitle, TextColor3 = Theme.SubText, TextSize = 11,
            Font = Enum.Font.Gotham,
            TextXAlignment = Enum.TextXAlignment.Left,
        })
    end

    local close = new("TextButton", {
        Parent = titleBar, Size = UDim2.fromOffset(24, 24),
        Position = UDim2.new(1, -34, 0, 12),
        BackgroundColor3 = Theme.Element, BorderSizePixel = 0,
        Text = "X", TextColor3 = Theme.SubText, TextSize = 12,
        Font = Enum.Font.GothamBold, AutoButtonColor = false,
    })
    corner(close, 6)
    close.MouseButton1Click:Connect(function() self:Destroy() end)

    -- Dragging
    local dragging, dragStart, startPos
    self._track(titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging  = true
            dragStart = input.Position
            startPos  = main.Position
        end
    end))
    self._track(UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local d = input.Position - dragStart
            main.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y
            )
        end
    end))
    self._track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end))

    -- Sidebar
    local sidebar = new("Frame", {
        Name = "Sidebar", Parent = main,
        Position = UDim2.fromOffset(10, 52),
        Size = UDim2.new(0, 140, 1, -62),
        BackgroundColor3 = Theme.Sidebar, BorderSizePixel = 0,
    })
    corner(sidebar, 8)
    local sideList = Instance.new("UIListLayout")
    sideList.Parent = sidebar
    sideList.Padding = UDim.new(0, 4)
    sideList.SortOrder = Enum.SortOrder.LayoutOrder
    local sidePad = Instance.new("UIPadding")
    sidePad.Parent = sidebar
    sidePad.PaddingTop    = UDim.new(0, 6)
    sidePad.PaddingBottom = UDim.new(0, 6)
    sidePad.PaddingLeft   = UDim.new(0, 6)
    sidePad.PaddingRight  = UDim.new(0, 6)

    -- Page container
    local pages = new("Frame", {
        Name = "Pages", Parent = main,
        Position = UDim2.fromOffset(160, 52),
        Size = UDim2.new(1, -170, 1, -62),
        BackgroundTransparency = 1,
    })

    self.Sidebar, self.PageContainer = sidebar, pages
    return self
end

function Potasium:_nextOrder()
    self._order = self._order + 1
    return self._order
end

function Potasium:SelectTab(tab)
    if self.SelectedTab == tab then return end
    self.SelectedTab = tab
    for _, t in ipairs(self.Tabs) do
        local active = (t == tab)
        t.Page.Visible = active
        t.Button.BackgroundColor3 = active and Theme.Accent or Theme.Element
        t.Button.TextColor3       = active and Theme.Text  or Theme.SubText
    end
end

function Potasium:Tab(config)
    config = config or {}
    local name = config.Name or "Tab"
    local idx = #self.Tabs + 1

    local btn = new("TextButton", {
        Parent = self.Sidebar,
        Size = UDim2.new(1, 0, 0, 30),
        BackgroundColor3 = Theme.Element, BorderSizePixel = 0,
        Text = name, TextColor3 = Theme.SubText, TextSize = 13,
        Font = Enum.Font.Gotham, AutoButtonColor = false,
        LayoutOrder = idx,
    })
    corner(btn, 6)

    local page = new("ScrollingFrame", {
        Parent = self.PageContainer,
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1, BorderSizePixel = 0,
        Visible = false, ScrollBarThickness = 4,
        ScrollBarImageColor3 = Theme.Stroke,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
    })
    local layout = Instance.new("UIListLayout")
    layout.Parent = page
    layout.Padding = UDim.new(0, 8)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    local pad = Instance.new("UIPadding")
    pad.Parent = page
    pad.PaddingTop    = UDim.new(0, 4)
    pad.PaddingBottom = UDim.new(0, 4)
    pad.PaddingLeft   = UDim.new(0, 4)
    pad.PaddingRight  = UDim.new(0, 4)

    local tab = setmetatable({
        Name = name, Button = btn, Page = page, Window = self,
    }, Tab)
    table.insert(self.Tabs, tab)

    btn.MouseButton1Click:Connect(function() self:SelectTab(tab) end)
    if #self.Tabs == 1 then self:SelectTab(tab) end
    return tab
end

-- ---------- Elements ----------

function Tab:Button(config)
    config = config or {}
    local callback = safeCallback(config.Callback)

    local btn = new("TextButton", {
        Parent = self.Page,
        Size = UDim2.new(1, 0, 0, 34),
        BackgroundColor3 = Theme.Element, BorderSizePixel = 0,
        Text = config.Name or "Button",
        TextColor3 = Theme.Text, TextSize = 13,
        Font = Enum.Font.Gotham, AutoButtonColor = false,
        LayoutOrder = self.Window:_nextOrder(),
    })
    corner(btn, 6); stroke(btn)

    btn.MouseEnter:Connect(function() btn.BackgroundColor3 = Theme.Stroke  end)
    btn.MouseLeave:Connect(function() btn.BackgroundColor3 = Theme.Element end)
    btn.MouseButton1Click:Connect(function() pcall(callback) end)
    return btn
end

function Tab:Toggle(config)
    config = config or {}
    local name     = config.Name or "Toggle"
    local default  = config.Default and true or false
    local callback = safeCallback(config.Callback)
    local flag     = config.Flag

    local container = new("Frame", {
        Parent = self.Page,
        Size = UDim2.new(1, 0, 0, 36),
        BackgroundColor3 = Theme.Element, BorderSizePixel = 0,
        LayoutOrder = self.Window:_nextOrder(),
    })
    corner(container, 6); stroke(container)

    new("TextLabel", {
        Parent = container, Size = UDim2.new(1, -60, 1, 0),
        Position = UDim2.fromOffset(12, 0), BackgroundTransparency = 1,
        Text = name, TextColor3 = Theme.Text, TextSize = 13,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
    })

    local switch = new("Frame", {
        Parent = container, Size = UDim2.fromOffset(36, 18),
        Position = UDim2.new(1, -48, 0.5, -9),
        BackgroundColor3 = Theme.Stroke, BorderSizePixel = 0,
    })
    corner(switch, 9)

    local knob = new("Frame", {
        Parent = switch, Size = UDim2.fromOffset(14, 14),
        Position = UDim2.fromOffset(2, 2),
        BackgroundColor3 = Theme.Text, BorderSizePixel = 0,
    })
    corner(knob, 7)

    local state = default
    local toggle = {}

    local function paint(animate)
        local color = state and Theme.Accent or Theme.Stroke
        local pos   = UDim2.fromOffset(state and 20 or 2, 2)
        if animate then
            TweenService:Create(switch, TweenInfo.new(0.15), { BackgroundColor3 = color }):Play()
            TweenService:Create(knob,   TweenInfo.new(0.15), { Position = pos }):Play()
        else
            switch.BackgroundColor3 = color
            knob.Position = pos
        end
    end
    paint(false)

    local hit = new("TextButton", {
        Parent = container, Size = UDim2.fromScale(1, 1),
        BackgroundTransparency = 1, Text = "",
    })
    hit.MouseButton1Click:Connect(function()
        state = not state
        paint(true)
        if flag then self.Window.Flags[flag] = state end
        pcall(callback, state)
    end)

    function toggle:Set(v)
        state = v and true or false
        paint(true)
        if flag then self.Window.Flags[flag] = state end
        pcall(callback, state)
    end
    function toggle:Get() return state end

    if flag then
        self.Window.FlagObjects[flag] = toggle
        self.Window.Flags[flag] = state
    end
    return toggle
end

function Tab:Slider(config)
    config = config or {}
    local name     = config.Name or "Slider"
    local min      = config.Min or 0
    local max      = config.Max or 100
    local default  = config.Default or min
    local decimals = config.Decimals or 0
    local suffix   = config.Suffix or ""
    local callback = safeCallback(config.Callback)
    local flag     = config.Flag

    local container = new("Frame", {
        Parent = self.Page,
        Size = UDim2.new(1, 0, 0, 52),
        BackgroundColor3 = Theme.Element, BorderSizePixel = 0,
        LayoutOrder = self.Window:_nextOrder(),
    })
    corner(container, 6); stroke(container)

    new("TextLabel", {
        Parent = container, Size = UDim2.new(1, -80, 0, 20),
        Position = UDim2.fromOffset(12, 6), BackgroundTransparency = 1,
        Text = name, TextColor3 = Theme.Text, TextSize = 13,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
    })

    local valueLabel = new("TextLabel", {
        Parent = container, Size = UDim2.new(0, 70, 0, 20),
        Position = UDim2.new(1, -82, 0, 6), BackgroundTransparency = 1,
        Text = tostring(default) .. suffix,
        TextColor3 = Theme.Accent, TextSize = 12,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Right,
    })

    local bar = new("Frame", {
        Parent = container,
        Size = UDim2.new(1, -24, 0, 6),
        Position = UDim2.new(0, 12, 1, -22),
        BackgroundColor3 = Theme.Stroke, BorderSizePixel = 0,
    })
    corner(bar, 3)

    local fill = new("Frame", {
        Parent = bar, Size = UDim2.fromScale(0, 1),
        BackgroundColor3 = Theme.Accent, BorderSizePixel = 0,
    })
    corner(fill, 3)

    local knob = new("Frame", {
        Parent = bar, Size = UDim2.fromOffset(14, 14),
        Position = UDim2.new(0, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = Theme.Text, BorderSizePixel = 0,
        ZIndex = 2,
    })
    corner(knob, 7)

    local state = math.clamp(default, min, max)
    local slider = {}

    local function format(v)
        if decimals <= 0 then
            return tostring(math.floor(v + 0.5)) .. suffix
        end
        local m = 10 ^ decimals
        return tostring(math.floor(v * m + 0.5) / m) .. suffix
    end

    local function paint(fire)
        local alpha = (state - min) / math.max(max - min, 1e-6)
        fill.Size = UDim2.fromScale(alpha, 1)
        knob.Position = UDim2.new(alpha, 0, 0.5, 0)
        valueLabel.Text = format(state)
        if flag then self.Window.Flags[flag] = state end
        if fire then pcall(callback, state) end
    end
    paint(false)

    local function fromX(x)
        local abs, size = bar.AbsolutePosition, bar.AbsoluteSize
        local alpha = math.clamp((x - abs.X) / math.max(size.X, 1), 0, 1)
        return min + (max - min) * alpha
    end

    local dragging = false
    local hit = new("TextButton", {
        Parent = container,
        Size = UDim2.new(1, -24, 0, 22),
        Position = UDim2.new(0, 12, 1, -30),
        BackgroundTransparency = 1, Text = "",
    })
    hit.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            state = fromX(input.Position.X)
            paint(true)
        end
    end)
    self.Window._track(UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            state = fromX(input.Position.X)
            paint(true)
        end
    end))
    self.Window._track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end))

    function slider:Set(v)
        state = math.clamp(v, min, max)
        paint(true)
    end
    function slider:Get() return state end

    if flag then
        self.Window.FlagObjects[flag] = slider
        self.Window.Flags[flag] = state
    end
    return slider
end

-- ---------- Config (uses Potassium FileSystem API) ----------

function Potasium:SaveConfig(name)
    if type(writefile) ~= "function"
        or type(isfolder) ~= "function"
        or type(makefolder) ~= "function" then
        return false
    end
    return (pcall(function()
        if not isfolder(self.ConfigFolder) then
            makefolder(self.ConfigFolder)
        end
        writefile(
            self.ConfigFolder .. "/" .. name .. ".json",
            HttpService:JSONEncode(self.Flags)
        )
    end))
end

function Potasium:LoadConfig(name)
    if type(isfile) ~= "function" or type(readfile) ~= "function" then
        return false
    end
    local path = self.ConfigFolder .. "/" .. name .. ".json"
    if not isfile(path) then return false end
    local ok, data = pcall(function()
        return HttpService:JSONDecode(readfile(path))
    end)
    if not ok or type(data) ~= "table" then return false end
    for flag, value in pairs(data) do
        local obj = self.FlagObjects[flag]
        if obj and type(obj.Set) == "function" then
            pcall(function() obj:Set(value) end)
        end
    end
    return true
end

-- ---------- Cleanup ----------

function Potasium:Destroy()
    for _, c in ipairs(self._conns) do
        pcall(function() c:Disconnect() end)
    end
    self._conns = {}
    if self.Gui then
        pcall(function() self.Gui:Destroy() end)
    end
end

return Potasium
