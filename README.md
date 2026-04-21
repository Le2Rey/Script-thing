# Script-thing
ModuleScript useful for guns, requires of other scripts to work properly but this is just an example.

local ContextActionService = game:GetService("ContextActionService")
local Players = game:GetService("Players")

local LocalPlayer = Players.LocalPlayer

local Effects = require(script.Parent.Parent.Effects)
local GetDamage = require(script.Parent.Parent.GetDamage)

local Remotes = script.Parent.Parent.Remotes
local FireRemote = Remotes.Fire
local KillRemote = Remotes.Kill

local Remotes2 = script.Parent.Parent.Parent.Remotes
local SoundRemote = Remotes2.Sound

local Gun = {
	Player = LocalPlayer,
	Mouse = LocalPlayer:GetMouse()
}
Gun.__index = Gun

local Id = 0

function Gun.New(Tool: Tool, Properties)
	local self = setmetatable({}, Gun)
	
	self.Tool = Tool
	self.Gui = script.GunGui:Clone()
	self.Gui.MainFrame.GunName.Text = self.Tool.Name
	
	Id += 1
	self.Id = Id
	
	self.Gui.Enabled = false
	self.Gui.MainFrame.GunFrame.Gun.Image = Properties.GunImage
	
	self.Gui.Parent = self.Player.PlayerGui
	
	self.MaxAmmo = Properties.MaxAmmo
	self.Ammo = self.MaxAmmo
	self.Firerate = Properties.Firerate
	self.ReloadTime = Properties.ReloadTime
	self.Damage = Properties.Damage
	self.Spread = Properties.Spread
	self.Range = Properties.Range
	self.AnimationIds = Properties.Animations
	self.HoldStates = Properties.HoldStates
	
	self.Destroyed = false
	self.Firing = false
	self.Equipped = false
	self.Reloading = false
	
	self.Character = self.Player.Character or self.Player.CharacterAdded:Wait()
	self.Humanoid = self.Character:WaitForChild("Humanoid")
	self.Animator = self.Humanoid:WaitForChild("Animator")
	
	self.Connections = {}
	self:LoadAnimations()
	
	self.RaycastParams = RaycastParams.new()
	self.RaycastParams.FilterType = Enum.RaycastFilterType.Exclude
	self.RaycastParams.IgnoreWater = true
	self.RaycastParams.FilterDescendantsInstances = {self.Tool, self.Character}
	
	self.MovementModule = self.Character:WaitForChild("MovementActivation"):WaitForChild("Movement")
	self.Movement = require(self.MovementModule)
	
	self.FireAction = "FireAction_"..self.Id
	self.ReloadAction = "ReloadAction_"..self.Id
	
	table.insert(self.Connections,
		self.MovementModule.StateChanged.Event:Connect(function(State)
			if State == "Sprint" then
				if self.Equipped then
					self.Animations.Idle:Stop()
				end
				self.Tool.Grip = self.HoldStates.Sprint
			else
				if self.Equipped then
					self.Animations.Idle:Play()
				end
				self.Tool.Grip = self.HoldStates.Idle
			end
		end)
	)
	
	table.insert(
		self.Connections, self.Tool.Equipped:Connect(function()
			self.Equipped = true
			self.Gui.Enabled = true
			if self.Movement:GetState() == "Sprint" then
				self.Movement:SetSprintAnimation(true)
			else
				self.Animations.Idle:Play()
			end
			if _G.Settings["Custom Cursor"] then
				self.Mouse.Icon = _G.CustomCursor or "rbxassetid://124821953531726"
			else
				self.Mouse.Icon = "rbxassetid://124821953531726"
			end
			
			ContextActionService:BindAction(
				self.FireAction,
				function(_, State)
					if State == Enum.UserInputState.Begin then
						if self.Humanoid.Health > 0 and self.Ammo > 0 and not self.Reloading and self.Equipped then
							self:Fire()
						end
					elseif State == Enum.UserInputState.End then
						self.Firing = false
					end
				end,
				false,
				Enum.UserInputType.MouseButton1
			)
			
			ContextActionService:BindAction(
				self.ReloadAction,
				function(_, State)
					if State == Enum.UserInputState.Begin then
						self:Reload()
					end
				end,
				false,
				Enum.KeyCode.R
			)
		end)
	)
	
	table.insert(
		self.Connections, self.Tool.Unequipped:Connect(function()
			ContextActionService:UnbindAction(self.FireAction)
			ContextActionService:UnbindAction(self.ReloadAction)
			
			self.Mouse.Icon = ""
			self.Equipped = false
			self.Gui.Enabled = false
			self.Firing = false
			self.Reloading = false
			
			self.Animations.Idle:Stop()
			self.Movement:SetSprintAnimation(false)
		end)
	)
	
	table.insert(self.Connections,
		KillRemote.OnClientEvent:Connect(function()
			Effects:Killmarker()
		end)
	)
	
	table.insert(self.Connections,
		self.Humanoid.Died:Connect(function()
			self:Destroy()
		end)
	)
	
	table.insert(self.Connections,
		self.Tool.Destroying:Connect(function()
			self:Destroy()
		end)
	)
	
	return self
