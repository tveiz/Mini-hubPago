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

    -- ===========================
    -- == PRINCIPAL TAB SECTION ==
    -- ===========================
    local Tab = Window:CreateTab("Principal", 4483362458)

    -- Aimbot (PC) Section
    local AimbotSection = Tab:CreateSection("Aimbot (PC)")
    -- ... (Aimbot PC code remains the same) ...
    Tab:CreateToggle({
        Name = "Aimbot (PC - Hold RMB)",
        Description = "Ativa/Desativa o Aimbot. Segure o botão direito do mouse para mirar.",
        CurrentValue = false,
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
    -- ... (Aimbot Mobile code remains the same) ...
    Tab:CreateToggle({
        Name = "Aimbot (Mobile)",
        Description = "Trava a mira no oponente mais próximo para dispositivos mobile (Contínuo)",
        CurrentValue = false,
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
    -- ... (Hitbox code remains the same) ...
    Tab:CreateToggle({
        Name = "Hitbox Expander",
        Description = "Aumenta o tamanho visual e de colisão (CanCollide=false) da cabeça dos outros jogadores.",
        CurrentValue = false,
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

    -- ==============================
    -- == MINI HUB FREE TAB SECTION ==
    -- ==============================
    local MiniHubTab = Window:CreateTab("Mini Hub Free", 4483362459)

    -- Mini Hub Free Tab Content
    local Paragraph = MiniHubTab:CreateParagraph({Title = "MINI HUB ON TOPPP", Content = "Feito por Th"})
    local Section = MiniHubTab:CreateSection("NECESSARIO")
    -- ... (Roubar Inv, Revistar, Anti-Revistar, ESP, Ver Itens code remains the same) ...
    MiniHubTab:CreateButton({
        Name = "Roubar inv (Loop)",
        Description = "Tenta adicionar itens repetidamente. Use 'Parar Roubar Inv' para cancelar.",
        Callback = roubarInventario,
    })

    local RevistarSection = MiniHubTab:CreateSection("Revistar")
    -- ... (Revistar code) ...
    if UserInputService.TouchEnabled then
        MiniHubTab:CreateButton({
            Name = "Abrir Botão Revistar (Mobile)",
            Description = "Cria um botão grande e arrastável na tela para mandar revistar.",
            Callback = criarRevistarUI,
        })
    end
    MiniHubTab:CreateToggle({
        Name = "Ativar Auto Revistar",
        Description = "Envia o comando 'revistar' a cada 2 segundos.",
        CurrentValue = false,
        Flag = "autorsv_toggle",
        Callback = function(Value)
            autoRevistarAtivo = Value
            print("Auto Revistar:", Value)
            if autoRevistarAtivo then
                if autoRevTask then task.cancel(autoRevTask) end
                autoRevTask = task.spawn(function()
                    while autoRevistarAtivo do
                        sendRevistarCommand()
                        task.wait(2)
                    end
                    print("Auto Revistar task ended.")
                end)
            else
                if autoRevTask then
                    task.cancel(autoRevTask)
                    autoRevTask = nil
                end
            end
        end,
    })

    local AntiRevistarSection = MiniHubTab:CreateSection("Anti Revistar")
    -- ... (Anti-Revistar code) ...
    MiniHubTab:CreateToggle({
        Name = "Anti ser revistado",
        Description = "Desconecta você se sua vida ficar muito baixa (evita ser revistado após ser derrubado).",
        CurrentValue = false,
        Flag = "antirevistar_toggle",
        Callback = function(Value)
            antiRevistarEnabledInternal = Value
            print("Anti Revistar:", Value)
            local character = LocalPlayer.Character
            local humanoid = character and character:FindFirstChild("Humanoid")
            if Value then
                if humanoid and not antiRevistarConnection then
                   antiRevistarConnection = humanoid.HealthChanged:Connect(antiRevistarCheck)
                   print("AntiRevistar connected to current humanoid.")
                end
                if not LocalPlayer:FindFirstChild("AntiRevistarCharAddedConn") then
                    local conn = LocalPlayer.CharacterAdded:Connect(function(char)
                       local hum = char:WaitForChild("Humanoid")
                        if antiRevistarEnabledInternal and not antiRevistarConnection then
                           antiRevistarConnection = hum.HealthChanged:Connect(antiRevistarCheck)
                           print("AntiRevistar connected to new humanoid.")
                        end
                   end)
                   conn.Name = "AntiRevistarCharAddedConn"
                   conn.Parent = LocalPlayer
                end
            else
                if antiRevistarConnection then
                   antiRevistarConnection:Disconnect()
                   antiRevistarConnection = nil
                   print("AntiRevistar disconnected.")
                end
            end
        end,
    })

    local ESPSection = MiniHubTab:CreateSection("ESP")
    -- ... (ESP code) ...
    MiniHubTab:CreateToggle({
        Name = "ESP nome/vida",
        Description = "Mostra nome e vida dos outros jogadores.",
        CurrentValue = false,
        Flag = "esp_toggle",
        Callback = function(Value)
            ESPAtivo = Value
            print("ESP:", Value)
            if Value then
                setupESPForAllPlayers()
                if not globalPlayerAddedConn or not globalPlayerAddedConn.Connected then
                    globalPlayerAddedConn = Players.PlayerAdded:Connect(function(player)
                       if player ~= LocalPlayer then handleCharacterESP(player) end
                    end)
                end
                if not globalPlayerRemovingConn or not globalPlayerRemovingConn.Connected then
                   globalPlayerRemovingConn = Players.PlayerRemoving:Connect(function(player)
                       removerESP(player)
                   end)
                end
            else
                for player, data in pairs(espConnections) do
                    if type(player) == "userdata" and player:IsA("Player") then
                        removerESP(player)
                    end
                end
                 if globalPlayerAddedConn and globalPlayerAddedConn.Connected then globalPlayerAddedConn:Disconnect() end
                 if globalPlayerRemovingConn and globalPlayerRemovingConn.Connected then globalPlayerRemovingConn:Disconnect() end
                 espConnections = {}
            end
        end,
    })

    local VerItensSection = MiniHubTab:CreateSection("Itens dos Jogadores")
    -- ... (Ver Itens code) ...
    MiniHubTab:CreateToggle({
        Name = "Ver itens UI",
        Description = "Mostra os itens no inventário/mão de outros jogadores.",
        CurrentValue = false,
        Flag = "veritensui_toggle",
        Callback = function(Value)
            verItensUIAtivo = Value
            print("Ver Itens UI:", Value)
            if Value then
                setupVerItensForAll()
                if not verItensPlayerAddedConn or not verItensPlayerAddedConn.Connected then
                    verItensPlayerAddedConn = Players.PlayerAdded:Connect(function(player)
                       if player ~= LocalPlayer then handleCharacterVerItens(player) end
                    end)
                end
                if not verItensPlayerRemovingConn or not verItensPlayerRemovingConn.Connected then
                   verItensPlayerRemovingConn = Players.PlayerRemoving:Connect(function(player)
                       removerVerItensUI(player)
                   end)
                end
            else
                for player, data in pairs(verItensConexoes) do
                    if type(player) == "userdata" and player:IsA("Player") then
                       removerVerItensUI(player)
                    end
                end
                if verItensPlayerAddedConn and verItensPlayerAddedConn.Connected then verItensPlayerAddedConn:Disconnect() end
                if verItensPlayerRemovingConn and verItensPlayerRemovingConn.Connected then verItensPlayerRemovingConn:Disconnect() end
                 verItensConexoes = {}
            end
        end,
    })


    -- ===========================
    -- == TELEPORTOS TAB SECTION ==
    -- ===========================
    local TeleportTab = Window:CreateTab("Teleportos")
    TeleportTab:CreateParagraph({Title = "Mini Hub", Content = "por th"})
    TeleportTab:CreateSection("TELEPORTS FIXOS")

    -- Define the helper function within this scope so it can use TeleportTab
    local function addTeleportButton(nome, cframe)
        TeleportTab:CreateButton({
            Name = nome,
            Callback = function()
                local player = game.Players.LocalPlayer
                if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                    player.Character:SetPrimaryPartCFrame(cframe * CFrame.new(0, 3, 0))
                    Rayfield:Notify({ Title = "Teleporte", Content = "Teleportado para " .. nome, Duration = 3 })
                else
                    Rayfield:Notify({ Title = "Erro", Content = "Personagem não encontrado para teleportar.", Duration = 5 })
                end
            end
        })
    end

    -- Locais fixos
    addTeleportButton("Teleport Praça", CFrame.new(-291.579559, 3.26299787, 342.192535))
    addTeleportButton("Teleport Gás", CFrame.new(-469.959015, 3.25349784, -54.3936005))
    addTeleportButton("Teletransportar HP", CFrame.new(-543.439941, 3.26299858, 645.16864))
    addTeleportButton("Teleport Tabacaria", CFrame.new(-83.1141129, 13.1430578, 74.7073364))
    addTeleportButton("Teleport Garagem", CFrame.new(-466.870148, 7.64567232, 350.242737))
    addTeleportButton("Teleport Concessionária", CFrame.new(-91.3902893, 8.07136822, 520.355347))
    addTeleportButton("Teletransportar Gari", CFrame.new(-518.672852, 3.16749811, -1.16962147))
    addTeleportButton("Teleport Imobiliária", CFrame.new(-284.904785, 8.26088619, -72.2896194))
    addTeleportButton("Teletransportar PM", CFrame.new(-980.181458, 2.27553082, 467.080536))
    addTeleportButton("Teletransporte PRF", CFrame.new(6662.24512, 36.6637421, 5047.83838))
    addTeleportButton("Teleport Mineração", CFrame.new(201.932144, 2.76136589, 145.50531))
    addTeleportButton("Teleport Mecânica", CFrame.new(-180.608261, 3.29813337, -532.4151))
    addTeleportButton("Teleport Fazenda", CFrame.new(817.243225, 3.26249814, -87.316864))
    addTeleportButton("Teleport Prefeitura", CFrame.new(-284.388458, 15.1148872, 88.0397873))
    addTeleportButton("Teleport Banco", CFrame.new(-27.2709007, 11.5685892, 418.200653))
    addTeleportButton("Teletransporte ilegal", CFrame.new(12037.2705, 27.5305443, 12794.0635))
    addTeleportButton("Teleportar Prédio 1", CFrame.new(-1595.23328, 204.074341, 555.895386))
    addTeleportButton("Teletransporte Devs Mini", CFrame.new(-291.579559, 3.26299787, 342.192535))


    -- ==============================
    -- == TELEPORT PLAYERS SECTION ==
    -- ==============================
    TeleportTab:CreateSection("TELEPORTAR PARA JOGADOR")

    local playerListUI = nil -- Variable to hold the ScreenGui instance
    local playerListConnections = {} -- Table to hold PlayerAdded/Removed connections

    local function updatePlayerList()
        if not playerListUI or not playerListUI.Parent then return end -- Check if UI exists

        local scrollFrame = playerListUI:FindFirstChild("MainFrame"):FindFirstChild("ScrollFrame")
        if not scrollFrame then return end

        -- Clear existing buttons
        for _, v in pairs(scrollFrame:GetChildren()) do
            if v:IsA("TextButton") then
                v:Destroy()
            end
        end

        -- Add button for each player (except local)
        for _, targetPlayer in pairs(Players:GetPlayers()) do
            if targetPlayer ~= LocalPlayer then
                local button = Instance.new("TextButton")
                button.Name = targetPlayer.Name -- Set name for easy reference
                button.Text = targetPlayer.Name
                button.TextColor3 = Color3.fromRGB(255, 255, 255)
                button.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
                button.BackgroundTransparency = 0.2
                button.BorderSizePixel = 1
                button.BorderColor3 = Color3.fromRGB(100, 100, 100)
                button.Size = UDim2.new(1, -10, 0, 35) -- Full width minus padding, fixed height
                button.Font = Enum.Font.SourceSansSemibold
                button.TextSize = 16
                button.Parent = scrollFrame

                button.MouseButton1Click:Connect(function()
                    if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
                        local targetRoot = targetPlayer.Character.HumanoidRootPart
                        local player = game.Players.LocalPlayer
                        if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                            player.Character:SetPrimaryPartCFrame(targetRoot.CFrame * CFrame.new(0, 5, 0)) -- TP slightly above target
                            Rayfield:Notify({ Title = "Teleporte", Content = "Teleportado para " .. targetPlayer.Name, Duration = 3 })
                        else
                            Rayfield:Notify({ Title = "Erro", Content = "Seu personagem não foi encontrado.", Duration = 5 })
                        end
                    else
                        Rayfield:Notify({ Title = "Erro", Content = "Jogador '" .. targetPlayer.Name .. "' ou seu personagem não encontrado.", Duration = 5 })
                         -- Optionally refresh list if player left unexpectedly
                         task.wait(0.1)
                         updatePlayerList()
                    end
                end)
            end
        end
    end

    local function createPlayerListUI()
        if playerListUI and playerListUI.Parent then return playerListUI end -- Return existing if valid

        local screenGui = Instance.new("ScreenGui")
        screenGui.Name = "PlayerListUI"
        screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        screenGui.ResetOnSpawn = false

        local mainFrame = Instance.new("Frame")
        mainFrame.Name = "MainFrame"
        mainFrame.Size = UDim2.new(0, 250, 0, 350) -- Decent size for list
        mainFrame.Position = UDim2.new(0.5, -125, 0.5, -175) -- Centered
        mainFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        mainFrame.BorderSizePixel = 2
        mainFrame.BorderColor3 = Color3.fromRGB(80, 80, 80)
        mainFrame.Active = true
        mainFrame.Draggable = true -- Allow dragging
        mainFrame.Parent = screenGui

        local titleLabel = Instance.new("TextLabel")
        titleLabel.Name = "Title"
        titleLabel.Size = UDim2.new(1, 0, 0, 30)
        titleLabel.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
        titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        titleLabel.Font = Enum.Font.SourceSansBold
        titleLabel.TextSize = 18
        titleLabel.Text = "Selecionar Jogador"
        titleLabel.Parent = mainFrame

        local scrollFrame = Instance.new("ScrollingFrame")
        scrollFrame.Name = "ScrollFrame"
        scrollFrame.Size = UDim2.new(1, 0, 1, -30) -- Fill frame below title
        scrollFrame.Position = UDim2.new(0, 0, 0, 30)
        scrollFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
        scrollFrame.BorderSizePixel = 0
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0) -- Auto-adjusts with UIListLayout
        scrollFrame.ScrollBarThickness = 6
        scrollFrame.VerticalScrollBarInset = Enum.ScrollBarInset.ScrollBar
        scrollFrame.Parent = mainFrame

        local listLayout = Instance.new("UIListLayout")
        listLayout.Padding = UDim.new(0, 5) -- Space between buttons
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
        listLayout.Parent = scrollFrame

        screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui") -- Add to PlayerGui
        playerListUI = screenGui -- Store reference
        return screenGui
    end

    local function removePlayerListUI()
        if playerListUI and playerListUI.Parent then
            playerListUI:Destroy()
        end
        playerListUI = nil -- Clear reference

        -- Disconnect player list update connections
        for i, conn in pairs(playerListConnections) do
            if conn and typeof(conn) == "RBXScriptConnection" then
                conn:Disconnect()
            end
            playerListConnections[i] = nil
        end
    end

    TeleportTab:CreateToggle({
        Name = "Teleportar para Jogador",
        Description = "Ative para mostrar uma lista de jogadores para teleportar.",
        CurrentValue = false,
        Flag = "tp_player_toggle", -- Unique flag
        Callback = function(Value)
            if Value then
                createPlayerListUI()
                updatePlayerList() -- Initial population

                -- Connect player added/removed signals to update the list dynamically
                if not playerListConnections["PlayerAdded"] then
                    playerListConnections["PlayerAdded"] = Players.PlayerAdded:Connect(updatePlayerList)
                end
                if not playerListConnections["PlayerRemoved"] then
                     playerListConnections["PlayerRemoved"] = Players.PlayerRemoving:Connect(function(player)
                        -- Need a small delay because Remove occurs before player is fully gone
                        task.wait(0.1)
                        updatePlayerList()
                    end)
                end
            else
                removePlayerListUI()
            end
        end,
    })


    print("Rayfield Hub Initialized with all tabs.")

end -- End of LoadMainHub function


-- --- Key System UI Creation and Logic ---
-- ... (Key System UI code remains exactly the same) ...
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

    local button = Instance.new("TextButton") -- Define button here
    button.Name = randomName(9)
    button.Size = UDim2.new(0.9, 0, 0, 35)
    button.Position = UDim2.new(0.05, 0, 0.55, 0) -- Position below textbox
    button.Text = "Verificar Key"
    button.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
    button.TextColor3 = Color3.new(1, 1, 1)
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 16
    button.Parent = frame

    textbox.FocusLost:Connect(function(enterPressed) -- Check key on Enter press
        if enterPressed then
            -- Trigger button click logic
            button.MouseButton1Click:Fire() -- Fire the click event
        end
    end)

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
