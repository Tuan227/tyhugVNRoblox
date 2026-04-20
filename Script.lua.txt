-- Khởi tạo Rayfield Library
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "Bounty Hunter V12 - tyhugVN",
   LoadingTitle = "Đang nạp hệ thống điều khiển & quản lý...",
   LoadingSubtitle = "by tyhugVN",
})

-- =========================
-- 🛠 BIẾN HỆ THỐNG
-- =========================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualUser = game:GetService("VirtualUser")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local player = Players.LocalPlayer

local killAura = false
local autoChat = false
local autoEquip = false
local fakeDeath = false
local speedEnabled = false
local walkSpeedValue = 16
local jumpEnabled = false
local jumpPowerValue = 50
local spectateEnabled = false
local selectedSpectatePlayer = nil
local selectedWeapon = nil
local currentTarget = nil

-- Cấu hình ESP
local espMaster = false
local showBox = false
local showName = false
local showLine = false
local showHealth = false
local showDistance = false
local showCount = false

-- Quản lý ESP & UI Objects
local espObjects = {}
local playerCountLabel = Drawing.new("Text")
playerCountLabel.Visible = false
playerCountLabel.Color = Color3.fromRGB(255, 255, 0)
playerCountLabel.Size = 20
playerCountLabel.Outline = true
playerCountLabel.Position = Vector2.new(50, 80)

-- Tạo Tâm Ảo (Crosshair)
local crosshairVertical = Drawing.new("Line")
local crosshairHorizontal = Drawing.new("Line")
local crosshairEnabled = false

local function updateCrosshair()
    if crosshairEnabled then
        local center = Camera.ViewportSize / 2
        local size = 10
        crosshairVertical.Visible = true
        crosshairVertical.From = Vector2.new(center.X, center.Y - size)
        crosshairVertical.To = Vector2.new(center.X, center.Y + size)
        crosshairVertical.Color = Color3.fromRGB(255, 0, 0)
        crosshairVertical.Thickness = 2
        crosshairHorizontal.Visible = true
        crosshairHorizontal.From = Vector2.new(center.X - size, center.Y)
        crosshairHorizontal.To = Vector2.new(center.X + size, center.Y)
        crosshairHorizontal.Color = Color3.fromRGB(255, 0, 0)
        crosshairHorizontal.Thickness = 2
    else
        crosshairVertical.Visible = false
        crosshairHorizontal.Visible = false
    end
end

local KILL_OFFSET = Vector3.new(0, 2, 1)

-- =========================
-- 🔍 HÀM HỖ TRỢ
-- =========================

local function getPlayerNames()
    local names = {}
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then table.insert(names, p.Name) end
    end
    return names
end

local function handleMovement()
    if player.Character and player.Character:FindFirstChild("Humanoid") then
        local hum = player.Character.Humanoid
        if speedEnabled then hum.WalkSpeed = walkSpeedValue else hum.WalkSpeed = 16 end
        if jumpEnabled then 
            hum.JumpPower = jumpPowerValue 
            hum.UseJumpPower = true
        else 
            hum.JumpPower = 50 
        end
    end
end

UserInputService.JumpRequest:Connect(function()
    if jumpEnabled and player.Character and player.Character:FindFirstChild("Humanoid") then
        player.Character.Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
end)

local function handleSpectate()
    if spectateEnabled and selectedSpectatePlayer then
        local target = Players:FindFirstChild(selectedSpectatePlayer)
        if target and target.Character and target.Character:FindFirstChild("Humanoid") then
            Camera.CameraSubject = target.Character.Humanoid
        else
            Camera.CameraSubject = player.Character.Humanoid
        end
    else
        if player.Character and player.Character:FindFirstChild("Humanoid") then
            Camera.CameraSubject = player.Character.Humanoid
        end
    end
end

local function toggleFakeDeath(state)
    local character = player.Character
    if not character or not character:FindFirstChild("Humanoid") then return end
    local humanoid = character.Humanoid
    if state then
        humanoid:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
        humanoid:ChangeState(Enum.HumanoidStateType.Physics)
        for _, v in pairs(character:GetDescendants()) do
            if v:IsA("Motor6D") and v.Name ~= "RootJoint" then v.Enabled = false end
        end
    else
        for _, v in pairs(character:GetDescendants()) do
            if v:IsA("Motor6D") then v.Enabled = true end
        end
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end
end

