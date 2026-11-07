--[[
 SH Hub - Roblox UI Script
 Tabs: نسخ (Copy) | تحكم (Control) | ريموت (Remote) | سبام اكسترا
]]

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
if LocalPlayer and string.lower(LocalPlayer.Name) == "vlyxan" then
    LocalPlayer:Kick("غير مسموح لهذا الحساب بتشغيل السكربت")
    return
end
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local capturedChatRemote = nil

-- helper: choose best parent for executor GUIs (Delta/Synapse/etc)
local function getGuiParent()
 local ok, hidden = pcall(function() return gethui() end)
 if ok and hidden then return hidden end
 local ok2, cg = pcall(function() return CoreGui end)
 if ok2 and cg then return cg end
 return PlayerGui
end
local guiParent = getGuiParent()

-- cleanup old
pcall(function()
 for _, n in ipairs({"SHHub","SHMini","SHSplash"}) do
 local old = guiParent:FindFirstChild(n)
 if old then old:Destroy() end
 local old2 = PlayerGui:FindFirstChild(n)
 if old2 then old2:Destroy() end
 end
end)

-- إشعار فريق SH Hub – يظهر بعد دقيقتين من تشغيل السكربت

local function getAvatar(username)
 local ok1, uid = pcall(function()
 return Players:GetUserIdFromNameAsync(username)
 end)
 if not ok1 then return "" end
 local ok2, url = pcall(function()
 return Players:GetUserThumbnailAsync(uid, Enum.ThumbnailType.AvatarBust, Enum.ThumbnailSize.Size420x420)
 end)
 return ok2 and url or ""
end

local function showTeamCard()
 pcall(function()
 local sg = Instance.new("ScreenGui")
 sg.Name = "SH_TeamCard"
 sg.DisplayOrder = 9999997
 sg.IgnoreGuiInset = true
 sg.ResetOnSpawn = false
 pcall(function() sg.Parent = CoreGui end)
 if not sg.Parent then sg.Parent = PlayerGui end

 local overlay = Instance.new("Frame", sg)
 overlay.Size = UDim2.new(1, 0, 1, 0)
 overlay.BackgroundColor3 = Color3.new(0, 0, 0)
 overlay.BackgroundTransparency = 0.5
 overlay.BorderSizePixel = 0

 local card = Instance.new("Frame", sg)
 card.Size = UDim2.new(0, 340, 0, 0)
 card.AnchorPoint = Vector2.new(0.5, 0.5)
 card.Position = UDim2.new(0.5, 0, 0.65, 0)
 card.BackgroundColor3 = Color3.fromRGB(5, 10, 20)
 card.BackgroundTransparency = 0.05
 card.BorderSizePixel = 0
 card.AutomaticSize = Enum.AutomaticSize.Y
 Instance.new("UICorner", card).CornerRadius = UDim.new(0, 18)
 local stroke = Instance.new("UIStroke", card)
 stroke.Color = Color3.fromRGB(0, 200, 255)
 stroke.Thickness = 1.8
 stroke.Transparency = 0.2

 local pad = Instance.new("UIPadding", card)
 pad.PaddingTop = UDim.new(0, 16)
 pad.PaddingBottom = UDim.new(0, 20)
 pad.PaddingLeft = UDim.new(0, 16)
 pad.PaddingRight = UDim.new(0, 16)

 local layout = Instance.new("UIListLayout", card)
 layout.Padding = UDim.new(0, 14)
 layout.SortOrder = Enum.SortOrder.LayoutOrder
 layout.HorizontalAlignment = Enum.HorizontalAlignment.Center

 local title = Instance.new("TextLabel", card)
 title.Size = UDim2.new(1, 0, 0, 26)
 title.BackgroundTransparency = 1
 title.Font = Enum.Font.GothamBold
 title.Text = "✦ فريق SH Hub ✦"
 title.TextSize = 17
 title.TextColor3 = Color3.fromRGB(0, 210, 255)
 title.TextXAlignment = Enum.TextXAlignment.Center
 title.LayoutOrder = 1

 local div = Instance.new("Frame", card)
 div.Size = UDim2.new(1, 0, 0, 1)
 div.BackgroundColor3 = Color3.fromRGB(0, 180, 255)
 div.BackgroundTransparency = 0.5
 div.BorderSizePixel = 0
 div.LayoutOrder = 2

 local function makePersonCard(username, roleText, order)
 local avatarUrl = getAvatar(username)

 local row = Instance.new("Frame", card)
 row.Size = UDim2.new(1, 0, 0, 90)
 row.BackgroundColor3 = Color3.fromRGB(10, 20, 35)
 row.BackgroundTransparency = 0.3
 row.BorderSizePixel = 0
 row.LayoutOrder = order
 Instance.new("UICorner", row).CornerRadius = UDim.new(0, 12)

 local img = Instance.new("ImageLabel", row)
 img.Size = UDim2.new(0, 72, 0, 72)
 img.Position = UDim2.new(0, 10, 0.5, -36)
 img.BackgroundColor3 = Color3.fromRGB(20, 30, 50)
 img.BorderSizePixel = 0
 img.Image = avatarUrl
 Instance.new("UICorner", img).CornerRadius = UDim.new(0, 36)
 local imgStroke = Instance.new("UIStroke", img)
 imgStroke.Color = Color3.fromRGB(0, 200, 255)
 imgStroke.Thickness = 2
 imgStroke.Transparency = 0.1

 local nameLbl = Instance.new("TextLabel", row)
 nameLbl.Size = UDim2.new(1, -96, 0, 32)
 nameLbl.Position = UDim2.new(0, 90, 0.5, -30)
 nameLbl.BackgroundTransparency = 1
 nameLbl.Font = Enum.Font.GothamBold
 nameLbl.Text = "@" .. username
 nameLbl.TextSize = 14
 nameLbl.TextColor3 = Color3.fromRGB(220, 240, 255)
 nameLbl.TextXAlignment = Enum.TextXAlignment.Right

 local roleLbl = Instance.new("TextLabel", row)
 roleLbl.Size = UDim2.new(1, -96, 0, 28)
 roleLbl.Position = UDim2.new(0, 90, 0.5, 4)
 roleLbl.BackgroundTransparency = 1
 roleLbl.Font = Enum.Font.GothamSemibold
 roleLbl.Text = roleText
 roleLbl.TextSize = 13
 roleLbl.TextColor3 = Color3.fromRGB(0, 210, 255)
 roleLbl.TextXAlignment = Enum.TextXAlignment.Right
 end

 makePersonCard("shhode320", "👑 المصممه", 3)

 local closeBtn = Instance.new("TextButton", card)
 closeBtn.Size = UDim2.new(1, 0, 0, 34)
 closeBtn.BackgroundColor3 = Color3.fromRGB(0, 140, 200)
 closeBtn.BackgroundTransparency = 0.1
 closeBtn.BorderSizePixel = 0
 closeBtn.AutoButtonColor = false
 closeBtn.Font = Enum.Font.GothamBold
 closeBtn.Text = "حسناً 👍"
 closeBtn.TextSize = 14
 closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
 closeBtn.LayoutOrder = 4
 Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 10)
 closeBtn.MouseEnter:Connect(function()
 TweenService:Create(closeBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(0, 180, 255)}):Play()
 end)
 closeBtn.MouseLeave:Connect(function()
 TweenService:Create(closeBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(0, 140, 200)}):Play()
 end)

 card.BackgroundTransparency = 1
 overlay.BackgroundTransparency = 1
 TweenService:Create(overlay, TweenInfo.new(0.3), {BackgroundTransparency = 0.5}):Play()
 TweenService:Create(card, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
 {Position = UDim2.new(0.5, 0, 0.5, 0), BackgroundTransparency = 0.05}):Play()

 local function closeCard()
 TweenService:Create(card, TweenInfo.new(0.25), {BackgroundTransparency = 1}):Play()
 TweenService:Create(overlay, TweenInfo.new(0.25), {BackgroundTransparency = 1}):Play()
 task.delay(0.28, function() pcall(function() sg:Destroy() end) end)
 end

 closeBtn.MouseButton1Click:Connect(closeCard)
 end)
end

task.delay(120, function()
 showTeamCard()
end)

