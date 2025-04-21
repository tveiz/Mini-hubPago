--[[
    Combined Script: Key System + Rayfield Hub
    1. Initializes necessary services.
    2. Defines a local key database and HWID check.
    3. Creates a simple UI to prompt for a key.
    4. Validates the entered key against the database, expiry, and HWID.
    5. If the key is valid:
        a. Destroys the key UI.
        b. Loads and initializes the Rayfield Hub.
        c. Sends a Discord webhook log.
        d. Executes the rest of the Hub features.
    6. If the key is invalid or expired, kicks the player.
    
    **Change:** Revistar button is now mobile-only, larger, and draggable. PC Keybind removed.
]]

-- Core Services
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local RbxAnalytics = game:GetService("RbxAnalyticsService")
local UserInputService = game:GetService("UserInputService") -- Needed for Rayfield part too
local ReplicatedStorage = game:GetService("ReplicatedStorage") -- Needed for Rayfield part too
local RunService = game:GetService("RunService") -- Needed for Rayfield part too
local Workspace = game:GetService("Workspace") -- Needed for Rayfield part too

-- Player and HWID Info
local LocalPlayer = Players.LocalPlayer
local hwid = RbxAnalytics:GetClientId() -- Using RbxAnalyticsService for HWID

-- --- Key System Configuration ---
-- Simulação de banco de dados local (In a real scenario, fetch this from a server)
local KeyDatabase = {
    ["ABC123"] = {expire = "2025-12-31", hwid = nil}, -- Extended expiry for testing
    ["DEF456"] = {expire = "2025-12-31", hwid = nil},
    ["GHI789"] = {expire = "2024-01-01", hwid = nil}, -- Example of expired key
    ["USED123"] = {expire = "2025-12-31", hwid = "some_other_hwid"}, -- Example of used key
}
local webhookURL = "https://discord.com/api/webhooks/1351282679420817492/KpPuA0jULdUAkBxqdGel7Uv4yYOvs1HX2cvYxL_PY09_EwFkStfMEvfNVTCZoRzCHQMM" -- Webhook URL from Rayfield script

-- --- Utility Functions ---

