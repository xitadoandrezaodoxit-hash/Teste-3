--// SERVIÇOS
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Teams = game:GetService("Teams")

--// CONFIG
local MAX_WALKSPEED = 20
local MAX_JUMPPOWER = 60
local MAX_TELEPORT_DISTANCE = 60

local ROUND_TIME = 300 -- 5 minutos
local REWARD_COINS = 100

--// REFERÊNCIAS
local brain = workspace:WaitForChild("Brain")
local brainSpawn = workspace:WaitForChild("BrainSpawn")
local baseRed = workspace:WaitForChild("BaseRed")
local baseBlue = workspace:WaitForChild("BaseBlue")

--// VARIÁVEIS
local carryingPlayer = nil
local playerData = {}
local scores = {
	Red = 0,
	Blue = 0
}

local roundActive = true

--// FUNÇÃO RESET BRAIN
local function resetBrain()
	brain.Parent = workspace
	brain.Position = brainSpawn.Position
	carryingPlayer = nil
end

--// DROP SE MORRER
local function setupDeathDrop(player)
	player.CharacterAdded:Connect(function(character)
		local humanoid = character:WaitForChild("Humanoid")

		humanoid.Died:Connect(function()
			if carryingPlayer == player then
				resetBrain()
			end
		end)
	end)
end

--// SISTEMA DE TIMES AUTOMÁTICO
local function assignTeam(player)
	if #Teams.Red:GetPlayers() <= #Teams.Blue:GetPlayers() then
		player.Team = Teams.Red
	else
		player.Team = Teams.Blue
	end
end

--// ANTI-CHEAT
local function setupAntiCheat(player)
	player.CharacterAdded:Connect(function(character)
		local humanoid = character:WaitForChild("Humanoid")
		local root = character:WaitForChild("HumanoidRootPart")

		playerData[player] = {
			LastPosition = root.Position
		}

		RunService.Heartbeat:Connect(function()
			if not character.Parent then return end

			-- Speed
			if humanoid.WalkSpeed > MAX_WALKSPEED then
				humanoid.WalkSpeed = 16
				warn(player.Name .. " Speed hack detectado")
			end

			-- Jump
			if humanoid.JumpPower > MAX_JUMPPOWER then
				humanoid.JumpPower = 50
				warn(player.Name .. " Jump hack detectado")
			end

			-- Teleport
			local currentPosition = root.Position
			local lastPosition = playerData[player].LastPosition

			if (currentPosition - lastPosition).Magnitude > MAX_TELEPORT_DISTANCE then
				root.Position = lastPosition
				warn(player.Name .. " Teleport suspeito")
			end

			playerData[player].LastPosition = root.Position
		end)
	end)
end

--// CAPTURA DO BRAIN
brain.Touched:Connect(function(hit)
	local character = hit.Parent
	local player = Players:GetPlayerFromCharacter(character)

	if player and not carryingPlayer and roundActive then
		carryingPlayer = player
		brain.Parent = character
		brain.Position = character.HumanoidRootPart.Position
	end
end)

--// MARCAR PONTO
local function scorePoint(teamName, player)
	scores[teamName] += 1
	print(teamName .. " marcou ponto! Placar: Red "
		.. scores.Red .. " x Blue " .. scores.Blue)

	-- recompensa
	local leaderstats = player:FindFirstChild("leaderstats")
	if leaderstats then
		local coins = leaderstats:FindFirstChild("Coins")
		if coins then
			coins.Value += REWARD_COINS
		end
	end

	resetBrain()
end

baseRed.Touched:Connect(function(hit)
	local player = Players:GetPlayerFromCharacter(hit.Parent)
	if player and carryingPlayer == player and player.Team.Name == "Red" then
		scorePoint("Red", player)
	end
end)

baseBlue.Touched:Connect(function(hit)
	local player = Players:GetPlayerFromCharacter(hit.Parent)
	if player and carryingPlayer == player and player.Team.Name == "Blue" then
		scorePoint("Blue", player)
	end
end)

--// SISTEMA DE ROUND
task.spawn(function()
	while true do
		roundActive = true
		scores.Red = 0
		scores.Blue = 0
		resetBrain()

		print("Round começou!")

		for i = ROUND_TIME, 0, -1 do
			wait(1)
		end

		roundActive = false

		if scores.Red > scores.Blue then
			print("Time Vermelho venceu!")
		elseif scores.Blue > scores.Red then
			print("Time Azul venceu!")
		else
			print("Empate!")
		end

		wait(10)
	end
end)

--// LEADERSTATS
local function setupLeaderstats(player)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local coins = Instance.new("IntValue")
	coins.Name = "Coins"
	coins.Value = 0
	coins.Parent = leaderstats
end

--// PLAYER ADDED
Players.PlayerAdded:Connect(function(player)
	setupLeaderstats(player)
	assignTeam(player)
	setupAntiCheat(player)
	setupDeathDrop(player)
end)