-- ─── نافذة "ما الجديد" تظهر بعد الإنترو ─────────────────────
local function showWhatsNew()
 local core = game:GetService("CoreGui")
 local wsg = Instance.new("ScreenGui")
 wsg.Name = "SH_WhatsNew"
 wsg.DisplayOrder = 9999998
 wsg.IgnoreGuiInset = true
 wsg.ResetOnSpawn = false
 wsg.Parent = core

 local overlay = Instance.new("Frame", wsg)
 overlay.Size = UDim2.new(1, 0, 1, 0)
 overlay.BackgroundColor3 = Color3.new(0, 0, 0)
 overlay.BackgroundTransparency = 0.45
 overlay.BorderSizePixel = 0

 local card = Instance.new("Frame", wsg)
 card.Size = UDim2.new(0, 400, 0, 0)
 card.AnchorPoint = Vector2.new(0.5, 0.5)
 card.Position = UDim2.new(0.5, 0, 0.5, 0)
 card.BackgroundColor3 = Color3.fromRGB(6, 18, 10)
 card.BackgroundTransparency = 0.08
 card.BorderSizePixel = 0
 card.AutomaticSize = Enum.AutomaticSize.Y
 Instance.new("UICorner", card).CornerRadius = UDim.new(0, 16)
 local cs = Instance.new("UIStroke", card)
 cs.Color = Color3.fromRGB(0, 230, 110); cs.Thickness = 1.8; cs.Transparency = 0.15

 local cpad = Instance.new("UIPadding", card)
 cpad.PaddingTop = UDim.new(0, 16); cpad.PaddingBottom = UDim.new(0, 18)
 cpad.PaddingLeft = UDim.new(0, 18); cpad.PaddingRight = UDim.new(0, 18)

 local layout = Instance.new("UIListLayout", card)
 layout.Padding = UDim.new(0, 10)
 layout.SortOrder = Enum.SortOrder.LayoutOrder

 local titleRow = Instance.new("Frame", card)
 titleRow.Size = UDim2.new(1, 0, 0, 30)
 titleRow.BackgroundTransparency = 1
 titleRow.LayoutOrder = 1
 local tLayout = Instance.new("UIListLayout", titleRow)
 tLayout.FillDirection = Enum.FillDirection.Horizontal
 tLayout.VerticalAlignment = Enum.VerticalAlignment.Center
 tLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

 local titleLbl = Instance.new("TextLabel", titleRow)
 titleLbl.BackgroundTransparency = 1
 titleLbl.Font = Enum.Font.GothamBold
 titleLbl.Text = "✦ ما الجديد؟ ✦"
 titleLbl.TextSize = 20
 titleLbl.TextColor3 = Color3.fromRGB(0, 255, 130)
 titleLbl.AutomaticSize = Enum.AutomaticSize.XY

 local divider = Instance.new("Frame", card)
 divider.Size = UDim2.new(1, 0, 0, 1)
 divider.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
 divider.BackgroundTransparency = 0.6
 divider.BorderSizePixel = 0
 divider.LayoutOrder = 2

 local bodyLines = {
 "✦  اخر الاحداسسسس 😝🤟 ✦",
 "",
 "🔹 تمت إضافة لوقس دخول وخروج",
 "🔹 تم اضافة تغير سيرفر وكذا ",
 "حياكم",
 " • حلتيته",
 " • حلاتيت",
 "🔹 حنتيتة ",
 "🔹 باسورد سكربت الاغاني MZA وبس حياكم ❣️🤚 ",
 "",
 "",
 "",
 "حسابي روب: shhode320~ والثاني IM_6767NEHAHAHA~~~~~💅 ",
 }

 local bodyLbl = Instance.new("TextLabel", card)
 bodyLbl.Size = UDim2.new(1, 0, 0, 0)
 bodyLbl.AutomaticSize = Enum.AutomaticSize.Y
 bodyLbl.BackgroundTransparency = 1
 bodyLbl.Font = Enum.Font.GothamSemibold
 bodyLbl.Text = table.concat(bodyLines, "\n")
 bodyLbl.TextSize = 14
 bodyLbl.TextColor3 = Color3.fromRGB(200, 245, 215)
 bodyLbl.TextXAlignment = Enum.TextXAlignment.Right
 bodyLbl.TextWrapped = true
 bodyLbl.RichText = false
 bodyLbl.LayoutOrder = 3

 local closeBtn = Instance.new("TextButton", card)
 closeBtn.Size = UDim2.new(1, 0, 0, 36)
 closeBtn.BackgroundColor3 = Color3.fromRGB(0, 160, 70)
 closeBtn.BackgroundTransparency = 0.1
 closeBtn.BorderSizePixel = 0
 closeBtn.AutoButtonColor = false
 closeBtn.Font = Enum.Font.GothamBold
 closeBtn.Text = "فاهمت، يلا نبدأ! 🚀"
 closeBtn.TextSize = 15
 closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
 closeBtn.LayoutOrder = 4
 Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 10)
 closeBtn.MouseEnter:Connect(function()
 TweenService:Create(closeBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(0, 210, 90)}):Play()
 end)
 closeBtn.MouseLeave:Connect(function()
 TweenService:Create(closeBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(0, 160, 70)}):Play()
 end)

 card.Position = UDim2.new(0.5, 0, 0.65, 0)
 card.BackgroundTransparency = 1
 TweenService:Create(card, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
 {Position = UDim2.new(0.5, 0, 0.5, 0), BackgroundTransparency = 0.08}):Play()

 local closed = false
 local function closeWhatsNew()
 if closed then return end; closed = true
 TweenService:Create(card, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
 TweenService:Create(overlay, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
 task.delay(0.22, function() pcall(function() wsg:Destroy() end) end)
 end

 closeBtn.MouseButton1Click:Connect(closeWhatsNew)
end

-- ─── الإنترو مع تخطي بالضغط مرتين ──────────────────────────
local function runSplash()
 pcall(function()
 loadstring(game:HttpGet("https://raw.githubusercontent.com/Shhd-code/Antro_ray/refs/heads/main/README.md"))()
 end)
 task.delay(0.5, function()
 showWhatsNew()
 end)
end
pcall(runSplash)

-- Main GUI

local gui = Instance.new("ScreenGui")
gui.Name = "SHHub"; gui.ResetOnSpawn = false; gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; gui.DisplayOrder = 9000
pcall(function() gui.Parent = guiParent end)
if not gui.Parent then gui.Parent = PlayerGui end

local main = Instance.new("Frame", gui)
main.Name = "Main"; main.AnchorPoint = Vector2.new(0.5, 0.5)
main.Position = UDim2.new(0.5, 0, 0.5, 0); main.Size = UDim2.new(0, 520, 0, 360)
main.BackgroundColor3 = Color3.fromRGB(6, 16, 9); main.BackgroundTransparency = 0.25
main.BorderSizePixel = 0
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 14)
local mStroke = Instance.new("UIStroke", main)
mStroke.Color = Color3.fromRGB(0, 230, 110); mStroke.Thickness = 1.4; mStroke.Transparency = 0.25
local mGradient = Instance.new("UIGradient", main)
mGradient.Color = ColorSequence.new{
 ColorSequenceKeypoint.new(0, Color3.fromRGB(10, 30, 16)),
 ColorSequenceKeypoint.new(1, Color3.fromRGB(4, 12, 7)),
}
mGradient.Rotation = 135

local glow = Instance.new("Frame", main)
glow.BackgroundColor3 = Color3.fromRGB(0, 255, 130); glow.BorderSizePixel = 0
glow.Size = UDim2.new(1, 0, 0, 2); glow.Position = UDim2.new(0, 0, 0, 44)
local glowGrad = Instance.new("UIGradient", glow)
glowGrad.Transparency = NumberSequence.new{
 NumberSequenceKeypoint.new(0, 1),
 NumberSequenceKeypoint.new(0.5, 0.2),
 NumberSequenceKeypoint.new(1, 1),
}
task.spawn(function()
 while glow.Parent do
 glowGrad.Offset = Vector2.new(-1, 0)
 TweenService:Create(glowGrad, TweenInfo.new(2.2, Enum.EasingStyle.Linear), {Offset = Vector2.new(1, 0)}):Play()
 task.wait(2.2)
 end
end)

local title = Instance.new("TextLabel", main)
title.BackgroundTransparency = 1
title.Size = UDim2.new(1, -90, 0, 44); title.Position = UDim2.new(0, 16, 0, 0)
title.Font = Enum.Font.GothamBlack; title.Text = "sh + K4 + IFQ"
title.TextSize = 22; title.TextColor3 = Color3.fromRGB(0, 255, 130)
title.TextXAlignment = Enum.TextXAlignment.Left

local closeBtn = Instance.new("TextButton", main)
closeBtn.AnchorPoint = Vector2.new(1, 0)
closeBtn.Position = UDim2.new(1, -8, 0, 8)
closeBtn.Size = UDim2.new(0, 28, 0, 28)
closeBtn.BackgroundColor3 = Color3.fromRGB(0, 60, 30)
closeBtn.BackgroundTransparency = 0.3; closeBtn.BorderSizePixel = 0
closeBtn.Text = "X"; closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextColor3 = Color3.fromRGB(180, 255, 200); closeBtn.TextSize = 16
closeBtn.AutoButtonColor = false
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 8)
local closeStroke = Instance.new("UIStroke", closeBtn)
closeStroke.Color = Color3.fromRGB(0, 200, 100); closeStroke.Transparency = 0.4
closeBtn.MouseEnter:Connect(function() TweenService:Create(closeBtn, TweenInfo.new(0.15), {BackgroundTransparency = 0.1}):Play() end)
closeBtn.MouseLeave:Connect(function() TweenService:Create(closeBtn, TweenInfo.new(0.15), {BackgroundTransparency = 0.3}):Play() end)

local minBtn = Instance.new("TextButton", main)
minBtn.AnchorPoint = Vector2.new(1, 0)
minBtn.Position = UDim2.new(1, -44, 0, 8)
minBtn.Size = UDim2.new(0, 28, 0, 28)
minBtn.BackgroundColor3 = Color3.fromRGB(0, 90, 45)
minBtn.BackgroundTransparency = 0.2; minBtn.BorderSizePixel = 0
minBtn.Text = ""; minBtn.AutoButtonColor = false
Instance.new("UICorner", minBtn).CornerRadius = UDim.new(1, 0)
local minStroke = Instance.new("UIStroke", minBtn)
minStroke.Color = Color3.fromRGB(0, 255, 130); minStroke.Transparency = 0.2; minStroke.Thickness = 1.4
minBtn.MouseEnter:Connect(function() TweenService:Create(minBtn, TweenInfo.new(0.15), {BackgroundTransparency = 0}):Play() end)
minBtn.MouseLeave:Connect(function() TweenService:Create(minBtn, TweenInfo.new(0.15), {BackgroundTransparency = 0.2}):Play() end)

local miniGui = Instance.new("ScreenGui")
miniGui.Name = "SHMini"; miniGui.ResetOnSpawn = false; miniGui.IgnoreGuiInset = true
miniGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; miniGui.DisplayOrder = 9001
pcall(function() miniGui.Parent = guiParent end)
if not miniGui.Parent then miniGui.Parent = PlayerGui end

local miniBubble = Instance.new("TextButton", miniGui)
miniBubble.AnchorPoint = Vector2.new(0, 0.5)
miniBubble.Position = UDim2.new(0, 16, 0.5, 0)
miniBubble.Size = UDim2.new(0, 44, 0, 44)
miniBubble.BackgroundColor3 = Color3.fromRGB(0, 110, 55)
miniBubble.BackgroundTransparency = 0.15; miniBubble.BorderSizePixel = 0
miniBubble.AutoButtonColor = false
miniBubble.Text = "SH"
miniBubble.Font = Enum.Font.GothamBlack
miniBubble.TextSize = 14
miniBubble.TextColor3 = Color3.fromRGB(230, 255, 240)
miniBubble.Visible = false
Instance.new("UICorner", miniBubble).CornerRadius = UDim.new(1, 0)
local miniStroke = Instance.new("UIStroke", miniBubble)
miniStroke.Color = Color3.fromRGB(0, 255, 130); miniStroke.Thickness = 1.6; miniStroke.Transparency = 0.2

task.spawn(function()
 while miniGui.Parent do
 if miniBubble.Visible then
 TweenService:Create(miniStroke, TweenInfo.new(0.7), {Transparency = 0.7}):Play(); task.wait(0.7)
 TweenService:Create(miniStroke, TweenInfo.new(0.7), {Transparency = 0.15}):Play(); task.wait(0.7)
 else
 task.wait(0.2)
 end
 end
end)

miniBubble.Visible = true

local function setHidden(hidden)
 if hidden then
 TweenService:Create(main, TweenInfo.new(0.2), {Size = UDim2.new(0, 0, 0, 0), BackgroundTransparency = 1}):Play()
 task.wait(0.22)
 main.Visible = false
 main.Size = UDim2.new(0, 520, 0, 360)
 main.BackgroundTransparency = 0.25
 else
 main.Visible = true
 main.Size = UDim2.new(0, 0, 0, 0)
 main.BackgroundTransparency = 1
 TweenService:Create(main, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 520, 0, 360), BackgroundTransparency = 0.25}):Play()
 end
