-- ==========================================================
-- [SH Script] - Remote Hook & Copy System v3.1 (Full / Fixed)
-- ==========================================================
local Players            = game:GetService("Players")
local StarterGui         = game:GetService("StarterGui")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local Workspace          = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ==========================================================
-- [CONFIG]
-- ==========================================================
local CONFIG = {
    FirstRunFile    = "SH_FirstRun.txt",
    RemoteCacheFile = "SH_RemoteCache.txt",
    CommandDelay    = 0.05,
}

local COPY_COMMANDS = {
    ";worm",";ice",";kill",";re",";res",";logs",";clogs",";cmdbar",
    ";warm",";jc",";nv",
}

-- ==========================================================
-- [SAFE STORAGE]
-- ==========================================================
local hasFS = (type(writefile) == "function") and (type(readfile) == "function") and (type(isfile) == "function")
local function fileExists(n) return hasFS and isfile(n) end
local function fileRead(n)
    if not hasFS then return nil end
    local ok, d = pcall(readfile, n); return ok and d or nil
end
local function fileWrite(n, d)
    if not hasFS then return false end
    local ok = pcall(writefile, n, d); return ok
end

-- ==========================================================
-- [STATE]
-- ==========================================================
local State = {
    isUnlocked     = false,
    firstRunDone   = fileExists(CONFIG.FirstRunFile),
    chatRemote     = nil,
    adminRemote    = nil,
    selectedPlayer = nil,
    copyRunning    = false,
    copyThread     = nil,
    mainGui        = nil,
    refreshRemoteStatus = nil,
    refreshToggle       = nil,
}

-- ==========================================================
-- [FIND BY PATH]
-- ==========================================================
local function findByPath(path)
    if not path then return nil end
    local obj = game
    for seg in path:gmatch("[^%.]+") do
        obj = obj and obj:FindFirstChild(seg)
        if not obj then return nil end
    end
    return obj
end

local function loadCachedRemotes()
    local data = fileRead(CONFIG.RemoteCacheFile)
    if not data then return end
    for line in data:gmatch("[^\r\n]+") do
        local kind, path = line:match("^(%w+)|(.+)$")
        if kind and path then
            local obj = findByPath(path)
            if obj then
                if kind == "chat"  then State.chatRemote  = obj end
                if kind == "admin" then State.adminRemote = obj end
            end
        end
    end
end
loadCachedRemotes()

local function saveCachedRemotes()
    local lines = {}
    if State.chatRemote  then table.insert(lines, "chat|"  .. State.chatRemote:GetFullName()) end
    if State.adminRemote then table.insert(lines, "admin|" .. State.adminRemote:GetFullName()) end
    fileWrite(CONFIG.RemoteCacheFile, table.concat(lines, "\n"))
end

-- ==========================================================
-- [NOTIFY]
-- ==========================================================
local function notify(title, text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title or "SH Script",
            Text  = text or "",
            Duration = 4,
        })
    end)
    print(string.format("[SH][%s] %s", tostring(title), tostring(text)))
end

-- ==========================================================
-- [REMOTE AUTO-DETECT]
-- ==========================================================
local function containsAny(str, list)
    local low = str:lower()
    for _, k in ipairs(list) do
        if low:find(k, 1, true) then return true end
    end
    return false
end

local CHAT_KW  = { "chat","say","message","msg","sendmessage","sendchat","chatmsg","sendmsg","talk" }
local ADMIN_KW = { "admin","command","cmd","execute","adonis","admincmd","cmdbar" }

local function autoFindRemotes()
    local foundChat, foundAdmin = nil, nil

    local function scan(container)
        if not container then return end
        local ok, descs = pcall(function() return container:GetDescendants() end)
        if not ok or not descs then return end
        for _, obj in ipairs(descs) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                local name = obj.Name
                if not foundAdmin and containsAny(name, ADMIN_KW) then
                    foundAdmin = obj
                elseif not foundChat and containsAny(name, CHAT_KW) and not containsAny(name, ADMIN_KW) then
                    foundChat = obj
                end
            end
        end
    end

    scan(ReplicatedStorage)
    scan(Workspace)
    pcall(scan, game:GetService("ServerScriptService"))
    pcall(scan, game:GetService("ReplicatedFirst"))
    pcall(scan, Players)

    if foundChat  then State.chatRemote  = foundChat  end
    if foundAdmin then State.adminRemote = foundAdmin end
    if foundChat or foundAdmin then saveCachedRemotes() end
    return foundChat, foundAdmin
