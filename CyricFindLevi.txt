--[[
    LEVIATHAN HUB - UPDATED VERSION
    Library: Fluent UI
    Status: Optimized & Movement Logic Only
]]

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/InterfaceManager.lua"))()

--------------------------------------------------------------------------------
-- 1. TẠO NÚT BẬT TẮT MENU (MINI TOGGLE BUTTON)
--------------------------------------------------------------------------------
local ToggleGui = Instance.new("ScreenGui")
local ToggleBtn = Instance.new("TextButton")
local ToggleCorner = Instance.new("UICorner")

ToggleGui.Name = "LeviathanToggle"
ToggleGui.Parent = game.CoreGui

ToggleBtn.Name = "ToggleBtn"
ToggleBtn.Parent = ToggleGui
ToggleBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
ToggleBtn.Position = UDim2.new(0.1, 0, 0.1, 0)
ToggleBtn.Size = UDim2.new(0, 50, 0, 50)
ToggleBtn.Font = Enum.Font.GothamBold
ToggleBtn.Text = "ON"
ToggleBtn.TextColor3 = Color3.fromRGB(0, 255, 127) -- Màu xanh lá
ToggleBtn.TextSize = 16
ToggleBtn.Draggable = true
ToggleBtn.AutoButtonColor = true

ToggleCorner.CornerRadius = UDim.new(0, 12)
ToggleCorner.Parent = ToggleBtn

local UI_Visible = true

ToggleBtn.MouseButton1Click:Connect(function()
    UI_Visible = not UI_Visible
    Fluent:ToggleWindow() -- Hàm ẩn hiện của Fluent
    
    if UI_Visible then
        ToggleBtn.Text = "ON"
        ToggleBtn.TextColor3 = Color3.fromRGB(0, 255, 127)
    else
        ToggleBtn.Text = "OFF"
        ToggleBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
    end
end)

--------------------------------------------------------------------------------
-- 2. CẤU HÌNH CỬA SỔ CHÍNH
--------------------------------------------------------------------------------
local Window = Fluent:CreateWindow({
    Title = "Leviathan Hub | Updated",
    SubTitle = "Helper Tool",
    TabWidth = 160,
    Size = UDim2.fromOffset(580, 460),
    Acrylic = true, 
    Theme = "Dark", -- Mặc định đen, vào Setting để chỉnh sang Light (Trắng)
    MinimizeKey = Enum.KeyCode.RightControl
})

local Tabs = {
    Status = Window:AddTab({ Title = "Trạng Thái (Status)", Icon = "activity" }),
    Attack = Window:AddTab({ Title = "Săn Leviathan", Icon = "sword" }),
    Teleport = Window:AddTab({ Title = "Di Chuyển & Spy", Icon = "map" }),
    Settings = Window:AddTab({ Title = "Cài Đặt", Icon = "settings" })
}

local Options = Fluent.Options
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