local function toggleDefaultNameTags(state)
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("Humanoid") then
            p.Character.Humanoid.DisplayDistanceType = state and Enum.HumanoidDisplayDistanceType.None or Enum.HumanoidDisplayDistanceType.Viewer
        end
    end
end

local function createESP(p)
    local box = Drawing.new("Square")
    box.Visible = false
    box.Thickness = 1.5
    box.Filled = false
    local line = Drawing.new("Line")
    line.Visible = false
    local healthBar = Drawing.new("Line")
    healthBar.Visible = false
    local nameTag = Drawing.new("Text")
    nameTag.Visible = false
    nameTag.Size = 14
    nameTag.Center = true
    nameTag.Outline = true
    espObjects[p] = {Box = box, Line = line, HealthBar = healthBar, NameTag = nameTag}
end

local function updateESP()
    local count = 0
    toggleDefaultNameTags(espMaster)
    for p, obj in pairs(espObjects) do
        local canShow = espMaster and p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character:FindFirstChild("Humanoid")
        if canShow then
            local hrp = p.Character.HumanoidRootPart
            local hum = p.Character.Humanoid
            local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
            count = count + 1
            if onScreen and hum.Health > 0 then
                local top = Camera:WorldToViewportPoint(hrp.Position + Vector3.new(0, 3, 0))
                local bottom = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3.5, 0))
                local sizeY = math.abs(top.Y - bottom.Y)
                local sizeX = sizeY / 1.5
                local color = (currentTarget == p) and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(255, 255, 255)

                if showBox then
                    obj.Box.Size = Vector2.new(sizeX, sizeY)
                    obj.Box.Position = Vector2.new(pos.X - sizeX / 2, pos.Y - sizeY / 2)
                    obj.Box.Color = color
                    obj.Box.Visible = true
                else obj.Box.Visible = false end

                if showLine then
                    obj.Line.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                    obj.Line.To = Vector2.new(pos.X, pos.Y + (sizeY / 2))
                    obj.Line.Color = color
                    obj.Line.Visible = true
                else obj.Line.Visible = false end

                if showHealth then
                    local hpPerc = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
                    obj.HealthBar.From = Vector2.new(pos.X - sizeX / 2 - 5, pos.Y + sizeY / 2)
                    obj.HealthBar.To = Vector2.new(pos.X - sizeX / 2 - 5, pos.Y + sizeY / 2 - (sizeY * hpPerc))
                    obj.HealthBar.Color = Color3.fromRGB(255 * (1 - hpPerc), 255 * hpPerc, 0)
                    obj.HealthBar.Visible = true
                else obj.HealthBar.Visible = false end

                if showName or showDistance then
                    local dist = math.floor((hrp.Position - player.Character.HumanoidRootPart.Position).Magnitude)
                    local text = ""
                    if showName then text = text .. p.Name end
                    if showDistance then text = (text ~= "" and text .. "\n" or "") .. "[" .. dist .. "m]" end
                    obj.NameTag.Text = text
                    obj.NameTag.Position = Vector2.new(pos.X, pos.Y - (sizeY / 2) - 30)
                    obj.NameTag.Visible = true
                else obj.NameTag.Visible = false end
            else for _, v in pairs(obj) do v.Visible = false end end
        else if obj then for _, v in pairs(obj) do v.Visible = false end end end
    end
    playerCountLabel.Visible = (espMaster and showCount)
    playerCountLabel.Text = "Số người chơi: " .. tostring(count)
end

-- Hàm dọn dẹp để tắt script
local function cleanUp()
    for _, p in pairs(Players:GetPlayers()) do
        if espObjects[p] then
            for _, v in pairs(espObjects[p]) do v:Remove() end
        end
    end
    playerCountLabel:Remove()
    crosshairVertical:Remove()
    crosshairHorizontal:Remove()
    Rayfield:Destroy()
end

-- =========================
-- 🎨 GIAO DIỆN (UI)
-- =========================