end

-- ==========================================================
-- [FIRE]
-- ==========================================================
local function sendCommandWithName(cmd, targetName)
    if State.chatRemote then
        pcall(function()
            if State.chatRemote:IsA("RemoteEvent") then
                State.chatRemote:FireServer(cmd .. " " .. targetName)
                State.chatRemote:FireServer(cmd, targetName)
            else
                State.chatRemote:InvokeServer(cmd .. " " .. targetName)
            end
        end)
    end
    if State.adminRemote then
        pcall(function()
            if State.adminRemote:IsA("RemoteEvent") then
                State.adminRemote:FireServer(cmd .. " " .. targetName)
                State.adminRemote:FireServer(cmd, targetName)
            else
                State.adminRemote:InvokeServer(cmd .. " " .. targetName)
            end
        end)
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
    stopCopy()
    if not targetName or targetName == "" then
        notify("SH Script", "⚠️ اختر لاعباً أولاً"); return
    end
    if not State.chatRemote and not State.adminRemote then
        notify("SH Script", "⚠️ لا يوجد ريموت، اضغط بحث"); return
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
-- [HOOK — التقاط ريموت الشات عند كتابة "شهد"]
-- ==========================================================
local function tryHookNameCall()
    if type(getrawmetatable) ~= "function" then return false, "getrawmetatable غير متوفر" end
    local gm = getrawmetatable(game)
    if not gm then return false, "لا يوجد metatable" end

    if type(setreadonly) == "function" then
        pcall(setreadonly, gm, false)
    end

    local oldNameCall = gm.__namecall
    if not oldNameCall then
        if type(setreadonly) == "function" then pcall(setreadonly, gm, true) end
        return false, "لا يوجد __namecall"
    end

    local hookFn = nil
    if type(hookfunction) == "function" then hookFn = hookfunction
    elseif type(replaceclosure) == "function" then hookFn = replaceclosure end
    if not hookFn then
        if type(setreadonly) == "function" then pcall(setreadonly, gm, true) end
        return false, "لا يوجد hookfunction/replaceclosure"
    end

    hookFn(oldNameCall, function(self, ...)
        local okm, method = pcall(getnamecallmethod)
        if okm and (method == "FireServer" or method == "InvokeServer") and not State.isUnlocked then
            local args = { ... }
            for _, a in ipairs(args) do
                if typeof(a) == "string" then
                    local low = a:lower()
                    if low:find("شهد") or low:find("shahd") then
                        local okIs, isRemote = pcall(function()
                            return self:IsA("RemoteEvent") or self:IsA("RemoteFunction")
                        end)
                        if okIs and isRemote then
                            State.chatRemote = self
                            saveCachedRemotes()
                            State.isUnlocked = true
                            print("🚀 [SH] سُحب ريموت الشات: " .. self:GetFullName())
                            if State.onUnlock then State.onUnlock() end
                        end
                        break
                    end
                end
            end
        end
        return oldNameCall(self, ...)
    end)

    if type(setreadonly) == "function" then pcall(setreadonly, gm, true) end
    return true
end

-- ==========================================================
-- [UI HELPERS]
-- ==========================================================
local function new(c, p, kids)
    local o = Instance.new(c)
    for k, v in pairs(p or {}) do o[k] = v end
    for _, k in ipairs(kids or {}) do k.Parent = o end
    return o
end
local function corner(p, r)
    return new("UICorner", { CornerRadius = UDim.new(0, r or 8), Parent = p })
end
local function stroke(p, c, t)
    return new("UIStroke", { Color = c or Color3.fromRGB(255,170,0), Thickness = t or 1, Parent = p })
