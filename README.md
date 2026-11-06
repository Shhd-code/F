-- ==========================================================
-- [SH Script] - Chat Remote Capture v3.0 (SAFE)
-- ==========================================================
local Players           = game:GetService("Players")
local StarterGui        = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace         = game:GetService("Workspace")
local RunService        = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ==========================================================
-- [CONFIG]
-- ==========================================================
local CONFIG = {
    FirstRunFile    = "SH_FirstRun.txt",
    ChatRemoteFile  = "SH_ChatRemote.txt",
}

-- ==========================================================
-- [STORAGE]
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
    firstRunDone = fileExists(CONFIG.FirstRunFile),
    chatRemote   = nil,
    chatPath     = nil,
    pendingObj   = nil,   -- يُضبط من الهوك، يُعالج في Heartbeat
    hookActive   = false,
    ui           = nil,
    onCaptured   = nil,
}

-- ==========================================================
-- [PATH HELPERS]
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

local function saveChatRemote()
    if State.chatRemote then
        fileWrite(CONFIG.ChatRemoteFile, State.chatRemote:GetFullName())
    end
end

local function loadChatRemote()
    local path = fileRead(CONFIG.ChatRemoteFile)
    if not path or path == "" then return end
    local obj = findByPath(path)
    if obj and (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction")) then
        State.chatRemote = obj
        State.chatPath = path
        print("[SH] chat remote loaded:", path)
    else
        print("[SH] cached chat remote missing, cleared")
        fileWrite(CONFIG.ChatRemoteFile, "")
    end
end

loadChatRemote()

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
-- [HEARTBEAT — يعالج الالتقاط خارج الهوك]
-- ==========================================================
RunService.Heartbeat:Connect(function()
    if State.pendingObj then
        local obj = State.pendingObj
        State.pendingObj = nil
        if obj and (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction")) then
            State.chatRemote = obj
            State.chatPath = obj:GetFullName()
            saveChatRemote()
            print("🚀 [SH] التُقط ريموت الشات:", State.chatPath)
            if State.onCaptured then
                local ok, err = pcall(State.onCaptured)
                if not ok then print("[SH] onCaptured error:", err) end
            end
        end
    end
end)

-- ==========================================================
-- [HOOK — آمن جداً]
-- ==========================================================
local function isRemoteInstance(obj)
    if typeof(obj) ~= "Instance" then return false end
    local ok, res = pcall(function()
        return obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction")
    end)
    return ok and res
end

local function hookProcessor(self, args)
    -- يتحقق إذا الرسالة تحتوي "شهد"
    if not args or #args == 0 then return false end
    for _, a in ipairs(args) do
        if typeof(a) == "string" then
            local low = a:lower()
            if low:find("شهد") or low:find("shahd") then
                if isRemoteInstance(self) then
                    State.pendingObj = self
                    return true
                end
            end
        end
    end
    return false
end

local function installHook()
    -- طريقة 1: hookmetamethod
    if type(hookmetamethod) == "function" then
        local ok, err = pcall(function()
            local old
            old = hookmetamethod(game, "__namecall", function(self, ...)
                local okm, method = pcall(getnamecallmethod)
                if not okm then return old(self, ...) end
                if method ~= "FireServer" and method ~= "InvokeServer" then
                    return old(self, ...)
                end
                if State.chatRemote then return old(self, ...) end -- نوقف بعد الالتقاط
                local okArgs, args = pcall(function() return { ... } end)
                if okArgs and args then
                    pcall(hookProcessor, self, args)
                end
                return old(self, ...)
            end)
            State.hookActive = true
        end)
        if ok and State.hookActive then
            print("✅ [SH] hookmetamethod مثبت")
            return true
        end
        State.hookActive = false
        print("⚠️ [SH] hookmetamethod فشل:", err)
    end

    -- طريقة 2: getrawmetatable + hookfunction
    if type(getrawmetatable) == "function" and type(hookfunction) == "function" then
        local ok, err = pcall(function()
            local mt = getrawmetatable(game)
            if type(setreadonly) == "function" then pcall(setreadonly, mt, false) end
            local old = mt.__namecall
            if not old then error("no __namecall") end
            hookfunction(old, function(self, ...)
                local okm, method = pcall(getnamecallmethod)
                if not okm then return old(self, ...) end
                if method ~= "FireServer" and method ~= "InvokeServer" then
                    return old(self, ...)
                end
                if State.chatRemote then return old(self, ...) end
                local okArgs, args = pcall(function() return { ... } end)
                if okArgs and args then
                    pcall(hookProcessor, self, args)
                end
                return old(self, ...)
            end)
            if type(setreadonly) == "function" then pcall(setreadonly, mt, true) end
            State.hookActive = true
        end)
        if ok and State.hookActive then
            print("✅ [SH] hookfunction مثبت")
            return true
        end
        State.hookActive = false
        print("⚠️ [SH] hookfunction فشل:", err)
    end

    return false
end

-- ==========================================================
-- [MANUAL PICKER — fallback لو الهوك ما اشتغل]
-- ==========================================================
local function getAllRemotes()
    local list, seen = {}, {}
    local function add(obj)
        local full = obj:GetFullName()
        if not seen[full] then
            seen[full] = true
            table.insert(list, obj)
        end
    end
    local function scan(c)
        if not c then return end
        local ok, d = pcall(function() return c:GetDescendants() end)
        if not ok or not d then return end
        for _, o in ipairs(d) do
            if o:IsA("RemoteEvent") or o:IsA("RemoteFunction") then add(o) end
        end
    end
    scan(ReplicatedStorage); scan(Workspace); scan(Players)
    return list
end

local function showManualPicker(onPicked)
    local gui = Instance.new("ScreenGui")
    gui.Name = "SH_ManualPicker"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = PlayerGui

    local f = Instance.new("Frame")
    f.Size = UDim2.new(0, 420, 0, 460)
    f.Position = UDim2.new(0.5, -210, 0.5, -230)
    f.BackgroundColor3 = Color3.fromRGB(15,15,18)
    f.BorderSizePixel = 0
    f.Active = true; f.Draggable = true
    f.Parent = gui
    local c1 = Instance.new("UICorner"); c1.CornerRadius = UDim.new(0, 12); c1.Parent = f
    local s1 = Instance.new("UIStroke"); s1.Color = Color3.fromRGB(255,170,0); s1.Thickness = 1.5; s1.Parent = f

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -20, 0, 30)
    title.Position = UDim2.new(0, 10, 0, 8)
    title.BackgroundTransparency = 1
    title.Text = "اختر ريموت الشات يدوياً"
    title.TextColor3 = Color3.fromRGB(255,170,0)
    title.TextSize = 15
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = f

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 26, 0, 26)
    close.Position = UDim2.new(1, -34, 0, 8)
    close.BackgroundColor3 = Color3.fromRGB(235,70,70)
    close.Text = "✕"; close.TextColor3 = Color3.new(1,1,1)
    close.TextSize = 14; close.Font = Enum.Font.GothamBold
    close.BorderSizePixel = 0; close.Parent = f
    local c2 = Instance.new("UICorner"); c2.CornerRadius = UDim.new(0, 6); c2.Parent = close
    close.MouseButton1Click:Connect(function() gui:Destroy() end)

    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, -20, 0, 400)
    scroll.Position = UDim2.new(0, 10, 0, 44)
    scroll.BackgroundColor3 = Color3.fromRGB(24,24,28)
    scroll.BorderSizePixel = 0
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    scroll.ScrollBarThickness = 5
    scroll.ScrollBarImageColor3 = Color3.fromRGB(255,170,0)
    scroll.Parent = f
    local c3 = Instance.new("UICorner"); c3.CornerRadius = UDim.new(0, 8); c3.Parent = scroll

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 4)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = scroll

    local pad = Instance.new("UIPadding")
    pad.PaddingTop = UDim.new(0, 6); pad.PaddingLeft = UDim.new(0, 6)
    pad.PaddingRight = UDim.new(0, 6); pad.PaddingBottom = UDim.new(0, 6)
    pad.Parent = scroll

    for _, remote in ipairs(getAllRemotes()) do
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, -6, 0, 28)
        btn.BackgroundColor3 = Color3.fromRGB(34,34,40)
        btn.Text = "📡 " .. remote:GetFullName()
        btn.TextColor3 = Color3.fromRGB(235,235,235)
        btn.TextSize = 11
        btn.Font = Enum.Font.GothamMedium
        btn.BorderSizePixel = 0
        btn.TextXAlignment = Enum.TextXAlignment.Left
        btn.TextTruncate = Enum.TextTruncate.AtEnd
        btn.Parent = scroll
        local c4 = Instance.new("UICorner"); c4.CornerRadius = UDim.new(0, 5); c4.Parent = btn
        btn.MouseButton1Click:Connect(function()
            if onPicked then onPicked(remote) end
            gui:Destroy()
        end)
    end

    task.defer(function()
        scroll.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 12)
    end)