end

minBtn.MouseButton1Click:Connect(function() setHidden(true) end)
miniBubble.MouseButton1Click:Connect(function()
 setHidden(main.Visible)
end)

closeBtn.MouseButton1Click:Connect(function()
 TweenService:Create(main, TweenInfo.new(0.2), {Size = UDim2.new(0, 0, 0, 0), BackgroundTransparency = 1}):Play()
 task.wait(0.22); gui:Destroy(); miniGui:Destroy()
end)

local function makeDraggable(handle, target)
 local dragging, dragStart, startPos
 handle.InputBegan:Connect(function(input)
 if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
 dragging = true; dragStart = input.Position; startPos = target.Position
 input.Changed:Connect(function()
 if input.UserInputState == Enum.UserInputState.End then dragging = false end
 end)
 end
 end)
 UserInputService.InputChanged:Connect(function(input)
 if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
 local delta = input.Position - dragStart
 target.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
 end
 end)
end
makeDraggable(title, main)
makeDraggable(miniBubble, miniBubble)

-- Tab bar

local tabBar = Instance.new("Frame", main)
tabBar.BackgroundTransparency = 1
tabBar.Position = UDim2.new(0, 16, 0, 56); tabBar.Size = UDim2.new(1, -32, 0, 36)
local tabLayout = Instance.new("UIListLayout", tabBar)
tabLayout.FillDirection = Enum.FillDirection.Horizontal; tabLayout.Padding = UDim.new(0, 8)

local pages, tabButtons = {}, {}

local function makeTab(name)
 local btn = Instance.new("TextButton", tabBar)
 btn.Size = UDim2.new(0, 118, 1, 0); btn.BackgroundColor3 = Color3.fromRGB(8, 30, 14)
 btn.BackgroundTransparency = 0.4; btn.BorderSizePixel = 0; btn.AutoButtonColor = false
 btn.Font = Enum.Font.GothamBold; btn.Text = name; btn.TextSize = 14
 btn.TextColor3 = Color3.fromRGB(160, 230, 180)
 Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
 local s = Instance.new("UIStroke", btn); s.Color = Color3.fromRGB(0, 200, 100); s.Transparency = 0.6

 local page = Instance.new("Frame", main)
 page.Position = UDim2.new(0, 16, 0, 100); page.Size = UDim2.new(1, -32, 1, -116)
 page.BackgroundColor3 = Color3.fromRGB(4, 12, 7); page.BackgroundTransparency = 0.4
 page.BorderSizePixel = 0; page.Visible = false
 Instance.new("UICorner", page).CornerRadius = UDim.new(0, 10)
 local ps = Instance.new("UIStroke", page); ps.Color = Color3.fromRGB(0, 180, 90); ps.Transparency = 0.7

 pages[name] = page; tabButtons[name] = btn
 btn.MouseButton1Click:Connect(function()
 for n, p in pairs(pages) do p.Visible = (n == name) end
 for n, b in pairs(tabButtons) do
 if n == name then
 TweenService:Create(b, TweenInfo.new(0.15), {BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(0, 255, 130)}):Play()
 else
 TweenService:Create(b, TweenInfo.new(0.15), {BackgroundTransparency = 0.4, TextColor3 = Color3.fromRGB(160, 230, 180)}):Play()
 end
 end
 end)
 return page, btn
end

local copyPage = makeTab("نسخ")
local controlPage = makeTab("تحكم")
local remotePage = makeTab("ريموت")
local extraPage = makeTab("سبام اكسترا")

-- Page: نسخ

local selectedName = nil
local nameMode = "full"
local silentMode = false

local function getEffectiveName()
 if not selectedName then return nil end
 if nameMode == "three" then
 return selectedName:sub(1, 3)
 end
 return selectedName
end

local playersBar = Instance.new("ScrollingFrame", copyPage)
playersBar.Position = UDim2.new(0, 10, 0, 10); playersBar.Size = UDim2.new(1, -20, 0, 46)
playersBar.BackgroundColor3 = Color3.fromRGB(2, 10, 5); playersBar.BackgroundTransparency = 0.5
playersBar.BorderSizePixel = 0; playersBar.ScrollBarThickness = 3
playersBar.ScrollingDirection = Enum.ScrollingDirection.X
playersBar.AutomaticCanvasSize = Enum.AutomaticSize.X
playersBar.CanvasSize = UDim2.new(0, 0, 0, 0)
playersBar.ScrollBarImageColor3 = Color3.fromRGB(0, 255, 130)
Instance.new("UICorner", playersBar).CornerRadius = UDim.new(0, 8)
local pbLayout = Instance.new("UIListLayout", playersBar)
pbLayout.FillDirection = Enum.FillDirection.Horizontal; pbLayout.Padding = UDim.new(0, 6)
pbLayout.SortOrder = Enum.SortOrder.LayoutOrder; pbLayout.VerticalAlignment = Enum.VerticalAlignment.Center
local pbPad = Instance.new("UIPadding", playersBar)
pbPad.PaddingLeft = UDim.new(0, 8); pbPad.PaddingRight = UDim.new(0, 8)

local selectedLabel = Instance.new("TextLabel", copyPage)
selectedLabel.BackgroundTransparency = 1
selectedLabel.Position = UDim2.new(0, 10, 0, 60); selectedLabel.Size = UDim2.new(1, -20, 0, 20)
selectedLabel.Font = Enum.Font.GothamSemibold; selectedLabel.TextSize = 13
selectedLabel.TextColor3 = Color3.fromRGB(180, 255, 200)
selectedLabel.TextXAlignment = Enum.TextXAlignment.Left
selectedLabel.Text = "اختر لاعب من القائمة"

local playerChips = {}
local function refreshPlayers()
 for _, c in ipairs(playersBar:GetChildren()) do
 if c:IsA("TextButton") then c:Destroy() end
 end
 playerChips = {}
 for _, p in ipairs(Players:GetPlayers()) do
 if p ~= LocalPlayer then
 local chip = Instance.new("TextButton", playersBar)
 chip.Size = UDim2.new(0, 0, 1, -8); chip.AutomaticSize = Enum.AutomaticSize.X
 chip.BackgroundColor3 = Color3.fromRGB(10, 35, 18); chip.BackgroundTransparency = 0.2
 chip.BorderSizePixel = 0; chip.AutoButtonColor = false
 chip.Font = Enum.Font.GothamBold; chip.Text = " " .. p.Name .. " "
 chip.TextSize = 13; chip.TextColor3 = Color3.fromRGB(200, 255, 215)
 Instance.new("UICorner", chip).CornerRadius = UDim.new(0, 8)
 local cs = Instance.new("UIStroke", chip); cs.Color = Color3.fromRGB(0, 200, 100); cs.Transparency = 0.5
 chip.MouseButton1Click:Connect(function()
 selectedName = p.Name
 selectedLabel.Text = "تم اختيار: " .. p.Name
 for _, ch in pairs(playerChips) do
 TweenService:Create(ch, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(10, 35, 18)}):Play()
 end
 TweenService:Create(chip, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(0, 120, 60)}):Play()
 end)
 chip.MouseEnter:Connect(function()
 if selectedName ~= p.Name then
 TweenService:Create(chip, TweenInfo.new(0.12), {BackgroundTransparency = 0.05}):Play()
 end
 end)
 chip.MouseLeave:Connect(function()
 if selectedName ~= p.Name then
 TweenService:Create(chip, TweenInfo.new(0.12), {BackgroundTransparency = 0.2}):Play()
 end
 end)
 playerChips[p.Name] = chip
 end
 end
end
refreshPlayers()
Players.PlayerAdded:Connect(refreshPlayers)
Players.PlayerRemoving:Connect(refreshPlayers)

local function makeBigBtn(parent, text, posY, color1, color2)
 color1 = color1 or Color3.fromRGB(0, 150, 75)
 color2 = color2 or Color3.fromRGB(0, 90, 45)
 local b = Instance.new("TextButton", parent)
 b.Position = UDim2.new(0, 10, 0, posY); b.Size = UDim2.new(1, -20, 0, 46)
 b.BackgroundColor3 = Color3.fromRGB(0, 110, 55); b.BackgroundTransparency = 0.15
 b.BorderSizePixel = 0; b.AutoButtonColor = false
 b.Font = Enum.Font.GothamBlack; b.Text = text; b.TextSize = 16
 b.TextColor3 = Color3.fromRGB(255, 255, 255)
 Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
 local s = Instance.new("UIStroke", b); s.Color = Color3.fromRGB(0, 255, 130); s.Transparency = 0.3
 local g = Instance.new("UIGradient", b)
 g.Color = ColorSequence.new{
 ColorSequenceKeypoint.new(0, color1),
 ColorSequenceKeypoint.new(1, color2),
 }
 g.Rotation = 90
 b.MouseEnter:Connect(function() TweenService:Create(b, TweenInfo.new(0.15), {BackgroundTransparency = 0}):Play() end)
 b.MouseLeave:Connect(function() TweenService:Create(b, TweenInfo.new(0.15), {BackgroundTransparency = 0.15}):Play() end)
 return b
end

local spamScroll = Instance.new("ScrollingFrame", copyPage)
spamScroll.Position = UDim2.new(0, 0, 0, 82)
spamScroll.Size = UDim2.new(1, 0, 1, -112)
spamScroll.BackgroundTransparency = 1
spamScroll.BorderSizePixel = 0
spamScroll.ScrollBarThickness = 4
spamScroll.ScrollBarImageColor3 = Color3.fromRGB(0, 255, 130)
spamScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
spamScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
local spamList = Instance.new("UIListLayout", spamScroll)
spamList.Padding = UDim.new(0, 6)
spamList.SortOrder = Enum.SortOrder.LayoutOrder
local spamPad = Instance.new("UIPadding", spamScroll)
spamPad.PaddingTop = UDim.new(0, 6)
spamPad.PaddingBottom = UDim.new(0, 6)
spamPad.PaddingLeft = UDim.new(0, 8)
spamPad.PaddingRight = UDim.new(0, 8)

local prefixRow = Instance.new("Frame", spamScroll)
prefixRow.BackgroundTransparency = 1
prefixRow.Size = UDim2.new(1, 0, 0, 36)
prefixRow.LayoutOrder = 0