-- Generate random names for UI elements to make detection harder
local function randomName(len)
    local charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    local name = ""
    for i = 1, len do
        local rand = math.random(1, #charset)
        name = name .. charset:sub(rand, rand)
    end
    return name
end

-- Get device type for logging
local function getDeviceType()
    if UserInputService.TouchEnabled then
        return "Celular/Tablet"
    elseif UserInputService.KeyboardEnabled then
        return "PC"
    else
        return "Desconhecido"
    end
end

-- Send Discord Webhook Log
local function sendWebhook(status, keyUsed)
    pcall(function() -- Wrap in pcall to prevent errors stopping script execution
        local statusMessage = status or "Execução Iniciada (Pós-Key)"
        local keyInfo = keyUsed and "\nKey Usada: " .. keyUsed or ""
        local data = {
            ["username"] = "LOGS execução",
            ["avatar_url"] = "https://i.imgur.com/CF7wYq5.png",
            ["content"] = statusMessage .. "\nNome: " .. LocalPlayer.Name .. "\nUserId: " .. LocalPlayer.UserId .. "\nHorário: " .. os.date("%d/%m/%Y %H:%M:%S") .. "\nDispositivo: " .. getDeviceType() .. keyInfo
        }
        local jsonData = HttpService:JSONEncode(data)
        local body = { Url = webhookURL, Body = jsonData, Method = "POST", Headers = { ["Content-Type"] = "application/json" } }

        -- Use appropriate HTTP request method based on executor
        if syn and syn.request then
            syn.request(body)
        elseif request then
            request(body)
        elseif http and http.request then
            http.request(body)
        else
            warn("Http Request function not found! Cannot send webhook.")
        end
    end)
end

-- Key Validation Function
local function CheckKey(key)
    local data = KeyDatabase[key]
    if not data then
        return false, "Key inválida."
    end

    -- Check Expiry
    local today = os.date("*t")
    local todayString = string.format("%04d-%02d-%02d", today.year, today.month, today.day)
    if data.expire and todayString > data.expire then
        return false, "Key expirada."
    end

    -- Check HWID
    if data.hwid == nil then
        -- First time use for this key, bind HWID
        data.hwid = hwid
        print("Key validated and HWID bound:", key, hwid) -- Log HWID binding
        -- In a real system, you would update this on your server here
        return true, "Key válida. Acesso liberado e vinculado."
    elseif data.hwid == hwid then
        -- HWID matches, allow access
        return true, "Key válida. Acesso liberado."
    else
        -- HWID mismatch
        return false, "Key usada em outro dispositivo."
    end
end


-- --- Main Hub Logic (To be executed after key validation) ---
local function LoadMainHub(validatedKey)
    print("Key accepted. Loading Main Hub...")
    sendWebhook("Hub Carregado com Sucesso", validatedKey) -- Send webhook log after successful validation

    local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

    local Window = Rayfield:CreateWindow({
        Name = "Mini Hub pago 😈",
        LoadingInfo = {
            Developer = "Th",
            Creator = "Th",
            Discord = "https://discord.gg/XAnFXPUR",
            LoadingTip = "Carregando...",
        },
        ConfigurationSaving = {
            Enabled = true,
            FolderName = "Mini Hub", -- Should be unique if saving configs
            FileName = "Config"
        }
    })

    local Tab = Window:CreateTab("Principal", 4483362458)
    local MiniHubTab = Window:CreateTab("Mini Hub Free", 4483362459)

    -- Aimbot (PC) Section
    local AimbotSection = Tab:CreateSection("Aimbot (PC)")
    local aimbotEnabled = false
    local aimatpart = nil
    local cam = Workspace.CurrentCamera
    local mouse = LocalPlayer:GetMouse()
    local aimbotConnection

    local function getfovxyz(p0, p1, deg)
        local x1, y1, z1 = p0:ToOrientation()
        local cf = CFrame.new(p0.p, p1.p)
        local x2, y2, z2 = cf:ToOrientation()
        if deg then return Vector3.new(math.deg(x1 - x2), math.deg(y1 - y2), math.deg(z1 - z2)) end
        return Vector3.new((x1 - x2), (y1 - y2), (z1 - z2))
    end

    local function checkfov(part)
        if not part or not part.Parent or not cam then return math.huge end
        local fov = getfovxyz(cam.CFrame, part.CFrame)
        return math.abs(fov.X) + math.abs(fov.Y) -- Simplified FOV check
    end

    local function aimat(part)
        if part and cam then
           cam.CFrame = CFrame.new(cam.CFrame.p, part.Position)
        end
    end

    local function findBestTargetPC()
       local bestTarget = nil
       local minAngle = math.rad(20) -- Max FOV angle

       for _, plr in pairs(Players:GetPlayers()) do
           if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head") and plr.Character:FindFirstChild("Humanoid") and plr.Character.Humanoid.Health > 0 then
               local head = plr.Character.Head
               local targetVector = (head.Position - cam.CFrame.Position).Unit
               local angle = math.acos(cam.CFrame.LookVector:Dot(targetVector)) -- Angle between camera look and target

               local screenPoint, onScreen = cam:WorldToScreenPoint(head.Position)
                if onScreen and angle < minAngle then -- Check if within angle and on screen
                    minAngle = angle
                    bestTarget = head
                end
           end
       end
       return bestTarget
    end

    local pcAimbotActive = false
    local pcAimbotMouseButtonDown = false

    mouse.Button2Down:Connect(function()
        if not pcAimbotActive then return end
        pcAimbotMouseButtonDown = true
        aimatpart = findBestTargetPC()
    end)

    mouse.Button2Up:Connect(function()
        if not pcAimbotActive then return end
        pcAimbotMouseButtonDown = false
        aimatpart = nil
    end)

    RunService.RenderStepped:Connect(function()
        if pcAimbotActive and pcAimbotMouseButtonDown and aimatpart then
            if aimatpart and aimatpart.Parent and aimatpart.Parent:FindFirstChild("Humanoid") and aimatpart.Parent.Humanoid.Health > 0 then
                 aimat(aimatpart)
            else
                aimatpart = nil -- Target died or became invalid
                pcAimbotMouseButtonDown = false -- Stop aiming if target is gone
            end
        end
    end)

    Tab:CreateToggle({
        Name = "Aimbot (PC - Hold RMB)",
        Description = "Ativa/Desativa o Aimbot. Segure o botão direito do mouse para mirar.",
        CurrentValue = false,
        Section = AimbotSection,
        Callback = function(Value)
            pcAimbotActive = Value
            if not Value then
                aimatpart = nil -- Reset target when toggling off
                pcAimbotMouseButtonDown = false
            end
            print("PC Aimbot:", Value)
        end
    })


    -- Aimbot (Mobile) Section
    local AimbotMobileSection = Tab:CreateSection("Aimbot (Mobile)")
    local MobileAimbotEnable = false
    local mobileAimbotConnection = nil
    local AimPartMobile = nil

    local function AimAtMobile(Part)
        if Part and cam then
           cam.CFrame = CFrame.new(cam.CFrame.Position, Part.Position)
        end
    end

     local function FindTargetMobile()
        local Character = LocalPlayer.Character
        if not Character or not Character:FindFirstChild("Humanoid") or Character.Humanoid.Health <= 0 then return nil end

        local ClosestTarget = nil
        local ClosestDist = math.huge

        for _, Player in pairs(Players:GetPlayers()) do
            if Player ~= LocalPlayer then
                local TargetCharacter = Player.Character
                if TargetCharacter and TargetCharacter:FindFirstChild("Humanoid") and TargetCharacter.Humanoid.Health > 0 and TargetCharacter:FindFirstChild("Head") then
                    local head = TargetCharacter.Head
                    local distance = (head.Position - cam.CFrame.Position).Magnitude
                    local screenPoint, onScreen = cam:WorldToScreenPoint(head.Position)

                    if onScreen and distance < ClosestDist then -- Prioritize closest on-screen target
                        ClosestTarget = head
                        ClosestDist = distance
                    end
                end
            end
        end
        return ClosestTarget
    end

    local function MobileAimbotLoop()
        if not MobileAimbotEnable then return end
        local Target = FindTargetMobile()
         AimPartMobile = Target -- Update target continuously
        if AimPartMobile then
            AimAtMobile(AimPartMobile)
        end
    end

    Tab:CreateToggle({
        Name = "Aimbot (Mobile)",
        Description = "Trava a mira no oponente mais próximo para dispositivos mobile (Contínuo)",
        CurrentValue = false,
        Section = AimbotMobileSection,
        Callback = function(Value)
            MobileAimbotEnable = Value
            print("Mobile Aimbot:", Value)
            if Value then
                if not mobileAimbotConnection then
                   mobileAimbotConnection = RunService.RenderStepped:Connect(MobileAimbotLoop)
                end
            else
                if mobileAimbotConnection then
                   mobileAimbotConnection:Disconnect()
                   mobileAimbotConnection = nil
                   AimPartMobile = nil -- Clear target when disabled
                end
            end
        end
    })


    -- Hitbox Section
    local HitboxSection = Tab:CreateSection("Hitbox (PC e Mobile)")
    local hitboxEnabled = false
    local originalSizes = {}
    local hitboxConnection = nil
    local hitboxSize = Vector3.new(5, 5, 5) -- Configurable size

    local function setHitbox(targetPlayer, size)
        if not targetPlayer or not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("Head") then return end
        local head = targetPlayer.Character.Head
        if not originalSizes[targetPlayer] then
            originalSizes[targetPlayer] = {
                Size = head.Size,
                Transparency = head.Transparency,
                Material = head.Material,
                Color = head.Color,
                CanCollide = head.CanCollide
            }
        end
        -- Apply visual hitbox changes
        head.Size = size
        head.Transparency = 0.7
        head.Material = Enum.Material.Neon
        head.Color = Color3.fromRGB(255, 0, 0)
        head.CanCollide = false -- Important for not blocking movement/bullets weirdly
    end

    local function resetHitbox(targetPlayer)
         if not targetPlayer or not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("Head") or not originalSizes[targetPlayer] then return end
         local head = targetPlayer.Character.Head
         local original = originalSizes[targetPlayer]
         head.Size = original.Size
         head.Transparency = original.Transparency
         head.Material = original.Material
         head.Color = original.Color
         head.CanCollide = original.CanCollide
         originalSizes[targetPlayer] = nil -- Remove entry after resetting
    end

     local function updateHitboxes()
         if not hitboxEnabled then return end
         local currentPlayers = {}
         for _, player in pairs(Players:GetPlayers()) do
             if player ~= LocalPlayer and player.Character then
                 setHitbox(player, hitboxSize)
                 currentPlayers[player] = true
             end
         end
         -- Reset hitbox for players who left or lost character
         for player, _ in pairs(originalSizes) do
             if not currentPlayers[player] then
                 resetHitbox(player) -- This might need refinement if character is temporarily nil
             end
         end
     end

     local function handleCharacter(player)
        if player ~= LocalPlayer and player.Character then
            local char = player.Character
            local humanoid = char:FindFirstChildOfClass("Humanoid")
             if humanoid then
                 -- Connect Died event to reset hitbox if player dies
                 local diedConn = humanoid.Died:Connect(function()
                     resetHitbox(player)
                 end)
                 -- Store connection to disconnect later if needed (optional)
             end
            if hitboxEnabled then
                setHitbox(player, hitboxSize)
            end
         else -- No character or local player
             resetHitbox(player)
         end
     end


    Tab:CreateToggle({
        Name = "Hitbox Expander",
        Description = "Aumenta o tamanho visual e de colisão (CanCollide=false) da cabeça dos outros jogadores.",
        CurrentValue = false,
        Section = HitboxSection,
        Callback = function(Value)
            hitboxEnabled = Value
            print("Hitbox Expander:", Value)
            if Value then
                 -- Apply to existing players
                for _, player in pairs(Players:GetPlayers()) do
                   handleCharacter(player)
                end
                -- Setup connections if not already done
                if not hitboxConnection then
                   hitboxConnection = RunService.RenderStepped:Connect(updateHitboxes) -- Continuously check
                    Players.PlayerAdded:Connect(function(player)
                        player.CharacterAdded:Connect(function() handleCharacter(player) end)
                        handleCharacter(player) -- Handle if character already exists on join
                    end)
                    Players.PlayerRemoving:Connect(resetHitbox)
                end
            else
                -- Disconnect loop and reset all
                if hitboxConnection then
                   hitboxConnection:Disconnect()
                   hitboxConnection = nil
                   -- Consider removing playeradded/removing connections too if added here
                end
                for player, _ in pairs(originalSizes) do
                    resetHitbox(player)
                end
                originalSizes = {} -- Clear the table
            end
        end
    })


    -- Mini Hub Free Tab Content
    local Paragraph = MiniHubTab:CreateParagraph({Title = "MINI HUB ON TOPPP", Content = "Feito por Th"})
    local Section = MiniHubTab:CreateSection("NECESSARIO")

    -- Utility: Deletar NotifyGui
    local function deletarNotifyGui()
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then
            for _, gui in ipairs(playerGui:GetChildren()) do
                if gui.Name == "NotifyGui" and gui:IsA("ScreenGui") then
                    gui:Destroy()
                    print("NotifyGui deleted.")
                end
            end
        end
    end

    -- Roubar Inventário Function
    local function roubarInventario()
        local itens = {"AK47", "Uzi", "PARAFAL", "Faca", "IA2", "G3", "Escudo", "C4", "AR-15", "Hi Power", "Natalina", "Tratamento", "Planta Limpa", "Planta Suja", "ArmaÃƒÂ§ÃƒÂ£o de Arma", "PeÃƒÂ§a de Arma", "Glock 17", "Skate"}
        local invRemote = ReplicatedStorage:FindFirstChild("Modules", 10) and ReplicatedStorage.Modules:FindFirstChild("InvRemotes", 10) and ReplicatedStorage.Modules.InvRemotes:FindFirstChild("InvRequest", 10)

        if not invRemote then
            warn("Could not find InvRequest remote. Roubar Inventário might not work.")
            Rayfield:Notify({ Title = "Erro", Content = "Remoto de inventário não encontrado.", Duration = 5,})
            return
        end

        local args = { [1] = "mudaInv", [2] = "PLACEHOLDER_INDEX", [3] = "PLACEHOLDER_ITEM", [4] = "1" }
        local running = true -- Control variable for the loop
        local stopButton = nil -- Declare here to access later

        -- Button to stop (optional but good practice for infinite loops)
        stopButton = MiniHubTab:CreateButton({
            Name = "Parar Roubar Inv",
            Callback = function()
                running = false
                print("Stopping Roubar Inventário...")
                if stopButton and stopButton.Parent then -- Check if button still exists
                    stopButton:Destroy() -- Remove the stop button itself
                end
            end,
        })

        task.spawn(function()
            while running do
                deletarNotifyGui()
                for i, item in ipairs(itens) do
                    if not running then break end -- Check flag inside inner loop too
                    if i <= 16 then -- Original condition
                        args[2] = tostring(i) -- Index
                        args[3] = item -- Item name
                        pcall(function() -- Wrap remote invoke in pcall
                            invRemote:InvokeServer(unpack(args))
                        end)
                        task.wait(0.05) -- Small delay between requests to avoid potential rate limits/errors
                    end
                end
                if not running then break end -- Check flag after outer loop cycle
                task.wait(0.1) -- Wait a bit before restarting the cycle
            end
            print("Roubar Inventário loop finished.")
             if stopButton and stopButton.Parent then stopButton:Destroy() end -- Clean up stop button if loop finished naturally or stopped
        end)

        Rayfield:Notify({ Title = "Info", Content = "Roubar Inventário iniciado.", Duration = 5,})
    end

    MiniHubTab:CreateButton({
        Name = "Roubar inv (Loop)",
        Description = "Tenta adicionar itens repetidamente. Use 'Parar Roubar Inv' para cancelar.",
        Callback = roubarInventario,
    })
    -- Revistar Section
    local RevistarSection = MiniHubTab:CreateSection("Revistar")
    
    -- Anti Revistar Section
    local AntiRevistarSection = MiniHubTab:CreateSection("Anti Revistar")
    local antiRevistarConnection = nil
    local antiRevistarEnabledInternal = false -- Internal flag for CharacterAdded connection
    local function antiRevistarCheck(health)
        if health <= 5 then
            LocalPlayer:Kick("ANTI SER REVISTADO ATIVADO")
        end
    end

    MiniHubTab:CreateToggle({
        Name = "Anti ser revistado",
        Description = "Desconecta você se sua vida ficar muito baixa (evita ser revistado após ser derrubado).",
        CurrentValue = false,
        Flag = "antirevistar_toggle", -- Unique flag
        Section = AntiRevistarSection,
        Callback = function(Value)
            antiRevistarEnabledInternal = Value -- Update internal flag first
            print("Anti Revistar:", Value)
            local character = LocalPlayer.Character
            local humanoid = character and character:FindFirstChild("Humanoid")

            if Value then
                 -- Attempt to connect if humanoid exists
                if humanoid and not antiRevistarConnection then
                   antiRevistarConnection = humanoid.HealthChanged:Connect(antiRevistarCheck)
                   print("AntiRevistar connected to current humanoid.")
                end
                -- Setup CharacterAdded connection regardless, it will check internal flag
                if not LocalPlayer:FindFirstChild("AntiRevistarCharAddedConn") then -- Avoid multiple CharacterAdded connects
                    local conn = LocalPlayer.CharacterAdded:Connect(function(char)
                       local hum = char:WaitForChild("Humanoid")
                        -- Only connect HealthChanged if the toggle is *still* enabled when the character loads
                        if antiRevistarEnabledInternal and not antiRevistarConnection then
                           antiRevistarConnection = hum.HealthChanged:Connect(antiRevistarCheck)
                           print("AntiRevistar connected to new humanoid.")
                        end
                   end)
                   -- Store connection reference on player object to manage it (optional but cleaner)
                   conn.Name = "AntiRevistarCharAddedConn"
                   conn.Parent = LocalPlayer
                end
            else
                -- Disconnect HealthChanged if it exists
                if antiRevistarConnection then
                   antiRevistarConnection:Disconnect()
                   antiRevistarConnection = nil
                   print("AntiRevistar disconnected.")
                end
                -- We leave the CharacterAdded connection active, as it checks the internal flag.
                -- Alternatively, disconnect it here too if you prefer.
                -- local charAddedConn = LocalPlayer:FindFirstChild("AntiRevistarCharAddedConn")
                -- if charAddedConn then charAddedConn:Destroy() end
            end
        end,
    })

    -- ESP Section
    local ESPSection = MiniHubTab:CreateSection("ESP")
    local ESPAtivo = false
    local espConnections = {} -- Store connections related to ESP

    local function removerESP(player)
        if not player then return end
        if player.Character and player.Character:FindFirstChild("Head") then
            local head = player.Character.Head
            local esp = head:FindFirstChild("ESP")
            if esp then
                esp:Destroy()
            end
        end
        -- Disconnect associated connections for this player
        if espConnections[player] then
            for _, conn in pairs(espConnections[player]) do
                if typeof(conn) == "RBXScriptConnection" then -- Ensure it's a connection before disconnecting
                    pcall(function() conn:Disconnect() end)
                end
            end
            espConnections[player] = nil -- Remove player entry
        end
    end

    local function criarESP(player)
         if not ESPAtivo or player == LocalPlayer or not player.Character or not player.Character:FindFirstChild("Head") then return end
         local head = player.Character.Head
         if head:FindFirstChild("ESP") then return end -- Already exists

         if not espConnections[player] then espConnections[player] = {} end

         local billboard = Instance.new("BillboardGui")
         billboard.Name = "ESP"
         billboard.Adornee = head
         billboard.AlwaysOnTop = true
         billboard.Size = UDim2.new(0, 150, 0, 40) -- Slightly smaller
         billboard.StudsOffset = Vector3.new(0, 2.5, 0) -- Adjust offset
         billboard.Enabled = true

         local nameLabel = Instance.new("TextLabel")
         nameLabel.Size = UDim2.new(1, 0, 0.5, 0)
         nameLabel.Position = UDim2.new(0, 0, 0, 0)
         nameLabel.BackgroundTransparency = 1
         nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
         nameLabel.TextStrokeTransparency = 0.3 -- Less stroke
         nameLabel.Font = Enum.Font.SourceSans -- Standard font
         nameLabel.TextScaled = true
         nameLabel.Text = player.Name
         nameLabel.Parent = billboard

         local healthLabel = Instance.new("TextLabel")
         healthLabel.Size = UDim2.new(1, 0, 0.5, 0)
         healthLabel.Position = UDim2.new(0, 0, 0.5, 0)
         healthLabel.BackgroundTransparency = 1
         healthLabel.TextColor3 = Color3.fromRGB(0, 255, 0) -- Green for health
         healthLabel.TextStrokeTransparency = 0.3
         healthLabel.Font = Enum.Font.SourceSans
         healthLabel.TextScaled = true
         healthLabel.Text = "Vida: ..."
         healthLabel.Parent = billboard

         billboard.Parent = head -- Parent last

         local humanoid = player.Character:FindFirstChild("Humanoid")
         if humanoid then
             local function atualizarVida()
                 if humanoid and humanoid.Health > 0 and billboard.Parent -- Check if humanoid and billboard still exist
                 then
                     local healthPercent = humanoid.Health / humanoid.MaxHealth
                     healthLabel.Text = "Vida: " .. math.floor(humanoid.Health)
                     -- Change color based on health
                     healthLabel.TextColor3 = Color3.fromHSV(0.33 * healthPercent, 1, 1) -- Green to Red gradient
                 else
                     -- If humanoid dies or billboard is removed, clean up
                     removerESP(player)
                 end
             end
             atualizarVida()
             table.insert(espConnections[player], humanoid.HealthChanged:Connect(atualizarVida))
             table.insert(espConnections[player], humanoid.Died:Connect(function() removerESP(player) end))
         else
              healthLabel.Text = "Vida: ?" -- No humanoid found
              task.wait(1) -- Wait a bit then remove if humanoid never appeared
              if not player.Character:FindFirstChild("Humanoid") then removerESP(player) end
         end

         -- Handle character removal event more reliably
         if not espConnections[player].CharacterRemoving then
             espConnections[player].CharacterRemoving = player.CharacterRemoving:Connect(function() removerESP(player) end)
         end
     end

     local function setupESPForAllPlayers()
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                 handleCharacterESP(player) -- Use a helper
             end
        end
     end

     local function handleCharacterESP(player)
         if player.Character then
             criarESP(player)
         else
             removerESP(player) -- Clean up if character is gone when checking
         end
         -- Setup CharacterAdded connection for this specific player if it doesn't exist
          if not espConnections[player] or not espConnections[player].CharacterAdded then
              if not espConnections[player] then espConnections[player] = {} end
              espConnections[player].CharacterAdded = player.CharacterAdded:Connect(function(char)
                  task.wait(0.1) -- Short delay for character to fully load
                  if ESPAtivo then -- Check flag again when character respawns/loads
                       criarESP(player)
                  end
              end)
          end
     end

    -- Global PlayerAdded/Removing listeners managed by the toggle
    local globalPlayerAddedConn = nil
    local globalPlayerRemovingConn = nil

    MiniHubTab:CreateToggle({
        Name = "ESP nome/vida",
        Description = "Mostra nome e vida dos outros jogadores.",
        CurrentValue = false,
        Flag = "esp_toggle", -- Unique flag
        Section = ESPSection,
        Callback = function(Value)
            ESPAtivo = Value
            print("ESP:", Value)
            if Value then
                setupESPForAllPlayers()
                 -- Setup PlayerAdded connection if not already active
                if not globalPlayerAddedConn or not globalPlayerAddedConn.Connected then
                    globalPlayerAddedConn = Players.PlayerAdded:Connect(function(player)
                       if player ~= LocalPlayer then handleCharacterESP(player) end
                    end)
                end
                 -- Setup PlayerRemoving connection if not already active
                if not globalPlayerRemovingConn or not globalPlayerRemovingConn.Connected then
                   globalPlayerRemovingConn = Players.PlayerRemoving:Connect(function(player)
                       removerESP(player) -- Cleans up connections and UI for the leaving player
                   end)
                end
            else
                -- Disable for all current players and disconnect specific player connections
                for player, data in pairs(espConnections) do
                    if type(player) == "userdata" and player:IsA("Player") then -- Check if key is a player object
                        removerESP(player)
                    end
                end
                -- Disconnect global PlayerAdded/Removing listeners
                 if globalPlayerAddedConn and globalPlayerAddedConn.Connected then globalPlayerAddedConn:Disconnect() end
                 if globalPlayerRemovingConn and globalPlayerRemovingConn.Connected then globalPlayerRemovingConn:Disconnect() end
                 espConnections = {} -- Clear the entire connection map
            end
        end,
    })

    
    -- Ver Itens Section
    local VerItensSection = MiniHubTab:CreateSection("Itens dos Jogadores")
    local verItensUIAtivo = false
    local verItensConexoes = {} -- Map: player -> { gui = billboardGui, connections = {conn1, ...}, CharacterAdded = conn }

    local function removerVerItensUI(player)
        if not player then return end
        if verItensConexoes[player] then
            if verItensConexoes[player].gui and verItensConexoes[player].gui.Parent then
                verItensConexoes[player].gui:Destroy()
            end
            if verItensConexoes[player].connections then
                for _, conn in ipairs(verItensConexoes[player].connections) do
                   if typeof(conn) == "RBXScriptConnection" then pcall(function() conn:Disconnect() end) end
                end
            end
            -- Don't disconnect CharacterAdded here, it's handled by the main toggle or PlayerRemoving
            verItensConexoes[player] = nil -- Clear player entry
        end
    end

    local function adicionarVerItensUI(player)
         if not verItensUIAtivo or player == LocalPlayer or not player.Character or not player.Character:FindFirstChild("Head") then return end
         local head = player.Character.Head
         -- Check if UI already exists via map OR by finding child (redundancy)
         if (verItensConexoes[player] and verItensConexoes[player].gui) or head:FindFirstChild("ItemUI") then return end

         local gui = Instance.new("BillboardGui")
         gui.Name = "ItemUI"
         gui.Adornee = head
         gui.Size = UDim2.new(0, 180, 0, 70) -- Slightly larger
         gui.StudsOffset = Vector3.new(0, 3.5, 0) -- Adjusted offset
         gui.AlwaysOnTop = true
         gui.Enabled = true

         local title = Instance.new("TextLabel")
         title.Size = UDim2.new(1, 0, 0.3, 0)
         title.Position = UDim2.new(0, 0, 0, 0)
         title.BackgroundTransparency = 1
         title.TextColor3 = Color3.new(1, 1, 1)
         title.TextStrokeTransparency = 0.5
         title.Font = Enum.Font.SourceSansSemibold
         title.TextScaled = true
         title.Text = player.Name .. "'s Items"
         title.Parent = gui

         local itensLabel = Instance.new("TextLabel")
         itensLabel.Size = UDim2.new(1, 0, 0.7, 0)
         itensLabel.Position = UDim2.new(0, 0, 0.3, 0)
         itensLabel.BackgroundTransparency = 1
         itensLabel.TextColor3 = Color3.new(0.8, 0.8, 0.8) -- Light gray
         itensLabel.TextStrokeTransparency = 0.7
         itensLabel.Font = Enum.Font.SourceSans
         itensLabel.TextScaled = false -- Don't scale, use TextWrapped
         itensLabel.TextSize = 12 -- Fixed size
         itensLabel.TextWrapped = true
         itensLabel.TextXAlignment = Enum.TextXAlignment.Left
         itensLabel.TextYAlignment = Enum.TextYAlignment.Top
         itensLabel.Text = "Loading..."
         itensLabel.Parent = gui

         -- Initialize entry in connection map
         if not verItensConexoes[player] then verItensConexoes[player] = {} end
         if not verItensConexoes[player].connections then verItensConexoes[player].connections = {} end
         verItensConexoes[player].gui = gui

         local function updateItemList()
             -- Check if player and GUI still exist before updating
             if not player or not player.Parent or not verItensConexoes[player] or not verItensConexoes[player].gui or not verItensConexoes[player].gui.Parent then
                 removerVerItensUI(player) -- Clean up if player left or UI was destroyed
                 return
             end
             local list = {}
             local backpack = player:FindFirstChild("Backpack")
             if backpack then
                 for _, item in ipairs(backpack:GetChildren()) do
                     if item:IsA("Tool") then table.insert(list, item.Name) end
                 end
             end
             if player.Character then
                 for _, item in ipairs(player.Character:GetChildren()) do
                     if item:IsA("Tool") then table.insert(list, item.Name) end
                 end
             end
             if #list == 0 then
                 itensLabel.Text = "(Nenhum item)"
             else
                 itensLabel.Text = table.concat(list, ", ")
             end
         end

         updateItemList() -- Initial update

         -- Connections to update list
         local backpack = player:WaitForChild("Backpack", 5)
         if backpack then
            table.insert(verItensConexoes[player].connections, backpack.ChildAdded:Connect(updateItemList))
            table.insert(verItensConexoes[player].connections, backpack.ChildRemoved:Connect(updateItemList))
         end
         if player.Character then
            table.insert(verItensConexoes[player].connections, player.Character.ChildAdded:Connect(function(child) if child:IsA("Tool") then updateItemList() end end))
            table.insert(verItensConexoes[player].connections, player.Character.ChildRemoved:Connect(function(child) if child:IsA("Tool") then updateItemList() end end))
         end
         -- Connection for character removal (more reliable than Died)
         if not verItensConexoes[player].CharacterRemoving then
            verItensConexoes[player].CharacterRemoving = player.CharacterRemoving:Connect(function() removerVerItensUI(player) end)
            table.insert(verItensConexoes[player].connections, verItensConexoes[player].CharacterRemoving) -- Track this connection too
         end

         gui.Parent = head -- Parent last
     end

     local function setupVerItensForAll()
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                 handleCharacterVerItens(player)
             end
        end
     end

     local function handleCharacterVerItens(player)
         if player.Character then
             adicionarVerItensUI(player)
         else
             removerVerItensUI(player) -- Clean up if no character on check
         end
         -- Set up CharacterAdded connection if it doesn't exist for this player
         if not verItensConexoes[player] or not verItensConexoes[player].CharacterAdded then
            if not verItensConexoes[player] then verItensConexoes[player] = {} end
            verItensConexoes[player].CharacterAdded = player.CharacterAdded:Connect(function(char)
                task.wait(0.1) -- Wait briefly
                if verItensUIAtivo then adicionarVerItensUI(player) end
            end)
            -- Track the CharacterAdded connection itself if needed (optional)
            -- table.insert(verItensConexoes[player].connections, verItensConexoes[player].CharacterAdded)
         end
     end

    -- Global PlayerAdded/Removing connections for VerItens
    local verItensPlayerAddedConn = nil
    local verItensPlayerRemovingConn = nil

    MiniHubTab:CreateToggle({
        Name = "Ver itens UI",
        Description = "Mostra os itens no inventário/mão de outros jogadores.",
        CurrentValue = false,
        Flag = "veritensui_toggle", -- Unique flag
        Section = VerItensSection,
        Callback = function(Value)
            verItensUIAtivo = Value
            print("Ver Itens UI:", Value)
            if Value then
                setupVerItensForAll()
                -- Setup PlayerAdded/Removing connections if not active
                if not verItensPlayerAddedConn or not verItensPlayerAddedConn.Connected then
                    verItensPlayerAddedConn = Players.PlayerAdded:Connect(function(player)
                       if player ~= LocalPlayer then handleCharacterVerItens(player) end
                    end)
                end
                if not verItensPlayerRemovingConn or not verItensPlayerRemovingConn.Connected then
                   verItensPlayerRemovingConn = Players.PlayerRemoving:Connect(function(player)
                       removerVerItensUI(player) -- Cleans up connections and UI
                   end)
                end
            else
                -- Disable for all
                for player, data in pairs(verItensConexoes) do
                    if type(player) == "userdata" and player:IsA("Player") then
                       removerVerItensUI(player)
                    end
                end
                 -- Disconnect PlayerAdded/Removing
                if verItensPlayerAddedConn and verItensPlayerAddedConn.Connected then verItensPlayerAddedConn:Disconnect() end
                if verItensPlayerRemovingConn and verItensPlayerRemovingConn.Connected then verItensPlayerRemovingConn:Disconnect() end
                 verItensConexoes = {} -- Clear the map
            end
        end,
    })

    print("Rayfield Hub Initialized.")

end -- End of LoadMainHub function


-- --- Key System UI Creation and Logic ---
pcall(function()
    -- Check if UI already exists to prevent duplicates on re-execution
    if CoreGui:FindFirstChild("KeySystemUI_MiniHub") then
        print("Key System UI already exists.")
        return
    end

    local guiName = "KeySystemUI_MiniHub" -- Use a fixed name for checking existence
    local ui = Instance.new("ScreenGui")
    ui.Name = guiName
    ui.ResetOnSpawn = false
    ui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ui.DisplayOrder = 999 -- Try to render on top
    ui.Parent = CoreGui -- Use CoreGui for exploit UI usually

    local frame = Instance.new("Frame")
    frame.Name = randomName(10)
    frame.Size = UDim2.new(0, 320, 0, 180) -- Slightly larger
    frame.Position = UDim2.new(0.5, -160, 0.5, -90)
    frame.BackgroundColor3 = Color3.fromRGB(35, 35, 35) -- Darker gray
    frame.BorderSizePixel = 1
    frame.BorderColor3 = Color3.fromRGB(80, 80, 80)
    frame.Active = true -- Allows input processing
    frame.Visible = true
    frame.Parent = ui

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Name = randomName(8)
    titleLabel.Size = UDim2.new(1, 0, 0, 30)
    titleLabel.Position = UDim2.new(0, 0, 0, 0)
    titleLabel.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    titleLabel.TextColor3 = Color3.new(1, 1, 1)
    titleLabel.Font = Enum.Font.SourceSansBold
    titleLabel.TextSize = 16
    titleLabel.Text = "Mini Hub - Verificação"
    titleLabel.Parent = frame

    local textbox = Instance.new("TextBox")
    textbox.Name = randomName(7)
    textbox.Size = UDim2.new(0.9, 0, 0, 40)
    textbox.Position = UDim2.new(0.05, 0, 0.25, 0)
    textbox.PlaceholderText = "Insira sua key aqui"
    textbox.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    textbox.TextColor3 = Color3.new(1, 1, 1)
    textbox.Font = Enum.Font.SourceSans
    textbox.TextSize = 14
    textbox.ClearTextOnFocus = false
    textbox.Parent = frame
    textbox.FocusLost:Connect(function(enterPressed) -- Check key on Enter press
        if enterPressed then
            -- Trigger button click logic here
            button.MouseButton1Click:Fire() -- Assuming 'button' is the verify button instance
        end
    end)


    local button = Instance.new("TextButton")
    button.Name = randomName(9)
    button.Size = UDim2.new(0.9, 0, 0, 35)
    button.Position = UDim2.new(0.05, 0, 0.55, 0) -- Position below textbox
    button.Text = "Verificar Key"
    button.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
    button.TextColor3 = Color3.new(1, 1, 1)
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 16
    button.Parent = frame

    local message = Instance.new("TextLabel")
    message.Name = randomName(6)
    message.Size = UDim2.new(0.9, 0, 0.2, 0)
    message.Position = UDim2.new(0.05, 0, 0.78, 0) -- Position below button
    message.BackgroundTransparency = 1
    message.TextColor3 = Color3.new(1, 1, 1)
    message.Font = Enum.Font.SourceSansItalic
    message.TextSize = 14
    message.TextWrapped = true
    message.Text = "" -- Initial message empty
    message.Parent = frame

    local isVerifying = false -- Prevent double clicks
    
    -- Button Action
    button.MouseButton1Click:Connect(function()
        if isVerifying then return end -- Prevent spamming check
        isVerifying = true
        button.Text = "Verificando..." -- Give visual feedback
        button.AutoButtonColor = false -- Disable color change during check
        button.BackgroundColor3 = Color3.fromRGB(100, 100, 100)

        local inputKey = textbox.Text
        message.Text = "Verificando key..."
        message.TextColor3 = Color3.fromRGB(255, 255, 0) -- Yellow for processing
        task.wait(0.5) -- Simulate check delay

        local success, msg = CheckKey(inputKey)

        if success then
            message.Text = msg
            message.TextColor3 = Color3.fromRGB(0, 255, 0) -- Green for success
            print("Key Check Successful:", msg)
            task.wait(1.5) -- Show success message briefly
            ui:Destroy() -- Remove the key UI

            -- Load the main Hub logic
            LoadMainHub(inputKey) -- Pass the validated key for potential use/logging

        else
            message.Text = msg
            message.TextColor3 = Color3.fromRGB(255, 80, 80) -- Red for failure
            print("Key Check Failed:", msg)
            sendWebhook("Falha na Key: " .. msg, inputKey) -- Log failed attempt
            task.wait(2) -- Show error message
             -- Don't kick immediately, let user see message and retry if needed
             -- LocalPlayer:Kick("Falha na verificação: " .. msg)
            isVerifying = false -- Allow retry
            button.Text = "Verificar Key"
            button.AutoButtonColor = true
            button.BackgroundColor3 = Color3.fromRGB(70, 70, 70) -- Reset color
            textbox:CaptureFocus() -- Refocus textbox for easy retry
        end
        -- Note: isVerifying only reset on failure, successful load removes the button anyway
    end)

    print("Key System UI Loaded. Waiting for input.")
    textbox:CaptureFocus() -- Focus the textbox automatically

end)
print("Mini Hub Script Initialized - Waiting for Key Verification")
