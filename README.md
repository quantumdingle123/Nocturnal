local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local PathfindingService = game:GetService("PathfindingService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer

--// GUI
local gui = Instance.new("ScreenGui")
gui.Name = "MobileFarmGui"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local toggleBox = Instance.new("TextButton")
toggleBox.Size = UDim2.new(0, 60, 0, 60)
toggleBox.Position = UDim2.new(0, 20, 0.5, -30)
toggleBox.Text = "☰"
toggleBox.TextScaled = true
toggleBox.TextColor3 = Color3.new(1, 1, 1)
toggleBox.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
toggleBox.Parent = gui
Instance.new("UICorner", toggleBox)

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 230, 0, 210)
frame.Position = UDim2.new(0.5, -115, 0.5, -105)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.Visible = false
frame.Parent = gui
Instance.new("UICorner", frame)

-- tap vs drag
local dragging = false
local dragStart
local startPos
local dragMoved = false

toggleBox.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragMoved = false
		dragStart = input.Position
		startPos = toggleBox.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
				if not dragMoved then
					frame.Visible = not frame.Visible
				end
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
		local delta = input.Position - dragStart

		if math.abs(delta.X) > 5 or math.abs(delta.Y) > 5 then
			dragMoved = true
		end

		if dragMoved then
			toggleBox.Position = UDim2.new(
				startPos.X.Scale,
				startPos.X.Offset + delta.X,
				startPos.Y.Scale,
				startPos.Y.Offset + delta.Y
			)
		end
	end
end)

-- buttons
local mainBtn = Instance.new("TextButton")
mainBtn.Size = UDim2.new(1, -20, 0, 40)
mainBtn.Position = UDim2.new(0, 10, 0, 30)
mainBtn.Text = "MAIN: OFF"
mainBtn.TextColor3 = Color3.new(1, 1, 1)
mainBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)
mainBtn.Parent = frame
Instance.new("UICorner", mainBtn)

local altBtn = Instance.new("TextButton")
altBtn.Size = UDim2.new(1, -20, 0, 40)
altBtn.Position = UDim2.new(0, 10, 0, 75)
altBtn.Text = "ALT: OFF"
altBtn.TextColor3 = Color3.new(1, 1, 1)
altBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)
altBtn.Parent = frame
Instance.new("UICorner", altBtn)

local resetBtn = Instance.new("TextButton")
resetBtn.Size = UDim2.new(1, -20, 0, 40)
resetBtn.Position = UDim2.new(0, 10, 0, 120)
resetBtn.Text = "RESET"
resetBtn.TextColor3 = Color3.new(1, 1, 1)
resetBtn.BackgroundColor3 = Color3.fromRGB(200, 120, 40)
resetBtn.Parent = frame
Instance.new("UICorner", resetBtn)

--// states
local mode = nil
local clicking = false
local moving = false
local ignoreZone = nil
local triggerUsed = false

--// positions
local farmTarget = Vector3.new(-10.6566801, 255.272766, -611.430786)
local triggerPoint = Vector3.new(4.31565142, 255.272766, -627.21637)

local zone1 = Vector3.new(-6.06593323, 256.009552, -599.597107)
local zone2 = Vector3.new(-33.4331932, 256.009552, -593.822449)

local zoneDistance = 7
local triggerDistance = 10
local reachDistance = 3

--// character
local character, humanoid, root

local function setup()
	character = player.Character or player.CharacterAdded:Wait()
	humanoid = character:WaitForChild("Humanoid")
	root = character:WaitForChild("HumanoidRootPart")
	moving = false
	ignoreZone = nil
	triggerUsed = false
	clicking = false
end

setup()

player.CharacterAdded:Connect(function()
	task.wait(0.2)
	setup()
end)

local function isAlive()
	return character and character.Parent and humanoid and humanoid.Health > 0 and root
end

local function near(point, dist)
	return root and (root.Position - point).Magnitude <= dist
end

-- one-time pathfind when mode turns on
local function goToFarm()
	if not isAlive() then
		return
	end

	local path = PathfindingService:CreatePath({
		AgentRadius = 2,
		AgentHeight = 5,
		AgentCanJump = true,
		AgentJumpHeight = 7,
		AgentMaxSlope = 45
	})

	path:ComputeAsync(root.Position, farmTarget)

	if path.Status == Enum.PathStatus.Success then
		local waypoints = path:GetWaypoints()

		for _, waypoint in ipairs(waypoints) do
			if mode == nil or not isAlive() then
				break
			end

			if near(zone1, zoneDistance) or near(zone2, zoneDistance) then
				break
			end

			if (root.Position - waypoint.Position).Magnitude > 4 then
				if waypoint.Action == Enum.PathWaypointAction.Jump then
					humanoid.Jump = true
				end

				humanoid:MoveTo(waypoint.Position)
				local reached = humanoid.MoveToFinished:Wait()
				if not reached then
					break
				end
			end
		end
	else
		humanoid:MoveTo(farmTarget)
	end