local prefixBox = Instance.new("TextBox", prefixRow)
prefixBox.Size = UDim2.new(0, 48, 1, 0); prefixBox.Position = UDim2.new(0, 0, 0, 0)
prefixBox.BackgroundColor3 = Color3.fromRGB(20, 80, 180); prefixBox.BackgroundTransparency = 0.15
prefixBox.BorderSizePixel = 0; prefixBox.Text = ";"
prefixBox.PlaceholderText = ";"
prefixBox.TextColor3 = Color3.fromRGB(255, 255, 255)
prefixBox.Font = Enum.Font.GothamBold; prefixBox.TextSize = 18
prefixBox.ClearTextOnFocus = false
Instance.new("UICorner", prefixBox).CornerRadius = UDim.new(0, 8)
local pbStroke = Instance.new("UIStroke", prefixBox)
pbStroke.Color = Color3.fromRGB(80, 160, 255); pbStroke.Thickness = 1.5; pbStroke.Transparency = 0.2

local prefixLabel = Instance.new("TextLabel", prefixRow)
prefixLabel.BackgroundTransparency = 1
prefixLabel.Position = UDim2.new(0, 56, 0, 0); prefixLabel.Size = UDim2.new(1, -56, 1, 0)
prefixLabel.Font = Enum.Font.GothamSemibold; prefixLabel.TextSize = 13
prefixLabel.TextColor3 = Color3.fromRGB(160, 210, 255)
prefixLabel.TextXAlignment = Enum.TextXAlignment.Left
prefixLabel.Text = "اكتب علامة الادمن الخاصة بك"

local nameModeRow = Instance.new("Frame", spamScroll)
nameModeRow.BackgroundTransparency = 1
nameModeRow.Size = UDim2.new(1, 0, 0, 34)
nameModeRow.LayoutOrder = 1

local fullNameBtn = Instance.new("TextButton", nameModeRow)
fullNameBtn.Size = UDim2.new(0.5, -4, 1, 0)
fullNameBtn.Position = UDim2.new(0, 0, 0, 0)
fullNameBtn.BackgroundColor3 = Color3.fromRGB(0, 160, 80)
fullNameBtn.BackgroundTransparency = 0.1
fullNameBtn.BorderSizePixel = 0
fullNameBtn.AutoButtonColor = false
fullNameBtn.Font = Enum.Font.GothamBold
fullNameBtn.Text = "اسم كامل"
fullNameBtn.TextSize = 14
fullNameBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
Instance.new("UICorner", fullNameBtn).CornerRadius = UDim.new(0, 9)
local fnStroke = Instance.new("UIStroke", fullNameBtn)
fnStroke.Color = Color3.fromRGB(0, 255, 130); fnStroke.Thickness = 2; fnStroke.Transparency = 0.1

local threeLettersBtn = Instance.new("TextButton", nameModeRow)
threeLettersBtn.Size = UDim2.new(0.5, -4, 1, 0)
threeLettersBtn.Position = UDim2.new(0.5, 4, 0, 0)
threeLettersBtn.BackgroundColor3 = Color3.fromRGB(30, 50, 30)
threeLettersBtn.BackgroundTransparency = 0.15
threeLettersBtn.BorderSizePixel = 0
threeLettersBtn.AutoButtonColor = false
threeLettersBtn.Font = Enum.Font.GothamBold
threeLettersBtn.Text = "ثلاث حروف"
threeLettersBtn.TextSize = 14
threeLettersBtn.TextColor3 = Color3.fromRGB(200, 255, 215)
Instance.new("UICorner", threeLettersBtn).CornerRadius = UDim.new(0, 9)
local tlStroke = Instance.new("UIStroke", threeLettersBtn)
tlStroke.Color = Color3.fromRGB(0, 200, 100); tlStroke.Thickness = 1.5; tlStroke.Transparency = 0.4

local function updateNameModeUI()
 if nameMode == "full" then
 TweenService:Create(fullNameBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(0, 160, 80), BackgroundTransparency = 0.05}):Play()
 fnStroke.Transparency = 0.05
 TweenService:Create(threeLettersBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(30, 50, 30), BackgroundTransparency = 0.2}):Play()
 tlStroke.Transparency = 0.5
 else
 TweenService:Create(fullNameBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(30, 50, 30), BackgroundTransparency = 0.2}):Play()
 fnStroke.Transparency = 0.5
 TweenService:Create(threeLettersBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(0, 160, 80), BackgroundTransparency = 0.05}):Play()
 tlStroke.Transparency = 0.05
 end
end

fullNameBtn.MouseButton1Click:Connect(function()
 nameMode = "full"
 updateNameModeUI()
end)
threeLettersBtn.MouseButton1Click:Connect(function()
 nameMode = "three"
 updateNameModeUI()
end)

local silentRow = Instance.new("Frame", spamScroll)
silentRow.BackgroundTransparency = 1
silentRow.Size = UDim2.new(1, 0, 0, 30)
silentRow.LayoutOrder = 2

local silentBtn = Instance.new("TextButton", silentRow)
silentBtn.Size = UDim2.new(1, 0, 1, 0)
silentBtn.Position = UDim2.new(0, 0, 0, 0)
silentBtn.BackgroundColor3 = Color3.fromRGB(18, 10, 38)
silentBtn.BackgroundTransparency = 0.05
silentBtn.BorderSizePixel = 0
silentBtn.AutoButtonColor = false
silentBtn.Font = Enum.Font.GothamBold
silentBtn.Text = "🔇 تشغيل نسخ خفي"
silentBtn.TextSize = 13
silentBtn.TextColor3 = Color3.fromRGB(180, 150, 255)
Instance.new("UICorner", silentBtn).CornerRadius = UDim.new(0, 9)
local silentStroke = Instance.new("UIStroke", silentBtn)
silentStroke.Color = Color3.fromRGB(100, 60, 200)
silentStroke.Thickness = 1.6
silentStroke.Transparency = 0.3

local function updateSilentBtn()
 if silentMode then
 TweenService:Create(silentBtn, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
 BackgroundColor3 = Color3.fromRGB(80, 30, 180),
 BackgroundTransparency = 0.0,
 TextColor3 = Color3.fromRGB(255, 235, 80),
 }):Play()
 silentStroke.Color = Color3.fromRGB(180, 120, 255)
 silentStroke.Thickness = 2.0
 silentStroke.Transparency = 0.0
 silentBtn.Text = "👁 ارجاع للنسخ الظاهر"
 else
 TweenService:Create(silentBtn, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
 BackgroundColor3 = Color3.fromRGB(18, 10, 38),
 BackgroundTransparency = 0.05,
 TextColor3 = Color3.fromRGB(180, 150, 255),
 }):Play()
 silentStroke.Color = Color3.fromRGB(100, 60, 200)
 silentStroke.Thickness = 1.6
 silentStroke.Transparency = 0.3
 silentBtn.Text = "🔇 تشغيل نسخ خفي"
 end
end

task.spawn(function()
 while silentBtn.Parent do
 if silentMode then
 TweenService:Create(silentStroke, TweenInfo.new(0.65, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Transparency = 0.55}):Play()
 task.wait(0.65)
 TweenService:Create(silentStroke, TweenInfo.new(0.65, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Transparency = 0.0}):Play()
 task.wait(0.65)
 else
 task.wait(0.2)
 end
 end
end)

local function makeSpamRow(order)
 local row = Instance.new("Frame", spamScroll)
 row.Size = UDim2.new(1, 0, 0, 42)
 row.BackgroundTransparency = 1
 row.LayoutOrder = order
 return row
end

local function makeRowLabel(parent, text)
 local lbl = Instance.new("TextLabel", parent)
 lbl.Position = UDim2.new(0, 0, 0, 0)
 lbl.Size = UDim2.new(0.34, -4, 1, 0)
 lbl.BackgroundColor3 = Color3.fromRGB(8, 25, 14)
 lbl.BackgroundTransparency = 0.3
 lbl.BorderSizePixel = 0
 lbl.Font = Enum.Font.GothamBold
 lbl.Text = text
 lbl.TextSize = 14
 lbl.TextColor3 = Color3.fromRGB(200, 255, 215)
 lbl.TextXAlignment = Enum.TextXAlignment.Center
 Instance.new("UICorner", lbl).CornerRadius = UDim.new(0, 10)
 return lbl
end

local function makeStartBtn(parent)
 local b = Instance.new("TextButton", parent)
 b.Position = UDim2.new(0.35, 2, 0, 0)
 b.Size = UDim2.new(0.33, -2, 1, 0)
 b.BackgroundColor3 = Color3.fromRGB(180, 25, 25)
 b.BackgroundTransparency = 0.05
 b.BorderSizePixel = 0; b.AutoButtonColor = false
 b.Font = Enum.Font.GothamBold
 b.Text = "تشغيل"
 b.TextSize = 14
 b.TextColor3 = Color3.fromRGB(255, 255, 255)
 Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
 local s = Instance.new("UIStroke", b)
 s.Color = Color3.fromRGB(255, 80, 80)
 s.Transparency = 0.2; s.Thickness = 1.8
 return b
end

local function makeStopBtn(parent)
 local b = Instance.new("TextButton", parent)
 b.Position = UDim2.new(0.69, 3, 0, 0)
 b.Size = UDim2.new(0.31, -3, 1, 0)
 b.BackgroundColor3 = Color3.fromRGB(55, 55, 65)
 b.BackgroundTransparency = 0.05
 b.BorderSizePixel = 0; b.AutoButtonColor = false
 b.Font = Enum.Font.GothamBold
 b.Text = "ايقاف"
 b.TextSize = 14
 b.TextColor3 = Color3.fromRGB(210, 210, 225)
 Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
 b.MouseEnter:Connect(function() TweenService:Create(b, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(80, 80, 95)}):Play() end)
 b.MouseLeave:Connect(function() TweenService:Create(b, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(55, 55, 65)}):Play() end)
 return b
end

local strongCopyCommand = ";explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc ;explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc ;explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc ;explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc ;explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc ;explode ;loopkill ;clogs ;logs ;ice ;loopwarp ;freez ;jc"
local spam2Command = ";wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar ;wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar ;wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar ;wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar ;wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar ;wormify ;ice ;kill ;re ;res ;logs ;clogs ;cmdbar"
local spam3Command = ";frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar ;frogify ;ice ;kill ;nv ;logs ;clogs ;cmdbar"
local spam4Command = ";worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill ;worm ;ice ;jc ;nv ;logs ;clogs ;worm ;ice ;jc ;nv ;logs ;clogs ;kill"

local rowStrong = makeSpamRow(3)
makeRowLabel(rowStrong, "نسخ غامض قوي")
local strongCopyBtn = makeStartBtn(rowStrong)
local stopStrongBtn = makeStopBtn(rowStrong)

local rowSpam2 = makeSpamRow(4)
makeRowLabel(rowSpam2, "نسخ سبام 2")
local spam2Btn = makeStartBtn(rowSpam2)
local stopSpam2Btn = makeStopBtn(rowSpam2)