end

local THEME = {
    bg     = Color3.fromRGB(15,15,18),
    panel  = Color3.fromRGB(24,24,28),
    panel2 = Color3.fromRGB(34,34,40),
    accent = Color3.fromRGB(255,170,0),
    green  = Color3.fromRGB(0,200,120),
    red    = Color3.fromRGB(235,70,70),
    text   = Color3.fromRGB(235,235,235),
    sub    = Color3.fromRGB(160,160,170),
}

-- ==========================================================
-- [WELCOME]
-- ==========================================================
local welcomeGui, welcomeStatus

local function showWelcome()
    local gui = new("ScreenGui", {
        Name = "SH_WelcomeGui",
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        Parent = PlayerGui,
    })
    local f = new("Frame", {
        Size = UDim2.new(0, 400, 0, 200),
        Position = UDim2.new(0.5, -200, 0.5, -100),
        BackgroundColor3 = THEME.bg,
        BorderSizePixel = 0,
        Parent = gui,
    })
    corner(f, 12); stroke(f, THEME.accent, 1.5)

    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 45),
        BackgroundTransparency = 1,
        Text = "⚡ SH SCRIPT - REMOTE HOOK ⚡",
        TextColor3 = THEME.accent,
        TextSize = 18,
        Font = Enum.Font.FredokaOne,
        Parent = f,
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
        Parent = f,
    })
    welcomeStatus = new("TextLabel", {
        Size = UDim2.new(1, -40, 0, 20),
        Position = UDim2.new(0, 20, 1, -30),
        BackgroundTransparency = 1,
        Text = "⏳ بانتظار الكلمة...",
        TextColor3 = THEME.sub,
        TextSize = 12,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = f,
    })
    welcomeGui = gui
    return gui
end