end

-- MAIN movement
local function walkToMain(target)
	while mode == "MAIN" and isAlive() do
		if near(zone1, zoneDistance) or near(zone2, zoneDistance) then
			break
		end

		if (root.Position - target).Magnitude <= reachDistance then
			break
		end

		humanoid:MoveTo(target)
		task.wait(0.1)
	end
end

-- ALT movement
local function walkToAlt(target, stopZone)
	while mode == "ALT" and isAlive() do
		if near(stopZone, zoneDistance) then
			break
		end

		if (root.Position - target).Magnitude <= reachDistance then
			break
		end

		humanoid:MoveTo(target)
		task.wait(0.1)
	end
end

-- auto click loop
task.spawn(function()
	while true do
		if clicking and mode == "MAIN" then
			VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 0)
		end
		task.wait()
	end
end)

-- MAIN mode logic
task.spawn(function()
	while true do
		if mode == "MAIN" and isAlive() then
			if near(zone1, zoneDistance) or near(zone2, zoneDistance) then
				task.wait(0.3)
			else
				walkToMain(farmTarget)
			end
		end
		task.wait(0.2)
	end
end)

-- ALT helpers
local function getZone()
	if near(zone1, zoneDistance) then
		return 1
	elseif near(zone2, zoneDistance) then
		return 2
	end
	return nil
end

local function goOtherSide(fromZone)
	if moving or mode ~= "ALT" or not isAlive() then
		return
	end

	moving = true

	if fromZone == 1 then
		walkToAlt(zone2, zone2)
		ignoreZone = 2
	elseif fromZone == 2 then
		walkToAlt(zone1, zone1)
		ignoreZone = 1
	end

	moving = false
end

-- ALT mode logic
RunService.Heartbeat:Connect(function()
	if mode ~= "ALT" or not isAlive() then
		return
	end

	local z = getZone()

	if z == nil then
		ignoreZone = nil
		return
	end

	if z == ignoreZone then
		return
	end

	task.spawn(function()
		goOtherSide(z)
	end)
end)

-- trigger point logic
RunService.Heartbeat:Connect(function()
	if not isAlive() or mode == nil then
		return
	end

	if near(zone1, zoneDistance) or near(zone2, zoneDistance) then
		triggerUsed = false
		return
	end

	if not near(triggerPoint, triggerDistance) then
		triggerUsed = false
	end

	if near(triggerPoint, triggerDistance) and not triggerUsed then
		triggerUsed = true
		task.spawn(function()
			if mode == "MAIN" then
				walkToMain(farmTarget)
			elseif mode == "ALT" then
				walkToAlt(farmTarget, farmTarget)
			end
		end)
	end
end)

-- click control
RunService.Heartbeat:Connect(function()
	if not isAlive() or mode ~= "MAIN" then
		clicking = false
		return
	end

	if near(triggerPoint, triggerDistance) then
		clicking = false
		return
	end

	if near(zone1, zoneDistance) or near(zone2, zoneDistance) then
		clicking = true
	else
		clicking = false
	end
end)

-- buttons
mainBtn.MouseButton1Click:Connect(function()
	mode = "MAIN"
	clicking = false
	moving = false
	ignoreZone = nil
	triggerUsed = false

	mainBtn.Text = "MAIN: ON"
	mainBtn.BackgroundColor3 = Color3.fromRGB(50, 170, 70)

	altBtn.Text = "ALT: OFF"
	altBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)

	task.spawn(goToFarm)
end)

altBtn.MouseButton1Click:Connect(function()
	mode = "ALT"
	clicking = false
	moving = false
	ignoreZone = nil
	triggerUsed = false

	altBtn.Text = "ALT: ON"
	altBtn.BackgroundColor3 = Color3.fromRGB(50, 170, 70)

	mainBtn.Text = "MAIN: OFF"
	mainBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)

	task.spawn(goToFarm)
end)

resetBtn.MouseButton1Click:Connect(function()
	mode = nil
	clicking = false
	moving = false
	ignoreZone = nil
	triggerUsed = false

	mainBtn.Text = "MAIN: OFF"
	mainBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)

	altBtn.Text = "ALT: OFF"
	altBtn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)
end)