local rowSpam3 = makeSpamRow(5)
makeRowLabel(rowSpam3, "نسخ سبام3")
local spam3Btn = makeStartBtn(rowSpam3)
local stopSpam3Btn = makeStopBtn(rowSpam3)

local rowSpam4 = makeSpamRow(6)
makeRowLabel(rowSpam4, "نسخ سبام 4")
local spam4Btn = makeStartBtn(rowSpam4)
local stopSpam4Btn = makeStopBtn(rowSpam4)

local rowNvReLogs = makeSpamRow(12)
makeRowLabel(rowNvReLogs, "NV • RE • LOGS")
local nvReLogsBtn = makeStartBtn(rowNvReLogs)
local stopNvReLogsBtn = makeStopBtn(rowNvReLogs)

local function setStartOn(btn)
 TweenService:Create(btn, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(0, 190, 75)}):Play()
 local s = btn:FindFirstChildOfClass("UIStroke")
 if s then s.Color = Color3.fromRGB(0, 255, 130) end
 btn.Text = "شغال ✓"
end
local function setStartOff(btn)
 TweenService:Create(btn, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(180, 25, 25)}):Play()
 local s = btn:FindFirstChildOfClass("UIStroke")
 if s then s.Color = Color3.fromRGB(255, 80, 80) end
 btn.Text = "تشغيل"
end

local rowA = makeSpamRow(7)
makeRowLabel(rowA, "سبام")
local copySpamBtn = makeStartBtn(rowA)
local stopBtnA = makeStopBtn(rowA)

local rowGhost = makeSpamRow(8)
makeRowLabel(rowGhost, "نسخ غامض")
local copyGhostBtn = makeStartBtn(rowGhost)
local stopBtnGhost = makeStopBtn(rowGhost)

local rowB = makeSpamRow(9)
makeRowLabel(rowB, "Logs")
local copyLogsBtn = makeStartBtn(rowB)
local stopBtnB = makeStopBtn(rowB)

local rowC = makeSpamRow(10)
makeRowLabel(rowC, "Re")
local copyReBtn = makeStartBtn(rowC)
local stopBtnC = makeStopBtn(rowC)

local rowD = makeSpamRow(11)
makeRowLabel(rowD, "⚡ سبام قوي")
local copyPowerBtn = makeStartBtn(rowD)
local stopBtnD = makeStopBtn(rowD)

local statusLbl = Instance.new("TextLabel", copyPage)
statusLbl.BackgroundTransparency = 1
statusLbl.Position = UDim2.new(0, 10, 1, -28); statusLbl.Size = UDim2.new(1, -20, 0, 22)
statusLbl.Font = Enum.Font.GothamSemibold; statusLbl.TextSize = 13
statusLbl.TextColor3 = Color3.fromRGB(150, 220, 170); statusLbl.Text = ""

local function setStatus(txt, persistent)
 statusLbl.Text = txt; statusLbl.TextTransparency = 0
 if not persistent then
 task.delay(2.5, function()
 if statusLbl.Text == txt then
 TweenService:Create(statusLbl, TweenInfo.new(0.6), {TextTransparency = 1}):Play()
 end
 end)
 end
end

local function buildLogsCmd(name, prefix)
 prefix = prefix or ";"
 local parts = {}
 for i = 1, 26 do parts[i] = prefix.."logs "..name end
 return table.concat(parts, " ")
end

local function buildReCmd(name, prefix)
 prefix = prefix or ";"
 local parts = {}
 for i = 1, 26 do parts[i] = prefix.."re "..name end
 return table.concat(parts, " ")
end

local function buildSpamA(name, prefix)
 prefix = prefix or ";"
 local block = prefix.."re "..name.." "..prefix.."res "..name.." "..prefix.."logs "..name.." "..prefix.."clogs "..name.." "..prefix.."nv "..name.." "..prefix.."cmdbar "..name.." "..prefix.."jc "..name.." "..prefix.."ice "..name
 local parts = {}
 for i = 1, 8 do parts[i] = block end
 return table.concat(parts, " ")
end

local function buildGhostSpam(name, prefix)
 prefix = prefix or ";"
 local block = prefix.."explode "..name.." "..prefix.."warp "..name.." "..prefix.."logs "..name.." "..prefix.."clogs "..name.." "..prefix.."res "..name.." "..prefix.."ice "..name.." "..prefix.."jc "..name.." "..prefix.."dog "..name.." "..prefix.."char "..name.." miri "..prefix.."fling "..name.." "..prefix.."nv "..name.." "..prefix.."re "..name
 local parts = {}
 for i = 1, 5 do parts[i] = block end
 return table.concat(parts, " ")
end

local function buildPowerSpam(name, prefix)
 prefix = prefix or ";"
 local cmds = {
 "apparate "..name.." inf",
 "fling "..name,
 "jp "..name.." inf",
 "jc "..name,
 "ice "..name,
 "emotes "..name,
 "phase "..name,
 "cmdbar "..name,
 "nv "..name,
 "jump "..name,
 "re "..name,
 "res "..name,
 "kill "..name,
 "ping "..name,
 }
 local out = {}
 for i, c in ipairs(cmds) do out[i] = prefix..c end
 return table.concat(out, " ")
end