-- Hàm Tween Di chuyển nhân vật (Bỏ qua va chạm)
local function TweenTo(targetCFrame)
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local Root = LocalPlayer.Character.HumanoidRootPart
        local Distance = (Root.Position - targetCFrame.Position).Magnitude
        local Speed = 300 -- Tốc độ bay
        local Time = Distance / Speed
        
        local TI = TweenInfo.new(Time, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
        local Tween = TweenService:Create(Root, TI, {CFrame = targetCFrame})
        Tween:Play()
        
        -- Giữ cho nhân vật không bị rơi
        if not Root:FindFirstChild("AntiGravity") then
            local BV = Instance.new("BodyVelocity")
            BV.Name = "AntiGravity"
            BV.Parent = Root
            BV.Velocity = Vector3.new(0,0,0)
            BV.MaxForce = Vector3.new(9e9, 9e9, 9e9)
        end
    end
end

local function StopTween()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local Root = LocalPlayer.Character.HumanoidRootPart
        if Root:FindFirstChild("AntiGravity") then
            Root.AntiGravity:Destroy()
        end
        -- Hủy tween hiện tại bằng cách tạo tween nhẹ tại chỗ
        TweenService:Create(Root, TweenInfo.new(0.1), {CFrame = Root.CFrame}):Play()
    end
end

--------------------------------------------------------------------------------
-- TAB 1: STATUS
--------------------------------------------------------------------------------
local StatusInfo = Tabs.Status:AddParagraph({
    Title = "Trạng thái Leviathan",
    Content = "Đang kiểm tra dữ liệu..."
})

-- Vòng lặp kiểm tra (Logic an toàn: Kiểm tra Item trong túi)
task.spawn(function()
    while true do
        task.wait(1)
        local hasChip = false
        -- Kiểm tra trong Backpack hoặc Character
        if LocalPlayer.Backpack:FindFirstChild("Leviathan Scale") or LocalPlayer.Backpack:FindFirstChild("Microchip") then
             -- Bạn cần đổi tên chính xác item "Chip" trong game vào đây
            hasChip = true
        end

        if hasChip then
            StatusInfo:SetDesc("✅ You can find Leviathan now!")
        else
            StatusInfo:SetDesc("❌ Buy Leviathan Chip please")
        end
        
        -- Kiểm tra xem Leviathan có spawn chưa (Check workspace)
        if workspace.Enemies:FindFirstChild("Leviathan") then
             StatusInfo:SetTitle("⚠️ LEVIATHAN ĐÃ XUẤT HIỆN! ⚠️")
        else
             StatusInfo:SetTitle("Trạng thái Leviathan: Chưa thấy")
        end
    end
end)

--------------------------------------------------------------------------------
-- TAB 2: ATTACK & MOVEMENT LEVIATHAN
--------------------------------------------------------------------------------

-- Chọn vũ khí (Chỉ Auto Equip)
local WeaponDropdown = Tabs.Attack:AddDropdown("SelectWeapon", {
    Title = "Chọn Vũ Khí để cầm",
    Values = {"Melee", "Sword", "Gun", "Blox Fruit"},
    Multi = false,
    Default = 1,
})

Tabs.Attack:AddToggle("AutoFlyLevi", {Title = "Auto Bay theo Leviathan (Safe)", Default = false })

-- Logic tìm Leviathan và bay theo
task.spawn(function()
    while true do
        task.wait()
        if Options.AutoFlyLevi.Value then
            local Leviathan = workspace.Enemies:FindFirstChild("Leviathan") -- Tên Folder chứa Levi
            local Char = LocalPlayer.Character
            
            if Leviathan and Char and Char:FindFirstChild("Humanoid") then
                local Hum = Char.Humanoid
                local Root = Char.HumanoidRootPart
                
                -- 1. Logic Hồi máu
                if Hum.Health < (Hum.MaxHealth * 0.3) then -- Dưới 30% máu
                    -- Bay lên cao an toàn
                    TweenTo(Root.CFrame * CFrame.new(0, 500, 0))
                    StatusInfo:SetDesc("Đang hồi máu trên cao...")
                    repeat task.wait(1) until Hum.Health > (Hum.MaxHealth * 0.7) or not Options.AutoFlyLevi.Value
                else
                    -- 2. Logic Bay theo các bộ phận (Thân -> Đầu)
                    local TargetPart = nil
                    
                    -- Ưu tiên tìm Segments (Thân) trước
                    for _, part in pairs(Leviathan:GetChildren()) do
                        if part.Name == "Segment" and part:FindFirstChild("Humanoid") and part.Humanoid.Health > 0 then
                            TargetPart = part
                            break
                        end
                    end
                    
                    -- Nếu hết thân, tìm đầu (Head)
                    if not TargetPart and Leviathan:FindFirstChild("Head") then
                        if Leviathan.Head:FindFirstChild("Humanoid") and Leviathan.Head.Humanoid.Health > 0 then
                            TargetPart = Leviathan.Head
                        end
                    end
                    
                    if TargetPart then
                        -- Bay tới vị trí quái (Cách 15 studs để không bị đụng)
                        TweenTo(TargetPart.CFrame * CFrame.new(0, 15, 0))
                        
                        -- Auto Equip Vũ khí
                        local toolType = Options.SelectWeapon.Value
                        for _, tool in pairs(LocalPlayer.Backpack:GetChildren()) do
                            if tool:IsA("Tool") and tool.ToolTip == toolType then
                                tool.Parent = Char -- Equip
                            end
                        end
                        -- Ở đây bạn tự đánh tay hoặc dùng auto click, script chỉ hỗ trợ bay
                    else
                         StatusInfo:SetDesc("Đang tìm Leviathan...")
                    end
                end
            else
                StopTween()
            end
        else
            StopTween()
        end
    end
end)

--------------------------------------------------------------------------------
-- TAB 3: TELEPORT & SPY
--------------------------------------------------------------------------------

-- Teleport đến NPC Spy
Tabs.Teleport:AddButton({
    Title = "Teleport to Spy NPC",
    Description = "Bay đến NPC để mua thông tin",
    Callback = function()
        -- Tọa độ giả định của Spy (Bạn cần lấy CFrame chính xác thay vào đây)
        -- Ví dụ: TweenTo(CFrame.new(1000, 50, 2000))
        print("Vui lòng nhập tọa độ Spy vào script!")
        Fluent:Notify({Title = "Thông báo", Content = "Cần điền tọa độ NPC Spy trong Code", Duration = 3})
    end
})

-- Tween Player
local TargetNameInput = Tabs.Teleport:AddInput("TargetName", {
    Title = "Tên người chơi",
    Placeholder = "Nhập 1 phần tên...",
})

Tabs.Teleport:AddToggle("TweenPlayer", {Title = "Bay theo người chơi", Default = false })

task.spawn(function()
    while true do
        task.wait()
        if Options.TweenPlayer.Value and Options.TargetName.Value ~= "" then
            for _, v in pairs(Players:GetPlayers()) do
                if v.Name:lower():find(Options.TargetName.Value:lower()) or v.DisplayName:lower():find(Options.TargetName.Value:lower()) then
                    if v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
                         TweenTo(v.Character.HumanoidRootPart.CFrame * CFrame.new(0, 5, 3))
                    end
                end
            end
        end
    end
end)

--------------------------------------------------------------------------------
-- TAB 4: SETTINGS (FIX LAG & THEME)
--------------------------------------------------------------------------------

Tabs.Settings:AddButton({
    Title = "Fix Lag Nhẹ (Boost FPS)",
    Description = "Xóa hiệu ứng nước, chi tiết map",
    Callback = function()
        -- Tắt đổ bóng
        game.Lighting.GlobalShadows = false
        game.Lighting.FogEnd = 9e9
        
        -- Giảm đồ họa nước
        local Terrain = workspace:FindFirstChildOfClass("Terrain")
        if Terrain then
            Terrain.WaterWaveSize = 0
            Terrain.WaterWaveSpeed = 0
            Terrain.WaterReflectance = 0
            Terrain.WaterTransparency = 1 -- Làm nước trong suốt hoặc mờ
        end
        
        -- Xóa Particle (Hạt hiệu ứng)
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Smoke") or v:IsA("Fire") then
                v:Destroy()
            end
        end
        
        Fluent:Notify({Title = "Success", Content = "Đã giảm đồ họa game!", Duration = 3})
    end
})

-- Hop Server
Tabs.Settings:AddButton({
    Title = "Hop Server (Đổi Server)",
    Callback = function()
        local Http = game:GetService("HttpService")
        local TPS = game:GetService("TeleportService")
        local PlaceId = game.PlaceId
        local Servers = Http:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
        
        for _, s in pairs(Servers.data) do
            if s.playing < s.maxPlayers and s.id ~= game.JobId then
                TPS:TeleportToPlaceInstance(PlaceId, s.id, LocalPlayer)
                break
            end
        end
    end
})

-- Hệ thống quản lý Theme (Màu sắc) của Fluent
InterfaceManager:SetLibrary(Fluent)
SaveManager:SetLibrary(Fluent)
SaveManager:IgnoreThemeSettings() 
SaveManager:SetFolder("LeviathanHub_Config")
InterfaceManager:BuildInterfaceSection(Tabs.Settings) -- Tạo menu chọn màu tại đây

Window:SelectTab(1)

Fluent:Notify({
    Title = "Leviathan Hub",
    Content = "Script Loaded! Use Right Control to Hide.",
    Duration = 5
})