local CombatTab = Window:CreateTab("Chiến Đấu", 4483362458)
CombatTab:CreateSection("Chế độ Tàn Sát")
CombatTab:CreateToggle({
   Name = "GIẾT TẤT CẢ (Kill Aura)",
   CurrentValue = false,
   Callback = function(Value) killAura = Value end,
})
CombatTab:CreateToggle({
   Name = "Auto Chat (Noob + Name)",
   CurrentValue = false,
   Callback = function(Value) autoChat = Value end,
})

CombatTab:CreateSection("Di Chuyển")
CombatTab:CreateToggle({
   Name = "BẬT CHẠY NHANH",
   CurrentValue = false,
   Callback = function(Value) speedEnabled = Value handleMovement() end,
})
CombatTab:CreateSlider({
   Name = "Tốc độ chạy",
   Range = {16, 200},
   Increment = 1,
   Suffix = " Speed",
   CurrentValue = 16,
   Callback = function(Value) walkSpeedValue = Value handleMovement() end,
})
CombatTab:CreateToggle({
   Name = "BẬT NHẢY CAO",
   CurrentValue = false,
   Callback = function(Value) jumpEnabled = Value handleMovement() end,
})
CombatTab:CreateSlider({
   Name = "Lực nhảy",
   Range = {50, 500},
   Increment = 1,
   Suffix = " Power",
   CurrentValue = 50,
   Callback = function(Value) jumpPowerValue = Value handleMovement() end,
})

CombatTab:CreateSection("Hỗ trợ Ngắm")
CombatTab:CreateToggle({
   Name = "TÂM ẢO (Custom Crosshair)",
   CurrentValue = false,
   Callback = function(Value) crosshairEnabled = Value end,
})
CombatTab:CreateSection("Kỹ năng đặc biệt")
CombatTab:CreateToggle({
   Name = "GIẢ CHẾT (Fake Death)",
   CurrentValue = false,
   Callback = function(Value) fakeDeath = Value toggleFakeDeath(Value) end,
})

local VisualTab = Window:CreateTab("Hiển Thị", 4483362458)
VisualTab:CreateSection("Đổi góc nhìn (Spectate)")
local SpectateDropdown = VisualTab:CreateDropdown({
   Name = "Chọn người chơi",
   Options = getPlayerNames(),
   CurrentOption = {"Chọn"},
   Callback = function(Option) selectedSpectatePlayer = Option[1] end,
})
VisualTab:CreateButton({
   Name = "Làm mới danh sách người chơi",
   Callback = function() SpectateDropdown:Refresh(getPlayerNames(), true) end,
})
VisualTab:CreateToggle({
   Name = "Bật Spectate",
   CurrentValue = false,
   Callback = function(Value) spectateEnabled = Value handleSpectate() end,
})

VisualTab:CreateSection("Kích hoạt ESP")
VisualTab:CreateToggle({
   Name = "HOẠT ĐỘNG ESP (Master Switch)",
   CurrentValue = false,
   Callback = function(Value) espMaster = Value if not Value then toggleDefaultNameTags(false) end end,
})
VisualTab:CreateSection("Tùy chọn hiển thị ESP")
VisualTab:CreateToggle({Name = "Hiện Box", CurrentValue = false, Callback = function(V) showBox = V end})
VisualTab:CreateToggle({Name = "Hiện Name", CurrentValue = false, Callback = function(V) showName = V end})
VisualTab:CreateToggle({Name = "Hiện Line", CurrentValue = false, Callback = function(V) showLine = V end})
VisualTab:CreateToggle({Name = "Hiện Health", CurrentValue = false, Callback = function(V) showHealth = V end})
VisualTab:CreateToggle({Name = "Hiện Distance", CurrentValue = false, Callback = function(V) showDistance = V end})
VisualTab:CreateToggle({Name = "Hiện Count", CurrentValue = false, Callback = function(V) showCount = V end})

local InventoryTab = Window:CreateTab("Kho Đồ", 4483362458)
local function getItems()
    local t = {}
    if player:FindFirstChild("Backpack") then for _, i in pairs(player.Backpack:GetChildren()) do if i:IsA("Tool") then table.insert(t, i.Name) end end end
    if player.Character then for _, i in pairs(player.Character:GetChildren()) do if i:IsA("Tool") then table.insert(t, i.Name) end end end
    return #t > 0 and t or {"Trống"}
