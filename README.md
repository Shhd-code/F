-- ==========================================================
-- [SH Script] - Remote Hook & Copy System v2.0
-- ==========================================================
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer

-- ==========================================================
-- [CONFIG]
-- ==========================================================
local CONFIG = {
    FirstRunFile   = "SH_FirstRun.txt",
    RemoteCacheFile = "SH_RemoteCache.txt",
    NotifyTitle    = "SH Script",
    CommandDelay   = 0.03, -- تأخير بسيط جداً بين الأوامر (0 = أقصى سرعة، قد يسبب kick)
}

local COPY_COMMANDS = {
    ";worm",";ice",";kill",";re",";res",";logs",";clogs",";cmdbar",
    ";warm",";jc",";nv",
}

-- ==========================================================
-- [STORAGE HELPERS]
-- ==========================================================
local hasFS = (typeof(writefile) == "function") and (typeof(readfile) == "function") and (typeof(isfile) == "function")

local function fileExists(name) return hasFS and isfile(name) end
local function fileRead(name)
    if not hasFS then return nil end
    local ok, data = pcall(readfile, name)
    return ok and data or nil
end
local function fileWrite(name, data)
    if not hasFS then return false end
    local ok = pcall(writefile, name, data)
    return ok
end

-- ==========================================================
-- [STATE]
-- ==========================================================
local State = {
    isUnlocked       = false,
    firstRunDone     = fileExists(CONFIG.FirstRunFile),
    chatRemote       = nil,
    adminRemote      = nil,
    selectedPlayer   = nil,
    copyRunning      = false,
    copyThread       = nil,
    mainGui          = nil,
}

-- استرجاع الريموت المحفوظ سابقاً
local function loadCachedRemote()
    local data = fileRead(CONFIG.RemoteCacheFile)
    if not data then return end
    for line in data:gmatch("[^\r\n]+") do
        local kind, path = line:match("^(%w+)|(.+)$")
        if kind == "chat" then
            local obj = game:FindFirstChild(path, true)
            if not obj then
                local segs = {}
                for s in path:gmatch("[^%.]+") do table.insert(segs, s) end
                obj = game
                for _, seg in ipairs(segs) do
                    obj = obj and obj:FindFirstChild(seg)
                end
            end
            if obj then State.chatRemote = obj end
        elseif kind == "admin" then
            local obj = game:FindFirstChild(path, true)
            if not obj then
                local segs = {}
                for s in path:gmatch("[^%.]+") do table.insert(segs, s) end
                obj = game
                for _, seg in ipairs(segs) do
                    obj = obj and obj:FindFirstChild(seg)
                end
            end
            if obj then State.adminRemote = obj end
        end
    end
end

local function saveCachedRemotes()
    if not hasFS then return end
    local lines = {}
    if State.chatRemote then table.insert(lines, "chat|" .. State.chatRemote:GetFullName()) end
    if State.adminRemote then table.insert(lines, "admin|" .. State.adminRemote:GetFullName()) end
    fileWrite(CONFIG.RemoteCacheFile, table.concat(lines, "\n"))
end

loadCachedRemote()

-- ==========================================================
-- [NOTIFICATION]
-- ==========================================================
local function notify(title, text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title or CONFIG.NotifyTitle,
            Text = text or "",
            Duration = 3,
        })
    end)
end