-- ==========================================================
-- [MAIN UI]
-- ==========================================================
local function buildMainUI()
    local gui = new("ScreenGui", {
        Name = "SH_MainGui",
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        Parent = PlayerGui,
    })
    State.mainGui = gui

    local main = new("Frame", {
        Size = UDim2.new(0, 460, 0, 360),
        Position = UDim2.new(0.5, -230, 0.5, -180),
        BackgroundColor3 = THEME.bg,
        BorderSizePixel = 0,
        Active = true,
        Draggable = true,
        Parent = gui,
    })
    corner(main, 14); stroke(main, THEME.accent, 1.5)

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

    local tabBar = new("Frame", {
        Size = UDim2.new(1, -20, 0, 36),
        Position = UDim2.new(0, 10, 0, 50),
        BackgroundColor3 = THEME.panel,
        BorderSizePixel = 0,
        Parent = main,
    })
    corner(tabBar, 8)

    local function makeTab(text, xPos)
        local b = new("TextButton", {
            Size = UDim2.new(0.5, -4, 1, -6),
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
    local tabRemotes = makeTab("📡 Remotes", 0)
    local tabCopy    = makeTab("📋 Copy",    0.5)

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
    local function setTab(w)
        remotesPage.Visible = (w == "r")
        copyPage.Visible    = (w == "c")
        tabRemotes.BackgroundColor3 = (w == "r") and THEME.accent or THEME.panel2
        tabRemotes.TextColor3       = (w == "r") and Color3.new(0,0,0) or THEME.text
        tabCopy.BackgroundColor3    = (w == "c") and THEME.accent or THEME.panel2
        tabCopy.TextColor3          = (w == "c") and Color3.new(0,0,0) or THEME.text
    end
    setTab("r")
    tabRemotes.MouseButton1Click:Connect(function() setTab("r") end)
    tabCopy.MouseButton1Click:Connect(function() setTab("c") end)

    ------------------------------------------------------------
    -- [REMOTES PAGE]
    ------------------------------------------------------------
    local chatStatus = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 22),
        Position = UDim2.new(0, 4, 0, 0),
        BackgroundTransparency = 1,
        Text = "", TextColor3 = THEME.sub, TextSize = 12,
        Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })
    local adminStatus = new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 22),
        Position = UDim2.new(0, 4, 0, 24),
        BackgroundTransparency = 1,
        Text = "", TextColor3 = THEME.sub, TextSize = 12,
        Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })
    local function refreshRemoteStatus()
        chatStatus.Text = State.chatRemote and ("✅ Chat: " .. State.chatRemote.Name) or "❌ Chat: غير موجود"
        chatStatus.TextColor3 = State.chatRemote and THEME.green or THEME.red
        adminStatus.Text = State.adminRemote and ("✅ Admin: " .. State.adminRemote.Name) or "❌ Admin: غير موجود"
        adminStatus.TextColor3 = State.adminRemote and THEME.green or THEME.red
    end
    State.refreshRemoteStatus = refreshRemoteStatus
    refreshRemoteStatus()

    local function mkBtn(text, color, xPos, xSize, yPos, ySize, parent)
        local b = new("TextButton", {
            Size = UDim2.new(xSize, -6, 0, ySize),
            Position = UDim2.new(xPos, 3, 0, yPos),
            BackgroundColor3 = color,
            Text = text, TextColor3 = Color3.new(1,1,1),
            TextSize = 14, Font = Enum.Font.GothamBold,
            BorderSizePixel = 0,
            Parent = parent or remotesPage,
        })
        corner(b, 8); return b
    end

    local searchBtn = mkBtn("🔍 بحث عن الريموت", THEME.accent, 0, 0.5, 60, 38)
    searchBtn.TextColor3 = Color3.new(0,0,0)
    searchBtn.MouseButton1Click:Connect(function()
        local c, a = autoFindRemotes()
        refreshRemoteStatus()
        if c or a then notify("SH Script", "✅ تم إيجاد الريموتات")
        else notify("SH Script", "⚠️ لم يتم إيجاد ريموتات") end
    end)

    local lockBtn = mkBtn("🔓/🔒 الالتقاط", THEME.panel2, 0.5, 0.5, 60, 38)
    lockBtn.MouseButton1Click:Connect(function()
        State.isUnlocked = not State.isUnlocked
        notify("SH Script", State.isUnlocked and "🔓 الالتقاط مفتوح" or "🔒 الالتقاط مقفل")
    end)

    local toggleBtn = mkBtn("▶️ تشغيل", THEME.green, 0, 1, 110, 44)
    local function refreshToggle()
        if State.copyRunning then
            toggleBtn.Text = "⏹️ إيقاف"
            toggleBtn.BackgroundColor3 = THEME.red
        else
            toggleBtn.Text = "▶️ تشغيل"
            toggleBtn.BackgroundColor3 = THEME.green
        end
    end
    State.refreshToggle = refreshToggle
    toggleBtn.MouseButton1Click:Connect(function()
        if State.copyRunning then stopCopy()
        else
            if State.selectedPlayer then startCopy(State.selectedPlayer)
            else notify("SH Script", "⚠️ اختر لاعباً من تبويب Copy"); setTab("c") end
        end
        refreshToggle()
    end)

    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 40),
        Position = UDim2.new(0, 4, 0, 165),
        BackgroundTransparency = 1,
        Text = "زر التشغيل يرسل الأوامر على آخر لاعب محدد.\nالإعدادات محفوظة بعد الخروج أو تغيير الماب.",
        TextColor3 = THEME.sub, TextSize = 11,
        Font = Enum.Font.Gotham, TextWrapped = true,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = remotesPage,
    })

    ------------------------------------------------------------
    -- [COPY PAGE]
    ------------------------------------------------------------
    new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 4, 0, 0),
        BackgroundTransparency = 1,
        Text = "اختر لاعباً:",
        TextColor3 = THEME.text, TextSize = 13,
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
    local layout = new("UIListLayout", {
        Padding = UDim.new(0, 4),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = scroll,
    })
    new("UIPadding", {
        PaddingTop = UDim.new(0, 6), PaddingLeft = UDim.new(0, 6),
        PaddingRight = UDim.new(0, 6), PaddingBottom = UDim.new(0, 6),
        Parent = scroll,
    })

    local function refreshPlayers()
        for _, c in ipairs(scroll:GetChildren()) do
            if c:IsA("TextButton") then c:Destroy() end
        end
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local btn = new("TextButton", {
                    Size = UDim2.new(1, -6, 0, 30),
                    BackgroundColor3 = (State.selectedPlayer == plr.Name) and THEME.accent or THEME.panel2,
                    Text = "👤 " .. plr.Name,
                    TextColor3 = (State.selectedPlayer == plr.Name) and Color3.new(0,0,0) or THEME.text,
                    TextSize = 12,
                    Font = Enum.Font.GothamMedium,
                    BorderSizePixel = 0,
                    TextXAlignment = Enum.TextXAlignment.Left,
                    Parent = scroll,
                })
                corner(btn, 6)
                btn.MouseButton1Click:Connect(function()
                    State.selectedPlayer = plr.Name
                    for _, o in ipairs(scroll:GetChildren()) do
                        if o:IsA("TextButton") then
                            o.BackgroundColor3 = THEME.panel2
                            o.TextColor3 = THEME.text
                        end
                    end
                    btn.BackgroundColor3 = THEME.accent
                    btn.TextColor3 = Color3.new(0,0,0)
                    notify("SH Script", "🎯 تم اختيار: " .. plr.Name)
                end)
            end
        end
        scroll.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 12)
    end
    refreshPlayers()
    Players.PlayerAdded:Connect(function() task.wait(1); refreshPlayers() end)
    Players.PlayerRemoving:Connect(function() task.wait(0.2); refreshPlayers() end)

    local startBtn = mkBtn("▶️ بدء النسخ", THEME.green, 0, 0.5, 182, 40, copyPage)
    startBtn.MouseButton1Click:Connect(function()
        if State.selectedPlayer then
            startCopy(State.selectedPlayer)
            if State.refreshToggle then State.refreshToggle() end
        else notify("SH Script", "⚠️ اختر لاعباً") end
    end)
    local stopBtn = mkBtn("⏹️ إيقاف", THEME.red, 0.5, 0.5, 182, 40, copyPage)
    stopBtn.MouseButton1Click:Connect(function()
        stopCopy()
        if State.refreshToggle then State.refreshToggle() end
    end)

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