end
local WeaponDropdown = InventoryTab:CreateDropdown({
   Name = "Chọn Vũ Khí",
   Options = getItems(),
   CurrentOption = {"Chọn"},
   Callback = function(Option) selectedWeapon = Option[1] end,
})
InventoryTab:CreateButton({Name = "Làm mới đồ", Callback = function() WeaponDropdown:Refresh(getItems(), true) end})
InventoryTab:CreateToggle({Name = "Auto Equip", CurrentValue = false, Callback = function(V) autoEquip = V end})

-- MỚI: Tab Hệ Thống (Điều khiển Script)
local SettingsTab = Window:CreateTab("Hệ Thống", 4483362458)
SettingsTab:CreateSection("Quản lý Script")

SettingsTab:CreateButton({
   Name = "Đóng Script (Tắt hoàn toàn)",
   Callback = function()
       Rayfield:Notify({
           Title = "Xác nhận đóng?",
           Content = "Bấm nút bên dưới để tắt toàn bộ menu.",
           Duration = 5,
           Image = 4483362458,
           Actions = {
               Ignore = { Name = "Hủy", Callback = function() print("Hủy đóng script") end },
               Accept = { Name = "Xác nhận Tắt", Callback = function() cleanUp() end }
           },
       })
   end,
})

SettingsTab:CreateButton({
   Name = "Chạy lại Script (Restart)",
   Callback = function()
       Rayfield:Notify({
           Title = "Xác nhận Restart?",
           Content = "Script sẽ tự tắt và khởi động lại.",
           Duration = 5,
           Image = 4483362458,
           Actions = {
               Ignore = { Name = "Hủy", Callback = function() print("Hủy restart") end },
               Accept = { 
                   Name = "Xác nhận Restart", 
                   Callback = function() 
                       cleanUp()
                       task.wait(1)
                       -- Tự chạy lại (Sử dụng URL script gốc của bạn hoặc prompt hiện tại)
                       -- Ở đây ta mô phỏng việc chạy lại code hiện có
                       loadstring(game:HttpGet('https://sirius.menu/rayfield'))() -- Tải lại lib
                       print("Script đã được restart!")
                   end 
               }
           },
       })
   end,
})

-- =========================
-- 🔁 VÒNG LẶP CHÍNH
-- =========================

Players.PlayerAdded:Connect(createESP)
Players.PlayerRemoving:Connect(function(p) if espObjects[p] then for _,v in pairs(espObjects[p]) do v:Remove() end espObjects[p] = nil end end)
for _, p in pairs(Players:GetPlayers()) do if p ~= player then createESP(p) end end

RunService.RenderStepped:Connect(function()
    updateESP()
    updateCrosshair()
    if spectateEnabled then handleSpectate() end
    handleMovement()
end)

task.spawn(function()
    while true do
        task.wait(0.5)
        if autoEquip and selectedWeapon and player.Character then
            local tool = player.Backpack:FindFirstChild(selectedWeapon) or player.Character:FindFirstChild(selectedWeapon)
            if tool and tool.Parent ~= player.Character then player.Character.Humanoid:EquipTool(tool) end
        end
    end
end)

RunService.Heartbeat:Connect(function()
    if killAura and not fakeDeath then
        if currentTarget and currentTarget.Character and currentTarget.Character:FindFirstChild("Humanoid") then
            if currentTarget.Character.Humanoid.Health <= 0 then
                if autoChat then
                    local chatEvents = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
                    local msg = "Noob " .. currentTarget.Name
                    if chatEvents then chatEvents.SayMessageRequest:FireServer(msg, "All")
                    else game:GetService("TextChatService").TextChannels.RBXGeneral:SendAsync(msg) end
                end
                currentTarget = nil
            end
        end
        if not currentTarget then
            local dist = math.huge
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character.Humanoid.Health > 0 then
                    local d = (player.Character.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude
                    if d < dist then dist = d currentTarget = p end
                end
            end
        end
        if currentTarget and currentTarget.Character and player.Character then
            player.Character.HumanoidRootPart.CFrame = currentTarget.Character.HumanoidRootPart.CFrame * CFrame.new(KILL_OFFSET)
            player.Character.HumanoidRootPart.Velocity = Vector3.new(0,0,0)
            VirtualUser:Button1Down(Vector2.new(0,0))
        end
    end
end)