-- ==========================================================
-- [REMOTE AUTO-DETECT]
-- ==========================================================
local function autoFindRemotes()
    local foundChat, foundAdmin = nil, nil
    local chatKeywords = { "chat", "say", "message", "msg", "sendmessage", "sendchat", "chatmsg" }
    local adminKeywords = { "admin", "command", "cmd", "execute", "adon", "adoniscmd", "admincmd" }

    local function containsAny(str, list)
        local low = str:lower()
        for _, k in ipairs(list) do
            if low:find(k, 1, true) then return true end
        end
        return false
    end

    local function scan(container)
        for _, obj in ipairs(container:GetDescendants()) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                local name = obj.Name
                if not foundChat and containsAny(name, chatKeywords) then
                    -- تجنب ريموتات الأدمن كشات
                    if not containsAny(name, adminKeywords) then
                        foundChat = obj
                    end
                end
                if not foundAdmin and containsAny(name, adminKeywords) then
                    foundAdmin = obj
                end
            end
        end
    end

    scan(ReplicatedStorage)
    scan(game:GetService("Workspace"))
    pcall(scan, game:GetService("ServerScriptService"))
    pcall(scan, Players)

    if foundChat then State.chatRemote = foundChat end
    if foundAdmin then State.adminRemote = foundAdmin end
    if foundChat or foundAdmin then saveCachedRemotes() end
    return foundChat, foundAdmin
end

-- ==========================================================
-- [COMMAND SENDER]
-- ==========================================================
local function fireRemote(remote, ...)
    if not remote then return false end
    local ok = pcall(function()
        if remote:IsA("RemoteEvent") then
            remote:FireServer(...)
        else
            remote:InvokeServer(...)
        end
    end)
    return ok
end

local function sendCommandWithName(cmd, targetName)
    -- نحاول إرسال الأوامر بعدة صيغ شائعة
    local targets = { State.chatRemote, State.adminRemote }
    for _, r in ipairs(targets) do
        if r then
            pcall(function()
                if r:IsA("RemoteEvent") then
                    r:FireServer(cmd .. " " .. targetName)
                    r:FireServer(cmd, targetName)
                    r:FireServer(targetName, cmd)
                else
                    r:InvokeServer(cmd .. " " .. targetName)
                    r:InvokeServer(cmd, targetName)
                end
            end)
        end
    end
end

local function stopCopy()
    if State.copyRunning then
        State.copyRunning = false
        if State.copyThread then
            pcall(task.cancel, State.copyThread)
            State.copyThread = nil
        end
        notify("SH Script", "🛑 تم إيقاف النسخ")
    end
end

local function startCopy(targetName)
    if State.copyRunning then stopCopy() end
    if not targetName or targetName == "" then
        notify("SH Script", "⚠️ اختر لاعباً أولاً")
        return
    end
    if not State.chatRemote and not State.adminRemote then
        notify("SH Script", "⚠️ لا يوجد ريموت، اضغط زر البحث")
        return
    end
    State.copyRunning = true
    notify("SH Script", "▶️ بدء النسخ على: " .. targetName)
    State.copyThread = task.spawn(function()
        while State.copyRunning do
            for _, cmd in ipairs(COPY_COMMANDS) do
                if not State.copyRunning then break end
                sendCommandWithName(cmd, targetName)
                if CONFIG.CommandDelay > 0 then task.wait(CONFIG.CommandDelay) end
            end
        end
    end)
end

-- ==========================================================
-- [HOOK: التقاط ريموت الشات عند كتابة "شهد" أو "shahd"]
-- ==========================================================
local oldNameCall
oldNameCall = hookmetamethod(game, "__namecall", function(self, ...)
    local method = getnamecallmethod()
    local args = { ... }
    if (not State.isUnlocked) and (method == "FireServer" or method == "InvokeServer") then
        for _, arg in ipairs(args) do
            if typeof(arg) == "string" then
                local low = arg:lower()
                if low:find("شهد") or low:find("shahd") then
                    if (self:IsA("RemoteEvent") or self:IsA("RemoteFunction")) then
                        State.chatRemote = self
                        saveCachedRemotes()
                        print("🚀 [SH] تم سحب ريموت الشات: " .. self:GetFullName())
                        State.isUnlocked = true
                        if State.onUnlock then State.onUnlock() end
                    end
                    break
                end
            end
        end
    end
    return oldNameCall(self, ...)
end)

-- ==========================================================
-- [UI HELPERS]
-- ==========================================================
local function new(className, props, children)
    local obj = Instance.new(className)
    for k, v in pairs(props or {}) do obj[k] = v end
    for _, c in ipairs(children or {}) do c.Parent = obj end
    return obj
end