local function buildPresetWithName(commandText, name, prefix)
 if not name then return nil end
 prefix = prefix or ";"
 local sourcePrefix = commandText:match("^%s*(%p+)")
 local parts = {}
 for token in commandText:gmatch("%S+") do
  if sourcePrefix and token:sub(1, #sourcePrefix) == sourcePrefix then
  table.insert(parts, prefix .. token:sub(#sourcePrefix + 1) .. " " .. name)
  else
  table.insert(parts, token)
  end
 end
 return #parts > 0 and table.concat(parts, " ") or nil
end

-- Remotes

local chatRemote, hdRemote, changeSettingRemote
pcall(function()
 local re = ReplicatedStorage:FindFirstChild("RemoteEvents")
 if re then chatRemote = re:FindFirstChild("ChatEvent") end
end)
pcall(function()
 local hd = ReplicatedStorage:FindFirstChild("HDAdminHDClient")
 if hd then
 local sig = hd:FindFirstChild("Signals")
 if sig then
 hdRemote = sig:FindFirstChild("RequestCommandModification")
 changeSettingRemote = sig:FindFirstChild("ChangeSetting")
 end
 end
end)

local function applyPrefix(newPrefix)
 if newPrefix == "" then newPrefix = ";" end
 pcall(function()
 if changeSettingRemote then
 local args = {[1] = {[1] = "Prefix", [2] = newPrefix}}
 changeSettingRemote:InvokeServer(unpack(args))
 else
 local sig = game:GetService("ReplicatedStorage")
 :FindFirstChild("HDAdminHDClient")
 :FindFirstChild("Signals")
 if sig then
 local r = sig:FindFirstChild("ChangeSetting")
 if r then
 local args = {[1] = {[1] = "Prefix", [2] = newPrefix}}
 r:InvokeServer(unpack(args))
 end
 end
 end
 end)
 setStatus("✓ علامة الأدمن: " .. newPrefix)
end

prefixBox.FocusLost:Connect(function()
 local val = prefixBox.Text
 if val == "" then val = ";" prefixBox.Text = ";" end
 applyPrefix(val)
end)

local function sendOnce(message)
 if not silentMode then
 pcall(function()
 local chatR = capturedChatRemote
 if not chatR then
 pcall(function() chatR = game:GetService("ReplicatedStorage").Remotes.ChatEvent end)
 end
 if chatR then
 local args = {[1] = message}
 chatR:FireServer(unpack(args))
 end
 end)
 end
 if hdRemote then
 task.spawn(function()
 pcall(function() hdRemote:InvokeServer(message) end)
 end)
 end
end

local function copyText(t)
 local ok = false
 if setclipboard then ok = pcall(setclipboard, t)
 elseif toclipboard then ok = pcall(toclipboard, t)
 elseif type(syn) == "table" and syn.write_clipboard then ok = pcall(syn.write_clipboard, t) end
 return ok
end

local spamRunning = false
local spamThread

local function stopSpam()
 spamRunning = false
 spamThread = nil
 setStatus("تم إيقاف السبام")
end

local function startSpam(message)
 if spamRunning then stopSpam(); task.wait(0.05) end
 spamRunning = true
 setStatus("السبام شغال... اضغط إيقاف للايقاف", true)
 spamThread = task.spawn(function()
 while spamRunning do
 sendOnce(message)
 task.wait(0.05)
 end
 end)
end

local spamARunning = false
local ghostSpamRunning = false
local logsSpamRunning = false
local reSpamRunning = false
local powerSpamRunning = false
local strongCopyRunning = false
local spam2Running = false
local spam3Running = false
local spam4Running = false
local nvReLogsRunning = false
local nvReLogsCommand = ";nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs ;nv ;re ;logs ;clogs"

local function doStop(runFlag, startBtn)
 stopSpam()
 if startBtn then setStartOff(startBtn) end
 return false
end

strongCopyBtn.MouseButton1Click:Connect(function()
 if strongCopyRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 local message = buildPresetWithName(strongCopyCommand, getEffectiveName(), prefix)
 if not message then setStatus("اكتب الأوامر أولاً") return end
 startSpam(message)
 strongCopyRunning = true
 setStartOn(strongCopyBtn)
end)
stopStrongBtn.MouseButton1Click:Connect(function()
 if strongCopyRunning then
  strongCopyRunning = doStop(strongCopyRunning, strongCopyBtn)
 end
end)

spam2Btn.MouseButton1Click:Connect(function()
 if spam2Running then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 local message = buildPresetWithName(spam2Command, getEffectiveName(), prefix)
 if not message then setStatus("اكتب الأوامر أولاً") return end
 startSpam(message)
 spam2Running = true
 setStartOn(spam2Btn)
end)
stopSpam2Btn.MouseButton1Click:Connect(function()
 if spam2Running then
  spam2Running = doStop(spam2Running, spam2Btn)
 end
end)

spam3Btn.MouseButton1Click:Connect(function()
 if spam3Running then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 local message = buildPresetWithName(spam3Command, getEffectiveName(), prefix)
 if not message then setStatus("اكتب الأوامر أولاً") return end
 startSpam(message)
 spam3Running = true
 setStartOn(spam3Btn)
end)
stopSpam3Btn.MouseButton1Click:Connect(function()
 if spam3Running then
  spam3Running = doStop(spam3Running, spam3Btn)
 end
end)

spam4Btn.MouseButton1Click:Connect(function()
 if spam4Running then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 local message = buildPresetWithName(spam4Command, getEffectiveName(), prefix)
 if not message then setStatus("اكتب الأوامر أولاً") return end
 startSpam(message)
 spam4Running = true
 setStartOn(spam4Btn)
end)
stopSpam4Btn.MouseButton1Click:Connect(function()
 if spam4Running then
  spam4Running = doStop(spam4Running, spam4Btn)
 end
end)

nvReLogsBtn.MouseButton1Click:Connect(function()
 if nvReLogsRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 local message = buildPresetWithName(nvReLogsCommand, getEffectiveName(), prefix)
 if not message then setStatus("اكتب الأوامر أولاً") return end
 startSpam(message)
 nvReLogsRunning = true
 setStartOn(nvReLogsBtn)
end)
stopNvReLogsBtn.MouseButton1Click:Connect(function()
 if nvReLogsRunning then
  nvReLogsRunning = doStop(nvReLogsRunning, nvReLogsBtn)
 end
end)

copySpamBtn.MouseButton1Click:Connect(function()
 if spamARunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 startSpam(buildSpamA(getEffectiveName(), prefix))
 spamARunning = true; setStartOn(copySpamBtn)
end)
stopBtnA.MouseButton1Click:Connect(function()
 if spamARunning then spamARunning = doStop(spamARunning, copySpamBtn) end
end)

copyGhostBtn.MouseButton1Click:Connect(function()
 if ghostSpamRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 startSpam(buildGhostSpam(getEffectiveName(), prefix))
 ghostSpamRunning = true; setStartOn(copyGhostBtn)
end)
stopBtnGhost.MouseButton1Click:Connect(function()
 if ghostSpamRunning then ghostSpamRunning = doStop(ghostSpamRunning, copyGhostBtn) end
end)

copyLogsBtn.MouseButton1Click:Connect(function()
 if logsSpamRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 startSpam(buildLogsCmd(getEffectiveName(), prefix))
 logsSpamRunning = true; setStartOn(copyLogsBtn)
end)
stopBtnB.MouseButton1Click:Connect(function()
 if logsSpamRunning then logsSpamRunning = doStop(logsSpamRunning, copyLogsBtn) end
end)

copyReBtn.MouseButton1Click:Connect(function()
 if reSpamRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 startSpam(buildReCmd(getEffectiveName(), prefix))
 reSpamRunning = true; setStartOn(copyReBtn)
end)
stopBtnC.MouseButton1Click:Connect(function()
 if reSpamRunning then reSpamRunning = doStop(reSpamRunning, copyReBtn) end
end)

copyPowerBtn.MouseButton1Click:Connect(function()
 if powerSpamRunning then return end
 if not selectedName then setStatus("اختر لاعب اولا") return end
 local prefix = (prefixBox.Text ~= "" and prefixBox.Text) or ";"
 startSpam(buildPowerSpam(getEffectiveName(), prefix))
 powerSpamRunning = true; setStartOn(copyPowerBtn)
end)
stopBtnD.MouseButton1Click:Connect(function()
 if powerSpamRunning then powerSpamRunning = doStop(powerSpamRunning, copyPowerBtn) end
end)

silentBtn.MouseButton1Click:Connect(function()
 silentMode = not silentMode
 updateSilentBtn()
 if silentMode then
 setStatus("🔇 النسخ الخفي شغال — أدمن فقط", true)
 statusLbl.TextColor3 = Color3.fromRGB(200, 160, 255)
 else
 statusLbl.TextColor3 = Color3.fromRGB(150, 220, 170)
 setStatus("👁 رجع للنسخ الظاهر — شات + أدمن")
 end
end)

-- Page: تحكم

local ctrlInfo = Instance.new("TextLabel", controlPage)
ctrlInfo.BackgroundTransparency = 1
ctrlInfo.Position = UDim2.new(0, 10, 0, 10); ctrlInfo.Size = UDim2.new(1, -20, 0, 24)
ctrlInfo.Font = Enum.Font.GothamBold; ctrlInfo.Text = "لوحة التحكم"
ctrlInfo.TextSize = 16; ctrlInfo.TextColor3 = Color3.fromRGB(0, 255, 130)
ctrlInfo.TextXAlignment = Enum.TextXAlignment.Left

local ctrlScroll = Instance.new("ScrollingFrame", controlPage)
ctrlScroll.Position = UDim2.new(0, 0, 0, 40)
ctrlScroll.Size = UDim2.new(1, 0, 1, -70)
ctrlScroll.BackgroundTransparency = 1
ctrlScroll.BorderSizePixel = 0
ctrlScroll.ScrollBarThickness = 4
ctrlScroll.ScrollBarImageColor3 = Color3.fromRGB(0, 200, 100)
ctrlScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
ctrlScroll.CanvasSize = UDim2.new(0, 0, 0, 0)

local protectionBtn = makeBigBtn(ctrlScroll, "حماية 🛡️", 4,
 Color3.fromRGB(0, 170, 220), Color3.fromRGB(0, 90, 150))
local spamBtn = makeBigBtn(ctrlScroll, "سبام", 58,
 Color3.fromRGB(255, 80, 80), Color3.fromRGB(170, 30, 30))
local skinsBtn = makeBigBtn(ctrlScroll, "سكنات", 112,
 Color3.fromRGB(255, 90, 200), Color3.fromRGB(170, 30, 130))
local copySkinsBtn = makeBigBtn(ctrlScroll, "نسخ سكنات", 598,
 Color3.fromRGB(220, 50, 50), Color3.fromRGB(150, 20, 20))
local dancesBtn = makeBigBtn(ctrlScroll, "رقصات", 166,
 Color3.fromRGB(230, 140, 30), Color3.fromRGB(160, 90, 10))
local loadBtn = makeBigBtn(ctrlScroll, "تحكم الراديو", 220,
 Color3.fromRGB(0, 200, 110), Color3.fromRGB(0, 130, 70))
local spinStartBtn = makeBigBtn(ctrlScroll, "تشغيل الدوران", 274,
 Color3.fromRGB(140, 220, 40), Color3.fromRGB(80, 150, 20))
local spinStopBtn = makeBigBtn(ctrlScroll, "إيقاف الدوران", 328,
 Color3.fromRGB(170, 30, 30), Color3.fromRGB(110, 15, 15))
local titleBtn = makeBigBtn(ctrlScroll, "تلوين الشات🖍️", 382,
 Color3.fromRGB(170, 70, 220), Color3.fromRGB(100, 30, 150))
local allBtn = makeBigBtn(ctrlScroll, "نسخ all", 436,
 Color3.fromRGB(30, 180, 255), Color3.fromRGB(10, 100, 180))
local blueBtn = makeBigBtn(ctrlScroll, "نسخه معدلة من سكربت بلو", 490,
 Color3.fromRGB(0, 120, 255), Color3.fromRGB(0, 60, 160))
local afkBtn = makeBigBtn(ctrlScroll, "تحكم في شات ال afk أو أحد مايعرف يسولف", 544,
 Color3.fromRGB(255, 160, 0), Color3.fromRGB(180, 90, 0))
local joinLogsBtn = makeBigBtn(ctrlScroll, "لوقس دخول وخروج", 652,
 Color3.fromRGB(80, 180, 255), Color3.fromRGB(20, 90, 180))
local changeServerBtn = makeBigBtn(ctrlScroll, "تغير السيرفر", 706,
 Color3.fromRGB(180, 80, 255), Color3.fromRGB(90, 30, 160))

local ctrlStatus = Instance.new("TextLabel", controlPage)
ctrlStatus.BackgroundTransparency = 1
ctrlStatus.Position = UDim2.new(0, 10, 1, -28); ctrlStatus.Size = UDim2.new(1, -20, 0, 22)
ctrlStatus.Font = Enum.Font.GothamSemibold; ctrlStatus.TextSize = 13
ctrlStatus.TextColor3 = Color3.fromRGB(150, 220, 170); ctrlStatus.Text = ""
ctrlStatus.TextXAlignment = Enum.TextXAlignment.Left

local function showBigNotice(text)
 local nGui = Instance.new("ScreenGui")
 nGui.Name = "SHNotice"; nGui.ResetOnSpawn = false; nGui.IgnoreGuiInset = true
 nGui.DisplayOrder = 9998
 pcall(function() nGui.Parent = guiParent end)
 if not nGui.Parent then nGui.Parent = PlayerGui end

 local nFrame = Instance.new("Frame", nGui)
 nFrame.AnchorPoint = Vector2.new(0.5, 0.5)
 nFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
 nFrame.Size = UDim2.new(0, 520, 0, 220)
 nFrame.BackgroundColor3 = Color3.fromRGB(8, 18, 10)
 nFrame.BackgroundTransparency = 0.1
 nFrame.BorderSizePixel = 0
 Instance.new("UICorner", nFrame).CornerRadius = UDim.new(0, 16)
 local nStroke = Instance.new("UIStroke", nFrame)
 nStroke.Color = Color3.fromRGB(0, 255, 130); nStroke.Thickness = 2; nStroke.Transparency = 0.1

 local title = Instance.new("TextLabel", nFrame)
 title.BackgroundTransparency = 1
 title.Position = UDim2.new(0, 12, 0, 14); title.Size = UDim2.new(1, -24, 0, 32)
 title.Font = Enum.Font.GothamBlack; title.Text = "ملاحظة مهمة"
 title.TextSize = 22; title.TextColor3 = Color3.fromRGB(0, 255, 130)

 local body = Instance.new("TextLabel", nFrame)
 body.BackgroundTransparency = 1
 body.Position = UDim2.new(0, 16, 0, 56); body.Size = UDim2.new(1, -32, 1, -116)
 body.Font = Enum.Font.GothamSemibold; body.Text = text
 body.TextSize = 20; body.TextColor3 = Color3.fromRGB(230, 255, 235)
 body.TextWrapped = true; body.TextYAlignment = Enum.TextYAlignment.Top

 local okBtn = Instance.new("TextButton", nFrame)
 okBtn.AnchorPoint = Vector2.new(0.5, 1)
 okBtn.Position = UDim2.new(0.5, 0, 1, -14); okBtn.Size = UDim2.new(0, 160, 0, 38)
 okBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100); okBtn.BorderSizePixel = 0
 okBtn.Font = Enum.Font.GothamBold; okBtn.Text = "تمام"
 okBtn.TextSize = 18; okBtn.TextColor3 = Color3.fromRGB(230, 255, 235)
 okBtn.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
 Instance.new("UICorner", okBtn).CornerRadius = UDim.new(0, 10)

 okBtn.MouseButton1Click:Connect(function() pcall(function() nGui:Destroy() end) end)
 task.delay(12, function() pcall(function() nGui:Destroy() end) end)
end

local protectionLoaded = false
protectionBtn.MouseButton1Click:Connect(function()
 if protectionLoaded then
  ctrlStatus.Text = "الحماية مفعلة بالفعل"
  return
 end
 ctrlStatus.Text = "جاري تشغيل الحماية..."
 task.spawn(function()
  local ok, err = pcall(function()
   loadstring(game:HttpGet("https://raw.githubusercontent.com/Shhd-code/HMAIA/refs/heads/main/README.md"))()
  end)
  if ok then
   protectionLoaded = true
   ctrlStatus.Text = "تم تشغيل الحماية 🛡️"
  else
   ctrlStatus.Text = "فشل تشغيل الحماية: " .. tostring(err):sub(1, 60)
  end
 end)
end)

local joinLogsLoaded = false
joinLogsBtn.MouseButton1Click:Connect(function()
 if joinLogsLoaded then
  ctrlStatus.Text = "لوقس الدخول والخروج مفعلة بالفعل"
  return
 end
 ctrlStatus.Text = "جاري تشغيل لوقس الدخول والخروج..."
 task.spawn(function()
  local ok, err = pcall(function()
   loadstring(game:HttpGet("https://raw.githubusercontent.com/Shhd-code/Log/refs/heads/main/README.md"))()
  end)
  if ok then
   joinLogsLoaded = true
   ctrlStatus.Text = "تم تشغيل لوقس الدخول والخروج"
  else
   ctrlStatus.Text = "فشل تشغيل اللوقس: " .. tostring(err):sub(1, 60)
  end
 end)
end)

local changeServerLoaded = false
changeServerBtn.MouseButton1Click:Connect(function()
 if changeServerLoaded then
  ctrlStatus.Text = "تغير السيرفر مفعل بالفعل"
  return
 end
 ctrlStatus.Text = "جاري تشغيل تغير السيرفر..."
 task.spawn(function()
  local ok, err = pcall(function()
   loadstring(game:HttpGet("https://raw.githubusercontent.com/Shhd-code/RJ/refs/heads/main/README.md"))()
  end)
  if ok then
   changeServerLoaded = true
   ctrlStatus.Text = "تم تشغيل تغير السيرفر"
  else
   ctrlStatus.Text = "فشل تشغيل تغير السيرفر: " .. tostring(err):sub(1, 60)
  end
 end)
end)

local dancesLoaded = false
dancesBtn.MouseButton1Click:Connect(function()
 if dancesLoaded then ctrlStatus.Text = "الرقصات مفعلة بالفعل" return end
 ctrlStatus.Text = "جاري تشغيل الرقصات..."
 task.spawn(function()
 local ok, err = pcall(function()
 loadstring(game:HttpGet("https://raw.githubusercontent.com/7yd7/Hub/refs/heads/Branch/GUIS/Emotes.lua"))()
 end)
 if ok then dancesLoaded = true; ctrlStatus.Text = "تم تشغيل الرقصات"
 else ctrlStatus.Text = "فشل: " .. tostring(err):sub(1, 60) end
 end)
end)

local controlLoaded = false
loadBtn.MouseButton1Click:Connect(function()
 showBigNotice("البس الراديو يلا يشتغل السكربت 🙌😌")
 if controlLoaded then ctrlStatus.Text = "تحكم الراديو مفعل بالفعل" return end
 ctrlStatus.Text = "جاري تشغيل الراديو..."
 task.spawn(function()
 local ok, err = pcall(function()
 loadstring(game:HttpGet("https://raw.githubusercontent.com/Shhd-code/Raduooo/refs/heads/main/README.md"))()
 end)
 if ok then controlLoaded = true; ctrlStatus.Text = "تم تشغيل تحكم الراديو"
 else ctrlStatus.Text = "فشل التشغيل: " .. tostring(err):sub(1, 60) end
 end)
end)

local spinning = false
local spinSpeed = 50
local RunService = game:GetService("RunService")

RunService.Heartbeat:Connect(function()
 if spinning then
 local char = LocalPlayer.Character
 if char and char:FindFirstChild("HumanoidRootPart") then
 char.HumanoidRootPart.CFrame = char.HumanoidRootPart.CFrame * CFrame.Angles(0, math.rad(spinSpeed), 0)
 end
 end
end)

spinStartBtn.MouseButton1Click:Connect(function()
 spinning = true
 ctrlStatus.Text = "تم تشغيل الدوران"
end)
spinStopBtn.MouseButton1Click:Connect(function()
 spinning = false
 ctrlStatus.Text = "تم إيقاف الدوران"
end)

-- ═══════════════════════════════════════════════════════
-- [قسم ريموت — Chat Remote Library]
-- ═══════════════════════════════════════════════════════
local remoteCapturing = false
local remoteLibrary = {}
local remoteActivePath = nil
local remoteScroll, remoteLayout, remoteStatusLbl
local remoteWelcomeGui = nil
local remoteStatusText = Instance.new("TextLabel", remotePage)

local function rFileRead(n)
 if not (readfile and isfile) then return nil end
 local ok, d = pcall(readfile, n); return ok and d or nil
end
local function rFileWrite(n, d)
 if not writefile then return end
 pcall(writefile, n, d)
end

local function rFindByPath(path)
 if not path or path == "" then return nil end
 local obj = game
 for seg in path:gmatch("[^%.]+") do
 obj = obj and obj:FindFirstChild(seg)
 if not obj then return nil end
 end
 return obj
end

local function rLoad()
 local data = rFileRead("SH_ChatLibrary.txt")
 if data then
 for line in data:gmatch("[^\r\n]+") do
 local path, label = line:match("^(.-)|(.*)$")
 if path and label and path ~= "" then
 table.insert(remoteLibrary, { path = path, label = label })
 end
 end
 end
 local active = rFileRead("SH_ChatRemote.txt")
 if active and active ~= "" then
 local obj = rFindByPath(active)
 if obj and (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction")) then
 capturedChatRemote = obj
 remoteActivePath = active
 end
 end
end

local function rSaveLib()
 local lines = {}
 for _, item in ipairs(remoteLibrary) do
 table.insert(lines, item.path .. "|" .. item.label)
 end
 rFileWrite("SH_ChatLibrary.txt", table.concat(lines, "\n"))
end

local function rAdd(remote)
 if not remote then return end
 local p = remote:GetFullName()
 for _, it in ipairs(remoteLibrary) do
 if it.path == p then return end
 end
 table.insert(remoteLibrary, { path = p, label = remote.Name })
 rSaveLib()
end

local function rRemove(p)
 for i = #remoteLibrary, 1, -1 do
 if remoteLibrary[i].path == p then
 table.remove(remoteLibrary, i)
 rSaveLib()
 return
 end
 end
end

local function rSetActive(remote)
 if not remote then return end
 capturedChatRemote = remote
 remoteActivePath = remote:GetFullName()
 rFileWrite("SH_ChatRemote.txt", remoteActivePath)
end

local function rClearActive()
 capturedChatRemote = nil
 remoteActivePath = nil
 rFileWrite("SH_ChatRemote.txt", "")
end

local function rRebuild()
 if not remoteScroll or not remoteLayout then return end
 for _, c in ipairs(remoteScroll:GetChildren()) do
 if c:IsA("Frame") or c:IsA("TextLabel") then c:Destroy() end
 end
 
 if remoteStatusLbl then
 local a = capturedChatRemote and capturedChatRemote:GetFullName() or "لا يوجد"
 remoteStatusLbl.Text = "⭐ النشط: " .. a
 remoteStatusLbl.TextColor3 = capturedChatRemote and Color3.fromRGB(0,255,130) or Color3.fromRGB(255,120,120)
 end
 
 if #remoteLibrary == 0 then
 local e = Instance.new("TextLabel")
 e.Size = UDim2.new(1, -8, 0, 50)
 e.BackgroundTransparency = 1
 e.Text = "📭 المكتبة فارغة\nاضغط زر التقاط واكتب (شهد) أو (شات)"
 e.TextColor3 = Color3.fromRGB(150,200,170)
 e.TextSize = 12
 e.Font = Enum.Font.Gotham
 e.TextWrapped = true
 e.Parent = remoteScroll
 else
 for _, item in ipairs(remoteLibrary) do
 local isActive = (remoteActivePath == item.path)
 local exists = rFindByPath(item.path)
 
 local row = Instance.new("Frame")
 row.Size = UDim2.new(1, -6, 0, 30)
 row.BackgroundColor3 = isActive and Color3.fromRGB(20,60,35) or Color3.fromRGB(8,30,14)
 row.BackgroundTransparency = 0.2
 row.BorderSizePixel = 0
 row.Parent = remoteScroll
 Instance.new("UICorner", row).CornerRadius = UDim.new(0, 7)
 local rs = Instance.new("UIStroke", row)
 rs.Color = isActive and Color3.fromRGB(0,255,130) or Color3.fromRGB(0,150,70)
 rs.Transparency = 0.4
 
 local lbl = Instance.new("TextLabel")
 lbl.Size = UDim2.new(1, -70, 1, 0)
 lbl.Position = UDim2.new(0, 8, 0, 0)
 lbl.BackgroundTransparency = 1
 local pre = isActive and "⭐ " or (exists and "📡 " or "🚫 ")
 lbl.Text = pre .. item.label
 lbl.TextColor3 = isActive and Color3.fromRGB(150,255,190) or (exists and Color3.fromRGB(220,250,230) or Color3.fromRGB(150,150,150))
 lbl.TextSize = 12
 lbl.Font = Enum.Font.GothamSemibold
 lbl.TextXAlignment = Enum.TextXAlignment.Left
 lbl.TextTruncate = Enum.TextTruncate.AtEnd
 lbl.Parent = row
 
 local actBtn = Instance.new("TextButton")
 actBtn.Size = UDim2.new(0, 26, 0, 22)
 actBtn.Position = UDim2.new(1, -62, 0, 4)
 actBtn.BackgroundColor3 = isActive and Color3.fromRGB(0,190,90) or Color3.fromRGB(0,140,70)
 actBtn.Text = isActive and "✅" or "▶"
 actBtn.TextColor3 = Color3.new(1,1,1)
 actBtn.TextSize = 11
 actBtn.Font = Enum.Font.GothamBold
 actBtn.BorderSizePixel = 0
 actBtn.AutoButtonColor = false
 actBtn.Parent = row
 Instance.new("UICorner", actBtn).CornerRadius = UDim.new(0, 5)
 actBtn.MouseButton1Click:Connect(function()
 local r = rFindByPath(item.path)
 if not r then setStatus("❌ الريموت غير موجود") return end
 rSetActive(r)
 rRebuild()
 setStatus("⭐ فُعّل: " .. r.Name)
 end)
 
 local delBtn = Instance.new("TextButton")
 delBtn.Size = UDim2.new(0, 26, 0, 22)
 delBtn.Position = UDim2.new(1, -32, 0, 4)
 delBtn.BackgroundColor3 = Color3.fromRGB(180,35,35)
 delBtn.Text = "🗑"
 delBtn.TextColor3 = Color3.new(1,1,1)
 delBtn.TextSize = 11
 delBtn.Font = Enum.Font.GothamBold
 delBtn.BorderSizePixel = 0
 delBtn.AutoButtonColor = false
 delBtn.Parent = row
 Instance.new("UICorner", delBtn).CornerRadius = UDim.new(0, 5)
 delBtn.MouseButton1Click:Connect(function()
 rRemove(item.path)
 if remoteActivePath == item.path then rClearActive() end
 rRebuild()
 setStatus("🗑️ حُذف من المكتبة")
 end)
 end
 end
 
 task.defer(function()
 if remoteScroll and remoteLayout then
 remoteScroll.CanvasSize = UDim2.new(0, 0, 0, remoteLayout.AbsoluteContentSize.Y + 10)
 end
 end)
end

remoteStatusLbl = Instance.new("TextLabel", remotePage)
remoteStatusLbl.BackgroundTransparency = 1
remoteStatusLbl.Position = UDim2.new(0, 10, 0, 8)
remoteStatusLbl.Size = UDim2.new(1, -20, 0, 30)
remoteStatusLbl.Font = Enum.Font.GothamSemibold
remoteStatusLbl.TextSize = 12
remoteStatusLbl.TextColor3 = Color3.fromRGB(150,220,170)
remoteStatusLbl.TextXAlignment = Enum.TextXAlignment.Left
remoteStatusLbl.Text = "⭐ النشط: لا يوجد"
remoteStatusLbl.TextWrapped = true
remoteStatusLbl.TextTruncate = Enum.TextTruncate.AtEnd

remoteScroll = Instance.new("ScrollingFrame", remotePage)
remoteScroll.Position = UDim2.new(0, 10, 0, 42)
remoteScroll.Size = UDim2.new(1, -20, 1, -110)
remoteScroll.BackgroundColor3 = Color3.fromRGB(2,10,5)
remoteScroll.BackgroundTransparency = 0.5
remoteScroll.BorderSizePixel = 0
remoteScroll.ScrollBarThickness = 4
remoteScroll.ScrollBarImageColor3 = Color3.fromRGB(0,255,130)
remoteScroll.CanvasSize = UDim2.new(0,0,0,0)
Instance.new("UICorner", remoteScroll).CornerRadius = UDim.new(0, 8)

remoteLayout = Instance.new("UIListLayout", remoteScroll)
remoteLayout.Padding = UDim.new(0, 4)
remoteLayout.SortOrder = Enum.SortOrder.LayoutOrder

local rp = Instance.new("UIPadding", remoteScroll)
rp.PaddingTop = UDim.new(0, 6); rp.PaddingBottom = UDim.new(0, 6)
rp.PaddingLeft = UDim.new(0, 6); rp.PaddingRight = UDim.new(0, 6)

local captureRemoteBtn = Instance.new("TextButton", remotePage)
captureRemoteBtn.Position = UDim2.new(0, 10, 1, -60)
captureRemoteBtn.Size = UDim2.new(1, -20, 0, 44)
captureRemoteBtn.BackgroundColor3 = Color3.fromRGB(0,140,70)
captureRemoteBtn.BackgroundTransparency = 0.1
captureRemoteBtn.BorderSizePixel = 0
captureRemoteBtn.AutoButtonColor = false
captureRemoteBtn.Font = Enum.Font.GothamBold
captureRemoteBtn.Text = "🔍 التقاط ريموت جديد (شهد / شات)"
captureRemoteBtn.TextSize = 14
captureRemoteBtn.TextColor3 = Color3.new(1,1,1)
Instance.new("UICorner", captureRemoteBtn).CornerRadius = UDim.new(0, 10)
local crs = Instance.new("UIStroke", captureRemoteBtn)
crs.Color = Color3.fromRGB(0,255,130); crs.Thickness = 1.8; crs.Transparency = 0.25
captureRemoteBtn.MouseEnter:Connect(function() TweenService:Create(captureRemoteBtn, TweenInfo.new(0.12), {BackgroundTransparency=0}):Play() end)
captureRemoteBtn.MouseLeave:Connect(function() TweenService:Create(captureRemoteBtn, TweenInfo.new(0.12), {BackgroundTransparency=0.1}):Play() end)

local function closeRemoteWelcome()
 if remoteWelcomeGui then
 pcall(function() remoteWelcomeGui:Destroy() end)
 remoteWelcomeGui = nil
 end
 remoteCapturing = false
end

local function showRemoteWelcome()
 if remoteWelcomeGui then pcall(function() remoteWelcomeGui:Destroy() end) end
 remoteWelcomeGui = Instance.new("ScreenGui")
 remoteWelcomeGui.Name = "SH_RemoteWelcome"
 remoteWelcomeGui.ResetOnSpawn = false
 remoteWelcomeGui.IgnoreGuiInset = true
 remoteWelcomeGui.DisplayOrder = 9999
 pcall(function() remoteWelcomeGui.Parent = guiParent end)
 if not remoteWelcomeGui.Parent then remoteWelcomeGui.Parent = PlayerGui end

 local card = Instance.new("Frame", remoteWelcomeGui)
 card.AnchorPoint = Vector2.new(0.5, 0.5)
 card.Position = UDim2.new(0.5, 0, 0.35, 0)
 card.Size = UDim2.new(0, 340, 0, 140)
 card.BackgroundColor3 = Color3.fromRGB(6,16,9)
 card.BackgroundTransparency = 0.1
 card.BorderSizePixel = 0
 Instance.new("UICorner", card).CornerRadius = UDim.new(0, 12)
 local cs = Instance.new("UIStroke", card)
 cs.Color = Color3.fromRGB(0,255,130); cs.Thickness = 1.6; cs.Transparency = 0.2

 local ttl = Instance.new("TextLabel", card)
 ttl.BackgroundTransparency = 1
 ttl.Position = UDim2.new(0, 12, 0, 6); ttl.Size = UDim2.new(1, -48, 0, 32)
 ttl.Font = Enum.Font.GothamBlack
 ttl.Text = "⚡ التقاط ريموت الشات ⚡"
 ttl.TextSize = 16
 ttl.TextColor3 = Color3.fromRGB(0,255,130)
 ttl.TextXAlignment = Enum.TextXAlignment.Left

 local xBtn = Instance.new("TextButton", card)
 xBtn.Position = UDim2.new(1, -32, 0, 6)
 xBtn.Size = UDim2.new(0, 26, 0, 26)
 xBtn.BackgroundColor3 = Color3.fromRGB(180,35,35)
 xBtn.BorderSizePixel = 0
 xBtn.Text = "✕"
 xBtn.Font = Enum.Font.GothamBold
 xBtn.TextColor3 = Color3.new(1,1,1)
 xBtn.TextSize = 13
 xBtn.AutoButtonColor = false
 Instance.new("UICorner", xBtn).CornerRadius = UDim.new(0, 6)
 xBtn.MouseButton1Click:Connect(function()
 closeRemoteWelcome()
 setStatus("تم إلغاء الالتقاط")
 end)

 local body = Instance.new("TextLabel", card)
 body.BackgroundTransparency = 1
 body.Position = UDim2.new(0, 12, 0, 42); body.Size = UDim2.new(1, -24, 0, 56)
 body.RichText = true
 body.Text = 'اكتب <font color="#00FF82"><b>شهد</b></font> أو <font color="#00FF82"><b>شات</b></font> في الشات العام الآن...\nسيتم التقاط ريموت الشات وحفظه في المكتبة.'
 body.TextColor3 = Color3.fromRGB(220,250,230)
 body.TextSize = 12
 body.Font = Enum.Font.GothamSemibold
 body.TextWrapped = true
 body.TextXAlignment = Enum.TextXAlignment.Left
 body.TextYAlignment = Enum.TextYAlignment.Top

 local st = Instance.new("TextLabel", card)
 st.BackgroundTransparency = 1
 st.Position = UDim2.new(0, 12, 1, -26); st.Size = UDim2.new(1, -24, 0, 18)
 st.Text = "⏳ بانتظار الكلمة..."
 st.TextColor3 = Color3.fromRGB(150,220,170)
 st.TextSize = 11
 st.Font = Enum.Font.Gotham
 st.TextXAlignment = Enum.TextXAlignment.Left
end

captureRemoteBtn.MouseButton1Click:Connect(function()
 remoteCapturing = true
 showRemoteWelcome()
end)

if hookmetamethod then
 local oldNameCall
 oldNameCall = hookmetamethod(game, "__namecall", function(self, ...)
 local method = getnamecallmethod()
 local args = { ... }
 if remoteCapturing and (method == "FireServer" or method == "InvokeServer") then
 local okIs, isRemote = pcall(function()
 return self:IsA("RemoteEvent") or self:IsA("RemoteFunction")
 end)
 if okIs and isRemote then
 local fullName = self:GetFullName():lower()
 local remoteName = self.Name:lower()
 if fullName:find("event") then
 if not (remoteName:find("move") or remoteName:find("pos") or remoteName:find("camera") or remoteName:find("mouse")) then
 for _, arg in ipairs(args) do
 if type(arg) == "string" then
 local low = arg:lower()
 local matched = low:find("شهد") or low:find("شات") or low:find("shahd") or low:find("%f[%a]chat%f[%A]") or low:find("\216\180")
 if matched then
 remoteCapturing = false
 capturedChatRemote = self
 rAdd(self)
 rSetActive(self)
 rRebuild()
 closeRemoteWelcome()
 setStatus("✅ التُقط: " .. self.Name)
 break
 end
 end
 end
 end
 end
 end
 end
 return oldNameCall(self, ...)
 end)
end

rLoad()
rRebuild()

-- Page: سبام اكسترا

local extraInfo = Instance.new("TextLabel", extraPage)
extraInfo.BackgroundTransparency = 1
extraInfo.Position = UDim2.new(0, 10, 0, 10); extraInfo.Size = UDim2.new(1, -20, 0, 24)
extraInfo.Font = Enum.Font.GothamBold; extraInfo.Text = "سبام اكسترا"
extraInfo.TextSize = 16; extraInfo.TextColor3 = Color3.fromRGB(0, 255, 130)
extraInfo.TextXAlignment = Enum.TextXAlignment.Left

-- Default tab

pages["نسخ"].Visible = true
TweenService:Create(tabButtons["نسخ"], TweenInfo.new(0.15), {BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(0, 255, 130)}):Play()