end

-- ==========================================================
-- [WELCOME SCREEN]
-- ==========================================================
local function buildWelcome()
    local gui = Instance.new("ScreenGui")
    gui.Name = "SH_Welcome"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = PlayerGui

    local f = Instance.new("Frame")
    f.Size = UDim2.new(0, 420, 0, 240)
    f.Position = UDim2.new(0.5, -210, 0.5, -120)
    f.BackgroundColor3 = Color3.fromRGB(15,15,18)
    f.BorderSizePixel = 0
    f.Active = true; f.Draggable = true
    f.Parent = gui
    local c1 = Instance.new("UICorner"); c1.CornerRadius = UDim.new(0, 12); c1.Parent = f
    local s1 = Instance.new("UIStroke"); s1.Color = Color3.fromRGB(255,170,0); s1.Thickness = 1.5; s1.Parent = f

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 45)
    title.BackgroundTransparency = 1
    title.Text = "⚡ SH SCRIPT — التقاط ريموت الشات"
    title.TextColor3 = Color3.fromRGB(255,170,0)
    title.TextSize = 17
    title.Font = Enum.Font.FredokaOne
    title.Parent = f

    local desc = Instance.new("TextLabel")
    desc.Size = UDim2.new(1, -40, 0, 90)
    desc.Position = UDim2.new(0, 20, 0, 45)
    desc.BackgroundTransparency = 1
    desc.Text = "اكتب كلمة (شهد) أو (shahd) في الشات العام الآن\nسيتم التقاط ريموت الشات تلقائياً وحفظه.\n\n(تظهر مرة واحدة فقط)"
    desc.TextColor3 = Color3.fromRGB(235,235,235)
    desc.TextSize = 13
    desc.Font = Enum.Font.GothamMedium
    desc.TextWrapped = true
    desc.Parent = f

    local status = Instance.new("TextLabel")
    status.Size = UDim2.new(1, -40, 0, 20)
    status.Position = UDim2.new(0, 20, 0, 140)
    status.BackgroundTransparency = 1
    status.Text = State.hookActive and "⏳ بانتظار الكلمة..." or "⚠️ الالتقاط التلقائي غير مدعوم — استخدم يدوي"
    status.TextColor3 = State.hookActive and Color3.fromRGB(160,160,170) or Color3.fromRGB(235,70,70)
    status.TextSize = 11
    status.Font = Enum.Font.Gotham
    status.TextXAlignment = Enum.TextXAlignment.Left
    status.Parent = f

    local manualBtn = Instance.new("TextButton")
    manualBtn.Size = UDim2.new(1, -40, 0, 40)
    manualBtn.Position = UDim2.new(0, 20, 1, -55)
    manualBtn.BackgroundColor3 = Color3.fromRGB(255,170,0)
    manualBtn.Text = "🖐️ اختيار يدوي للريموت"
    manualBtn.TextColor3 = Color3.new(0,0,0)
    manualBtn.TextSize = 13
    manualBtn.Font = Enum.Font.GothamBold
    manualBtn.BorderSizePixel = 0
    manualBtn.Parent = f
    local c2 = Instance.new("UICorner"); c2.CornerRadius = UDim.new(0, 8); c2.Parent = manualBtn
    manualBtn.MouseButton1Click:Connect(function()
        showManualPicker(function(remote)
            State.chatRemote = remote
            State.chatPath = remote:GetFullName()
            saveChatRemote()
            if State.onCaptured then State.onCaptured() end
        end)
    end)

    State.ui = { gui = gui, status = status, desc = desc }
    return gui