local function corner(parent, r)
    return new("UICorner", { CornerRadius = UDim.new(0, r or 8), Parent = parent })
end

local function stroke(parent, color, thickness)
    return new("UIStroke", {
        Color = color or Color3.fromRGB(255,170,0),
        Thickness = thickness or 1,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        Parent = parent,
    })
end

local THEME = {
    bg        = Color3.fromRGB(15, 15, 18),
    panel     = Color3.fromRGB(24, 24, 28),
    panel2    = Color3.fromRGB(32, 32, 38),
    accent    = Color3.fromRGB(255, 170, 0),
    green     = Color3.fromRGB(0, 220, 130),
    red       = Color3.fromRGB(240, 70, 70),
    text      = Color3.fromRGB(235, 235, 235),
    subtext   = Color3.fromRGB(160, 160, 170),
}

-- ==========================================================
-- [WELCOME SCREEN - تظهر أول مرة فقط]
-- ==========================================================
local function showWelcome(onDone)
    local gui = new("ScreenGui", {
        Name = "SH_WelcomeGui",
        ResetOnSpawn = false,
        Parent = CoreGui,
    })
    local frame = new("Frame", {
        Size = UDim2.new(0, 400, 0, 200),
        Position = UDim2.new(0.5, -200, 0.5, -100),
        BackgroundColor3 = THEME.bg,
        BorderSizePixel = 0,
        Parent = gui,
    })
    corner(frame, 12); stroke(frame, THEME.accent, 1)

    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 45),
        BackgroundTransparency = 1,
        Text = "⚡ SH SCRIPT - REMOTE HOOK ⚡",
        TextColor3 = THEME.accent,
        TextSize = 19,
        Font = Enum.Font.FredokaOne,
        Parent = frame,
    })

    new("TextLabel", {
        Size = UDim2.new(1, -40, 0, 110),
        Position = UDim2.new(0, 20, 0, 45),
        BackgroundTransparency = 1,
        Text = "اكتب كلمة (شهد) أو (shahd) في الشات العام الآن\nسيتم سحب ريموت الشات تلقائياً وتفعيل السكربت.\n\n(هذه الشاشة تظهر مرة واحدة فقط)",
        TextColor3 = THEME.text,
        TextSize = 13,
        Font = Enum.Font.GothamMedium,
        TextWrapped = true,
        Parent = frame,
    })

    local status = new("TextLabel", {
        Size = UDim2.new(1, -40, 0, 20),
        Position = UDim2.new(0, 20, 1, -30),
        BackgroundTransparency = 1,
        Text = "⏳ بانتظار الكلمة...",
        TextColor3 = THEME.subtext,
        TextSize = 12,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = frame,
    })

    return gui, status
end