local function onChatUnlocked()
    if welcomeGui then
        if welcomeStatus then
            welcomeStatus.Text = "✅ تم سحب الريموت! جاري فتح الواجهة..."
            welcomeStatus.TextColor3 = THEME.green
        end
        task.delay(0.8, function()
            pcall(function() if welcomeGui then welcomeGui:Destroy() end end)
            welcomeGui = nil
            fileWrite(CONFIG.FirstRunFile, "done")
            State.firstRunDone = true
            openMainUI()
        end)
    else
        openMainUI()
    end
end
State.onUnlock = onChatUnlocked

-- محاولة الهوك
local hookOk, hookErr = tryHookNameCall()
if hookOk then print("✅ [SH] الـ __namecall hook مثبت")
else print("⚠️ [SH] الهوك غير متاح: " .. tostring(hookErr) .. " — نعتمد على البحث التلقائي") end

-- مسح مبكر
if not State.chatRemote or not State.adminRemote then
    autoFindRemotes()
end

-- مراقبة ريموتات جديدة
pcall(function()
    ReplicatedStorage.DescendantAdded:Connect(function(o)
        if (o:IsA("RemoteEvent") or o:IsA("RemoteFunction")) and not State.adminRemote then
            if containsAny(o.Name, ADMIN_KW) then
                State.adminRemote = o
                saveCachedRemotes()
                if State.refreshRemoteStatus then State.refreshRemoteStatus() end
            end
        end
    end)
end)

-- البداية
if not State.firstRunDone then
    showWelcome()
else
    openMainUI()
    notify("SH Script", "👋 أهلاً بك مجدداً")
end

print("--- [SH Script v3.1] جاهز. اكتب (شهد) في الشات إن ظهرت شاشة الترحيب ---")