end

-- ==========================================================
-- [MAIN UI — ريموت الشات فقط]
-- ==========================================================
local function buildMain()
    local gui = Instance.new("ScreenGui")
    gui.Name = "SH_Main"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = PlayerGui

    local main = Instance.new("Frame")
    main.Size = UDim2.new(0, 420, 0, 280)
    main.Position = UDim2.new(0.5, -210, 0.5, -140)
    main.BackgroundColor3 = Color3.fromRGB(15,15,18)
    main.BorderSizePixel = 0
    main.Active = true; main.Draggable = true
    main.Parent = gui
    local c1 = Instance.new("UICorner"); c1.CornerRadius = UDim.new(0, 14); c1.Parent = main
    local s1 = Instance.new("UIStroke"); s1.Color = Color3.fromRGB(255,170,0); s1.Thickness = 1.5; s1.Parent = main

    local header = Instance.new("Frame")
    header.Size = UDim2.new(1, 0, 0, 42)
    header.BackgroundColor3 = Color3.fromRGB(24,24,28)
    header.BorderSizePixel = 0
    header.Parent = main
    local c2 = Instance.new("UICorner"); c2.CornerRadius = UDim.new(0, 14); c2.Parent = header

    local t = Instance.new("TextLabel")
    t.Size = UDim2.new(1, -60, 1, 0)
    t.Position = UDim2.new(0, 15, 0, 0)
    t.BackgroundTransparency = 1
    t.Text = "📡 SH — ريموت الشات"
    t.TextColor3 = Color3.fromRGB(255,170,0)
    t.TextSize = 17
    t.Font = Enum.Font.FredokaOne
    t.TextXAlignment = Enum.TextXAlignment.Left
    t.Parent = header

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 30, 0, 30)
    close.Position = UDim2.new(1, -38, 0, 6)
    close.BackgroundColor3 = Color3.fromRGB(235,70,70)
    close.Text = "✕"; close.TextColor3 = Color3.new(1,1,1)
    close.TextSize = 16; close.Font = Enum.Font.GothamBold
    close.BorderSizePixel = 0; close.Parent = header
    local c3 = Instance.new("UICorner"); c3.CornerRadius = UDim.new(0, 8); c3.Parent = close
    close.MouseButton1Click:Connect(function() gui.Enabled = false end)

    local statusLbl = Instance.new("TextLabel")
    statusLbl.Size = UDim2.new(1, -24, 0, 40)
    statusLbl.Position = UDim2.new(0, 12, 0, 52)
    statusLbl.BackgroundTransparency = 1
    statusLbl.Text = ""
    statusLbl.TextColor3 = Color3.fromRGB(235,235,235)
    statusLbl.TextSize = 12
    statusLbl.Font = Enum.Font.Gotham
    statusLbl.TextXAlignment = Enum.TextXAlignment.Left
    statusLbl.TextWrapped = true
    statusLbl.Parent = main

    local function refresh()
        if State.chatRemote then
            statusLbl.Text = "✅ ريموت الشات:\n" .. State.chatRemote:GetFullName()
            statusLbl.TextColor3 = Color3.fromRGB(0,200,120)
        else
            statusLbl.Text = "❌ لا يوجد ريموت شات محفوظ"
            statusLbl.TextColor3 = Color3.fromRGB(235,70,70)
        end
    end
    State.refreshMain = refresh
    refresh()

    local pickBtn = Instance.new("TextButton")
    pickBtn.Size = UDim2.new(1, -24, 0, 44)
    pickBtn.Position = UDim2.new(0, 12, 0, 110)
    pickBtn.BackgroundColor3 = Color3.fromRGB(255,170,0)
    pickBtn.Text = "🔍 تغيير ريموت الشات (اختيار يدوي)"
    pickBtn.TextColor3 = Color3.new(0,0,0)
    pickBtn.TextSize = 13
    pickBtn.Font = Enum.Font.GothamBold
    pickBtn.BorderSizePixel = 0
    pickBtn.Parent = main
    local c4 = Instance.new("UICorner"); c4.CornerRadius = UDim.new(0, 8); c4.Parent = pickBtn
    pickBtn.MouseButton1Click:Connect(function()
        showManualPicker(function(remote)
            State.chatRemote = remote
            State.chatPath = remote:GetFullName()
            saveChatRemote()
            refresh()
            notify("SH", "✅ حُفظ: " .. remote.Name)
        end)
    end)

    local clearBtn = Instance.new("TextButton")
    clearBtn.Size = UDim2.new(1, -24, 0, 36)
    clearBtn.Position = UDim2.new(0, 12, 0, 164)
    clearBtn.BackgroundColor3 = Color3.fromRGB(45,45,52)
    clearBtn.Text = "🗑️ حذف الريموت المحفوظ"
    clearBtn.TextColor3 = Color3.fromRGB(235,235,235)
    clearBtn.TextSize = 12
    clearBtn.Font = Enum.Font.GothamBold
    clearBtn.BorderSizePixel = 0
    clearBtn.Parent = main
    local c5 = Instance.new("UICorner"); c5.CornerRadius = UDim.new(0, 8); c5.Parent = clearBtn
    clearBtn.MouseButton1Click:Connect(function()
        State.chatRemote = nil
        State.chatPath = nil
        fileWrite(CONFIG.ChatRemoteFile, "")
        refresh()
        notify("SH", "🗑️ حُذف الريموت")
    end)

    local note = Instance.new("TextLabel")
    note.Size = UDim2.new(1, -24, 0, 60)
    note.Position = UDim2.new(0, 12, 0, 210)
    note.BackgroundTransparency = 1
    note.Text = "• الريموت محفوظ ويبقى بعد الخروج أو تغيير الماب.\n• لو فقدت الالتقاط التلقائي، استخدم الاختيار اليدوي."
    note.TextColor3 = Color3.fromRGB(160,160,170)
    note.TextSize = 11
    note.Font = Enum.Font.Gotham
    note.TextWrapped = true
    note.TextXAlignment = Enum.TextXAlignment.Left
    note.TextYAlignment = Enum.TextYAlignment.Top
    note.Parent = main

    return gui
end

-- ==========================================================
-- [FLOW]
-- ==========================================================
local function closeWelcome()
    if State.ui and State.ui.gui then
        pcall(function() State.ui.gui:Destroy() end)
        State.ui = nil
    end
end

local function openMain()
    closeWelcome()
    pcall(buildMain)
end

State.onCaptured = function()
    notify("SH", "✅ التُقط ريموت الشات!")
    openMain()
end

-- 1) تثبيت الهوك (قبل أي شيء)
installHook()

-- 2) إذا كان عندنا ريموت محفوظ → نفتح الواجهة الرئيسية مباشرة
if State.chatRemote then
    openMain()
    notify("SH", "👋 أهلاً بك مجدداً")
else
    -- 3) شاشة الترحيب (سواء أول مرة أو لا يوجد ريموت)
    buildWelcome()
    -- بعد 90 ثانية، إذا لم يُلتقط نغلق الشاشة ونفتح الرئيسية
    task.delay(90, function()
        if State.ui and State.ui.gui then
            openMain()
            notify("SH", "⏱️ انتهت مدة الانتظار — يمكنك الاختيار اليدوي")
        end
    end)
end

print("--- [SH Script v3.0] جاهز — التقاط ريموت الشات فقط ---")