-- ==========================================================
-- [MAIN UI]
-- ==========================================================
local function buildMainUI()
    local gui = new("ScreenGui", {
        Name = "SH_MainGui",
        ResetOnSpawn = false,
        Parent = CoreGui,
    })
    State.mainGui = gui

    local main = new("Frame", {
        Size = UDim2.new(0, 460, 0, 340),
        Position = UDim2.new(0.5, -230, 0.5, -170),
        BackgroundColor3 = THEME.bg,
        BorderSizePixel = 0,
        Active = true,
        Draggable = true,
        Parent = gui,
    })
    corner(main, 14); stroke(main, THEME.accent, 1.5)

    -- الهيدر
    local header = new("Frame", {
        Size = UDim2.new(1, 0, 0, 42),
        BackgroundColor3 = THEME.panel,
        BorderSizePixel = 0,
        Parent = main,
    })
    corner(header, 14)
    new("TextLabel", {
        Size = UDim2.new(1, -60, 1, 0),
        Position = UDim2.new(0, 15, 0, 0),
        BackgroundTransparency = 1,
        Text = "⚡ SH SCRIPT",
        TextColor3 = THEME.accent,
        TextSize = 18,
        Font = Enum.Font.FredokaOne,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = header,
    })
    local closeBtn = new("TextButton", {
        Size = UDim2.new(0, 30, 0, 30),
        Position = UDim2.new(1, -38, 0, 6),
        BackgroundColor3 = THEME.red,
        Text = "✕",
        TextColor3 = Color3.new(1,1,1),
        TextSize = 16,
        Font = Enum.Font.GothamBold,
        BorderSizePixel = 0,
        Parent = header,
    })
    corner(closeBtn, 8)
    closeBtn.MouseButton1Click:Connect(function() gui.Enabled = false end)

    -- شريط التبويبات
    local tabBar = new("Frame", {
        Size = UDim2.new(1, -20, 0, 36),
        Position = UDim2.new(0, 10, 0, 50),
        BackgroundColor3 = THEME.panel,
        BorderSizePixel = 0,
        Parent = main,
    })
    corner(tabBar, 8)

    local function makeTab(text, xPos, xSize)
        local b = new("TextButton", {
            Size = UDim2.new(xSize, -4, 1, -6),
            Position = UDim2.new(xPos, 2, 0, 3),
            BackgroundColor3 = THEME.panel2,
            Text = text,
            TextColor3 = THEME.text,
            TextSize = 13,
            Font = Enum.Font.GothamBold,
            BorderSizePixel = 0,
            Parent = tabBar,
        })
        corner(b, 6)
        return b
    end

    local tabRemotes = makeTab("📡 Remotes", 0, 0.5)
    local tabCopy    = makeTab("📋 Copy",    0.5, 0.5)

    -- الحاويات
    local remotesPage = new("Frame", {
        Size = UDim2.new(1, -20, 1, -100),
        Position = UDim2.new(0, 10, 0, 92),
        BackgroundTransparency = 1,
        Parent = main,
    })
    local copyPage = new("Frame", {
        Size = UDim2.new(1, -20, 1, -100),
        Position = UDim2.new(0, 10, 0, 92),
        BackgroundTransparency = 1,
        Visible = false,
        Parent = main,
    })

    local function setActiveTab(which)
        remotesPage.Visible = (which == "remotes")
        copyPage.Visible    = (which == "copy")
        tabRemotes.BackgroundColor3 = (which == "remotes") and THEME.accent or THEME.panel2
        tabRemotes.TextColor3       = (which == "remotes") and Color3.new(0,0,0) or THEME.text
        tabCopy.BackgroundColor3    = (which == "copy")    and THEME.accent or THEME.panel2
        tabCopy.TextColor3          = (which == "copy")    and Color3.new(0,0,0) or THEME.text
    end
    setActiveTab("remotes")
    tabRemotes.MouseButton1Click:Connect(function() setActiveTab("remotes") end)
    tabCopy.MouseButton1Click:Connect(function() setActiveTab("copy") end)

    ------------------------------------------------------------
    -- [PAGE 1: REMOTES]
    ------------------------------------------------------------
    local chatStatus = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 22),
        Position = UDim2.new(0, 4, 0, 0),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = THEME.subtext,
        TextSize = 12,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })

    local adminStatus = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 22),
        Position = UDim2.new(0, 4, 0, 24),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = THEME.subtext,
        TextSize = 12,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })

    local function refreshRemoteStatus()
        chatStatus.Text = State.chatRemote
            and ("✅ Chat Remote: " .. State.chatRemote.Name)
            or  "❌ Chat Remote: غير موجود"
        chatStatus.TextColor3 = State.chatRemote and THEME.green or THEME.red
        adminStatus.Text = State.adminRemote
            and ("✅ Admin Remote: " .. State.adminRemote.Name)
            or  "❌ Admin Remote: غير موجود"
        adminStatus.TextColor3 = State.adminRemote and THEME.green or THEME.red
    end
    State.refreshRemoteStatus = refreshRemoteStatus
    refreshRemoteStatus()

    local function makeButton(text, color, xPos, xSize, yPos, ySize, parent)
        local b = new("TextButton", {
            Size = UDim2.new(xSize, -6, 0, ySize),
            Position = UDim2.new(xPos, 3, 0, yPos),
            BackgroundColor3 = color,
            Text = text,
            TextColor3 = Color3.new(1,1,1),
            TextSize = 14,
            Font = Enum.Font.GothamBold,
            BorderSizePixel = 0,
            Parent = parent or remotesPage,
        })
        corner(b, 8)
        return b
    end

    local searchBtn = makeButton("🔍 بحث عن الريموت", THEME.accent, 0, 0.5, 60, 38, remotesPage)
    searchBtn.TextColor3 = Color3.new(0,0,0)
    searchBtn.MouseButton1Click:Connect(function()
        local c, a = autoFindRemotes()
        refreshRemoteStatus()
        if c or a then
            notify("SH Script", "✅ تم إيجاد الريموتات وحفظها")
        else
            notify("SH Script", "⚠️ لم يتم إيجاد ريموتات")
        end
    end)

    local lockBtn = makeButton("🔒 قفل/فتح", THEME.panel2, 0.5, 0.5, 60, 38, remotesPage)
    lockBtn.MouseButton1Click:Connect(function()
        State.isUnlocked = not State.isUnlocked
        notify("SH Script", State.isUnlocked and "🔓 الالتقاط مفتوح" or "🔒 الالتقاط مقفل")
    end)

    -- زر تشغيل النسخ (يبقى مشتغل حتى لو خرج/دخل)
    local toggleCopyBtn = makeButton("▶️ تشغيل", THEME.green, 0, 1, 110, 44, remotesPage)
    local function refreshToggle()
        if State.copyRunning then
            toggleCopyBtn.Text = "⏹️ إيقاف"
            toggleCopyBtn.BackgroundColor3 = THEME.red
        else
            toggleCopyBtn.Text = "▶️ تشغيل"
            toggleCopyBtn.BackgroundColor3 = THEME.green
        end
    end
    State.refreshToggle = refreshToggle
    toggleCopyBtn.MouseButton1Click:Connect(function()
        if State.copyRunning then
            stopCopy()
        else
            if not State.selectedPlayer then
                notify("SH Script", "⚠️ اختر لاعباً من تبويب Copy")
                setActiveTab("copy")
            else
                startCopy(State.selectedPlayer)
            end
        end
        refreshToggle()
    end)

    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 40),
        Position = UDim2.new(0, 4, 0, 165),
        BackgroundTransparency = 1,
        Text = "ملاحظة: زر التشغيل يرسل الأوامر على آخر لاعب محدد.\nالإعدادات محفوظة حتى بعد الخروج أو تغيير الماب.",
        TextColor3 = THEME.subtext,
        TextSize = 11,
        Font = Enum.Font.Gotham,
        TextWrapped = true,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })

    ------------------------------------------------------------
    -- [PAGE 2: COPY]
    ------------------------------------------------------------
    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 4, 0, 0),
        BackgroundTransparency = 1,
        Text = "اختر لاعباً من القائمة:",
        TextColor3 = THEME.text,
        TextSize = 13,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = copyPage,
    })

    local scroll = new("ScrollingFrame", {
        Size = UDim2.new(1, 0, 0, 150),
        Position = UDim2.new(0, 0, 0, 24),
        BackgroundColor3 = THEME.panel,
        BorderSizePixel = 0,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        ScrollBarThickness = 5,
        ScrollBarImageColor3 = THEME.accent,
        Parent = copyPage,
    })
    corner(scroll, 8)

    local listLayout = new("UIListLayout", {
        Padding = UDim.new(0, 4),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = scroll,
    })
    new("UIPadding", {
        PaddingTop = UDim.new(0, 6),
        PaddingLeft = UDim.new(0, 6),
        PaddingRight = UDim.new(0, 6),
        PaddingBottom = UDim.new(0, 6),
        Parent = scroll,
    })

    local function refreshPlayerList()
        for _, c in ipairs(scroll:GetChildren()) do
            if c:IsA("TextButton") then c:Destroy() end
        end
        local players = Players:GetPlayers()
        for _, plr in ipairs(players) do
            if plr ~= LocalPlayer then
                local btn = new("TextButton", {
                    Size = UDim2.new(1, -6, 0, 30),
                    BackgroundColor3 = THEME.panel2,
                    Text = "👤 " .. plr.Name .. " (@" .. plr.DisplayName .. ")",
                    TextColor3 = THEME.text,
                    TextSize = 12,
                    Font = Enum.Font.GothamMedium,
                    BorderSizePixel = 0,
                    TextXAlignment = Enum.TextXAlignment.Left,
                    Parent = scroll,
                })
                corner(btn, 6)
                btn.MouseButton1Click:Connect(function()
                    State.selectedPlayer = plr.Name
                    for _, other in ipairs(scroll:GetChildren()) do
                        if other:IsA("TextButton") then
                            other.BackgroundColor3 = THEME.panel2
                        end
                    end
                    btn.BackgroundColor3 = THEME.accent
                    btn.TextColor3 = Color3.new(0,0,0)
                    notify("SH Script", "🎯 تم اختيار: " .. plr.Name)
                end)
            end
        end
        scroll.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 12)
    end

    Players.PlayerAdded:Connect(function() task.wait(1); refreshPlayerList() end)
    Players.PlayerRemoving:Connect(function() task.wait(0.2); refreshPlayerList() end)
    refreshPlayerList()

    -- أزرار النسخ
    local copyStartBtn = makeButton("▶️ بدء النسخ", THEME.green, 0, 0.5, 182, 40, copyPage)
    copyStartBtn.MouseButton1Click:Connect(function()
        if State.selectedPlayer then
            startCopy(State.selectedPlayer)
            if State.refreshToggle then State.refreshToggle() end
        else
            notify("SH Script", "⚠️ اختر لاعباً أولاً")
        end
    end)

    local copyStopBtn = makeButton("⏹️ إيقاف النسخ", THEME.red, 0.5, 0.5, 182, 40, copyPage)
    copyStopBtn.MouseButton1Click:Connect(function()
        stopCopy()
        if State.refreshToggle then State.refreshToggle() end
    end)

    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 18),
        Position = UDim2.new(0, 4, 0, 228),
        BackgroundTransparency = 1,
        Text = "⚡ الأوامر تُرسل بأقصى سرعة عبر Chat/Admin Remote",
        TextColor3 = THEME.subtext,
        TextSize = 11,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = copyPage,
    })

    return gui