end

function Gun:LoadAnimations()
	self.Animations = {}
	self.Garbage = self.Garbage or {}
	for Name, Id in self.AnimationIds do
		self.Garbage[Name] = Instance.new("Animation")
		self.Garbage[Name].AnimationId = Id
		self.Animations[Name] = self.Animator:LoadAnimation(self.Garbage[Name])
	end
end

function Gun:Fire()
	self:Shoot()
end

function Gun:SetAmmo(Ammo: number)
	self.Ammo = Ammo
	local Progress = self.Ammo/self.MaxAmmo
	self.Gui.MainFrame.AmmoFrame.Ammo.Text = tostring(math.floor(Progress*100)).."%"
	self.Gui.MainFrame.Bar.Progress:TweenSize(
		UDim2.fromScale(Progress, 1),
		Enum.EasingDirection.Out,
		Enum.EasingStyle.Quad,
		0.05,
		false
	)
end

function Gun:Shoot()
	self.CanFire = false
	self.Animations.Shoot.Looped = false
	self.Animations.Shoot:Play()
	
	self.Tool.Handle.Fire:Play()
	SoundRemote:FireServer(self.Tool.Handle.Fire)
	
	local Origin = self.Character.Head.Position
	local Aim = self.Mouse.Hit.Position
	
	local Distance = (Aim-Origin).Magnitude
	local MinSpread = -(self.Spread)*Distance 
	local MaxSpread =  (self.Spread)*Distance
	
	Aim = Vector3.new(
		Aim.X + math.random(MinSpread, MaxSpread), 
		Aim.Y + math.random(MinSpread, MaxSpread), 
		Aim.Z + math.random(MinSpread, MaxSpread)
	)

	local Direction = (Aim-Origin).Unit*self.Range
	local RaycastResult = workspace:Raycast(Origin, Direction, self.RaycastParams)
	if not RaycastResult or not RaycastResult.Position then
		RaycastResult = {Position = Origin+Direction}
	end

	local Humanoid
	if RaycastResult.Instance then
		if RaycastResult.Instance.Parent ~= game.Workspace then
			local Hit: BasePart = RaycastResult.Instance
			local Rig = Hit:FindFirstAncestorOfClass("Model")
			if Rig then
				Humanoid = Rig:FindFirstChildOfClass("Humanoid")
			end
		end
	end
	
	if Humanoid and Humanoid.Health > 0 then
		local Damage = GetDamage(self.Player, RaycastResult.Instance, Humanoid, self.Damage)
		if Damage ~= 0 then
			local Critical = RaycastResult.Instance.Name == "Head"
			Effects:DamageIndicator(Damage, Humanoid.Parent.HumanoidRootPart, Critical)
			Effects:Hitmarker(Critical)
			Effects:LimbIndicator(RaycastResult.Instance)
		end
	end
	
	FireRemote:FireServer(self.Tool, Humanoid, RaycastResult.Instance, Origin, RaycastResult.Position)
	Effects:Beam(self.Tool.Handle.Barrel.WorldPosition, RaycastResult.Position, self.Player.TeamColor.Color)
	
	self:SetAmmo(self.Ammo - 1)
end

function Gun:Reload()
	if self.Reloading or self.Ammo >= self.MaxAmmo then
		return
	end
	self.Reloading = true
	self.Firing = false

	task.spawn(function()
		while not self.Destroyed and self.Reloading and self.Ammo < self.MaxAmmo do
			self:SetAmmo(self.Ammo + 1)
			task.wait(self.ReloadTime / self.MaxAmmo)
		end
		self.Reloading = false
	end)
end

function Gun:Destroy()
	if self.Destroyed then
		return
	end
	self.Destroyed = true
	
	ContextActionService:UnbindAction(self.FireAction)
	ContextActionService:UnbindAction(self.ReloadAction)
	
	for _, v in self.Connections do
		v:Disconnect()
	end
	self.Connections = nil
	for _, v in self.Animations do
		v:Stop()
		v:Destroy()
	end
	for _, v in self.Garbage do
		v:Destroy()
	end
	if self.Gui then
		self.Gui:Destroy()
	end
end

return Gun