end

-- ==========================================================
-- [FLOW]
-- ==========================================================
local function openMainUI()
    if State.mainGui then
        State.mainGui.Enabled = true
        if State.refreshRemoteStatus then State.refreshRemoteStatus() end
        if State.refreshToggle then State.refreshToggle() end
        return
    end
    buildMainUI()
end

local welcomeGui, welcomeStatus

local function onChatUnlocked()
    if welcomeGui then
        if welcomeStatus then
            welcomeStatus.Text = "✅ تم سحب الريموت! جاري فتح الواجهة..."
            welcomeStatus.TextColor3 = THEME.green
        end
        task.delay(0.8, function()
            pcall(function() welcomeGui:Destroy() end)
            welcomeGui = nil
            -- حفظ علامة أن الشاشة الأولى انتهت
            if hasFS then fileWrite(CONFIG.FirstRunFile, "done") end
            State.firstRunDone = true
            openMainUI()
        end)
    else
        openMainUI()
    end
end
State.onUnlock = onChatUnlocked

-- منطق البداية
if not State.firstRunDone then
    welcomeGui, welcomeStatus = showWelcome()
else
    -- سبق وشاف الشاشة → يفتح مباشرة، وإن لم يوجد ريموت يحاول البحث تلقائياً
    if not State.chatRemote and not State.adminRemote then
        autoFindRemotes()
    end
    openMainUI()
    notify("SH Script", "👋 أهلاً بك مجدداً")
end

print("--- [SH Script v2.0]: النظام يعمل. اكتب (شهد) في الشات إن كانت الشاشة الأولى مفتوحة ---")
