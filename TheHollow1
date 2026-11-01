--[[
  Jurang Kelam — Delta script
  Runtime : Roblox / Luau, executor Delta
  Target  : PlaceId 116463962153167 (GameId 10770728771), the hub
  Source  : Dumper++ v5.0 dump, 2026-11-02, place version 167
  Run     : join the hub -> paste into Delta -> Execute.
            Re-executing unloads the previous copy first.
  Status  : written from the decompiled client scripts, NOT tested in-game.
            Anything marked (inferred) depends on a name the dump did not show live.

  How each feature works (all of it rides the game's own client paths):
   Auto collect  Eggs/UGC drops are Parts with a "PickUp" RemoteEvent. The game's own
                 code fires PickUp when the part's ProximityPrompt is shown (MaxActivationDistance 5).
                 So we just stand on the part; if it is still unclaimed after 0.8s we fire PickUp
                 ourselves, which is the exact call the game makes.
   Auto gifts    Gift spawns create a ProximityPrompt "Insert Tokens" (range 28). Triggering it runs
                 the game's insert animation, which then fires InsertToken. We trigger it while
                 SpentTokens < EarnedTokens (Elite gifts: SpentEliteTokens / EarnedEliteTokens).
   ESP           CollectionService tags: "%StateMob" (the Radar tool uses the same tag),
                 "%Portal", "%UGCPedestal" and drop parts. Highlight + distance label.
   Remote log    __namecall hook, prints FireServer / InvokeServer from the game's own scripts.
]]

local env = (getgenv and getgenv()) or _G
if env.__JK_UNLOAD then pcall(env.__JK_UNLOAD) end

local cloneref = cloneref or function(x) return x end
local Players = cloneref(game:GetService("Players"))
local RunService = cloneref(game:GetService("RunService"))
local UIS = cloneref(game:GetService("UserInputService"))
local CollectionService = cloneref(game:GetService("CollectionService"))
local Lighting = cloneref(game:GetService("Lighting"))
local lp = Players.LocalPlayer

if game.PlaceId ~= 116463962153167 then
  warn("[JK] PlaceId " .. game.PlaceId .. " is not the dumped hub (116463962153167); some features will find nothing.")
end

-- ───────────────────────── state ─────────────────────────
local alive = true
local conns = {}
local function track(c) conns[#conns + 1] = c; return c end

local S = {
  collect = false, gifts = false,
  espMobs = false, espDrops = false, espPortals = false,
  bright = false, speed = false, noclip = false, infjump = false, spy = false,
}
local SPEED = 32
local status -- TextLabel, set by the UI below
local function notify(msg)
  if status then status.Text = msg end
end

-- ───────────────────────── helpers ─────────────────────────
local function root()
  local c = lp.Character
  return c and c:FindFirstChild("HumanoidRootPart")
end

local function humanoid()
  local c = lp.Character
  return c and c:FindFirstChildOfClass("Humanoid")
end

local function posOf(i)
  if i:IsA("BasePart") then return i.Position end
  if i:IsA("Attachment") then return i.WorldPosition end
  if i:IsA("Model") then return i:GetPivot().Position end
  return nil
end

local function anchorOf(i)
  if i:IsA("BasePart") or i:IsA("Attachment") then return i end
  if i:IsA("Model") then return i.PrimaryPart or i:FindFirstChildWhichIsA("BasePart", true) end
  return nil
end

local function pickRemote(part)
  local r = part:FindFirstChild("PickUp")
  if r and string.find(r.ClassName, "RemoteEvent") then return r end
  return nil
end

-- Egg drops carry attribute EggDrop; UGC drops carry ItemId + BadgeName. Both have a PickUp remote
-- and flip the "Claimed" attribute once taken.
local function findDrops()
  local out, seen = {}, {}
  local function add(p)
    if p:IsA("BasePart") and not seen[p] and not p:GetAttribute("Claimed") and pickRemote(p) then
      seen[p] = true
      out[#out + 1] = p
    end
  end
  for _, tag in ipairs({ "%EggDrop", "%UGCDrop" }) do -- (inferred tag names)
    for _, p in ipairs(CollectionService:GetTagged(tag)) do add(p) end
  end
  for _, d in ipairs(workspace:GetDescendants()) do
    if d:IsA("BasePart")
      and (d:GetAttribute("EggDrop") ~= nil or (d:GetAttribute("ItemId") ~= nil and d:GetAttribute("BadgeName") ~= nil)) then
      add(d)
    end
  end
  return out
end

-- ───────────────────────── auto collect ─────────────────────────
local function grab(p)
  local r = root()
  if not r then return end
  r.CFrame = CFrame.new(p.Position + Vector3.new(0, 2, 0)) -- inside the 5-stud prompt range
  local t0 = os.clock()
  while alive and p.Parent and not p:GetAttribute("Claimed") and os.clock() - t0 < 0.8 do
    task.wait(0.05)
  end
  if alive and p.Parent and not p:GetAttribute("Claimed") then
    local rem = pickRemote(p)
    if rem then rem:FireServer() end -- same call as the game's PromptShown handler
    task.wait(0.25)
  end
end

task.spawn(function()
  while alive do
    if S.collect and root() then
      local list = findDrops()
      if #list > 0 then
        local home = root().CFrame
        notify("collecting " .. #list .. " drop(s)")
        for _, p in ipairs(list) do
          if not (alive and S.collect) then break end
          grab(p)
        end
        local r = root()
        if r then r.CFrame = home end
        notify("collect pass done")
      end
    end
    task.wait(2)
  end
end)

-- ───────────────────────── auto gifts ─────────────────────────
local function findGifts()
  local out = {}
  for _, d in ipairs(workspace:GetDescendants()) do
    if d:IsA("BasePart") and d:GetAttribute("MaxTokens") ~= nil
      and d:FindFirstChild("InsertToken") and d:FindFirstChildOfClass("ProximityPrompt") then
      out[#out + 1] = d
    end
  end
  return out
end

local function isElite(part)
  local s = string.lower(part.Name)
  for _, t in ipairs(CollectionService:GetTags(part)) do s = s .. " " .. string.lower(t) end
  local par = part.Parent
  while par and par ~= workspace do
    s = s .. " " .. string.lower(par.Name)
    par = par.Parent
  end
  return string.find(s, "elite") ~= nil
end

local function firePrompt(pp)
  pp.Enabled = true
  if fireproximityprompt then
    fireproximityprompt(pp)
  elseif getconnections then
    for _, c in ipairs(getconnections(pp.Triggered)) do pcall(function() c:Fire() end) end
  end
end

local giftCooldown, giftHome = {}, nil

-- returns true if it acted (so the loop re-evaluates after every token)
local function giftPass()
  for _, part in ipairs(findGifts()) do
    local elite = isElite(part)
    local spentK = elite and "SpentEliteTokens" or "SpentTokens"
    local earnedK = elite and "EarnedEliteTokens" or "EarnedTokens"
    local spent, earned, max = lp:GetAttribute(spentK), lp:GetAttribute(earnedK), part:GetAttribute("MaxTokens")
    if spent and earned and max and spent < earned and spent < max and (giftCooldown[part] or 0) < os.clock() then
      local r = root()
      if not r then return false end
      if not giftHome then giftHome = r.CFrame end
      r.CFrame = CFrame.new(part.Position + Vector3.new(0, 3, 10)) -- prompt range is 28
      task.wait(0.4)
      firePrompt(part:FindFirstChildOfClass("ProximityPrompt"))
      notify(("inserting token (%d/%d)"):format(spent, earned))
      local t0 = os.clock()
      while alive and S.gifts and lp:GetAttribute(spentK) == spent and os.clock() - t0 < 10 do
        task.wait(0.2)
      end
      if lp:GetAttribute(spentK) == spent then giftCooldown[part] = os.clock() + 60 end
      return true
    end
  end
  return false
end

task.spawn(function()
  while alive do
    if S.gifts and root() then
      local acted = giftPass()
      if not acted and giftHome then
        local r = root()
        if r then r.CFrame = giftHome end
        giftHome = nil
        notify("no tokens left to insert")
      end
    end
    task.wait(1)
  end
end)

-- ───────────────────────── ESP ─────────────────────────
local espFolder
local marks = {}

local ESP = {
  { key = "espMobs",    color = Color3.fromRGB(255, 70, 70),  targets = function() return CollectionService:GetTagged("%StateMob") end },
  { key = "espDrops",   color = Color3.fromRGB(255, 215, 60), targets = findDrops },
  { key = "espPortals", color = Color3.fromRGB(90, 200, 255), targets = function()
      local t = {}
      for _, tag in ipairs({ "%Portal", "%UGCPedestal" }) do
        for _, i in ipairs(CollectionService:GetTagged(tag)) do t[#t + 1] = i end
      end
      return t
    end },
}

local function addMark(inst, cfg)
  if marks[inst] then return end
  local anchor = anchorOf(inst)
  if not anchor then return end
  local hl = Instance.new("Highlight")
  hl.Adornee = inst
  hl.FillColor = cfg.color
  hl.OutlineColor = Color3.new(1, 1, 1)
  hl.FillTransparency = 0.6
  hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
  hl.Parent = espFolder
  local bb = Instance.new("BillboardGui")
  bb.Adornee = anchor
  bb.AlwaysOnTop = true
  bb.Size = UDim2.fromOffset(140, 18)
  bb.StudsOffset = Vector3.new(0, 3, 0)
  local tl = Instance.new("TextLabel")
  tl.Size = UDim2.fromScale(1, 1)
  tl.BackgroundTransparency = 1
  tl.TextColor3 = cfg.color
  tl.TextStrokeTransparency = 0.4
  tl.Font = Enum.Font.GothamBold
  tl.TextSize = 12
  tl.Parent = bb
  bb.Parent = espFolder
  marks[inst] = { hl = hl, bb = bb, tl = tl, key = cfg.key }
end

local function dropMark(inst)
  local m = marks[inst]
  if m then
    m.hl:Destroy()
    m.bb:Destroy()
    marks[inst] = nil
  end
end

task.spawn(function()
  local tick = 0
  while alive do
    tick = tick + 1
    local r = root()
    if tick % 2 == 1 then -- rediscover targets every second
      for _, cfg in ipairs(ESP) do
        if S[cfg.key] then
          for _, inst in ipairs(cfg.targets()) do addMark(inst, cfg) end
        end
      end
    end
    for inst, m in pairs(marks) do
      if not inst:IsDescendantOf(game) or not S[m.key] then
        dropMark(inst)
      elseif r then
        local p = posOf(inst)
        if p then m.tl.Text = ("%s [%d]"):format(inst.Name, (p - r.Position).Magnitude) end
      end
    end
    task.wait(0.5)
  end
end)

-- ───────────────────────── lighting ─────────────────────────
local lightOrig = {
  Brightness = Lighting.Brightness, ClockTime = Lighting.ClockTime, FogEnd = Lighting.FogEnd,
  FogStart = Lighting.FogStart, GlobalShadows = Lighting.GlobalShadows,
  Ambient = Lighting.Ambient, OutdoorAmbient = Lighting.OutdoorAmbient,
}
local atmos = {}

local function brightOn()
  Lighting.Brightness = 2
  Lighting.ClockTime = 14
  Lighting.FogStart = 1e6
  Lighting.FogEnd = 1e6
  Lighting.GlobalShadows = false
  Lighting.Ambient = Color3.fromRGB(190, 190, 190)
  Lighting.OutdoorAmbient = Color3.fromRGB(190, 190, 190)
  for _, a in ipairs(Lighting:GetChildren()) do
    if a:IsA("Atmosphere") then
      if atmos[a] == nil then atmos[a] = a.Density end
      a.Density = 0
    end
  end
end

local function brightOff()
  for k, v in pairs(lightOrig) do Lighting[k] = v end
  for a, d in pairs(atmos) do
    if a.Parent then a.Density = d end
  end
  atmos = {}
end

-- the game's LightingZone/LightingProfiles keep tweening Lighting, so re-apply every frame
track(RunService.RenderStepped:Connect(function()
  if S.bright then brightOn() end
end))

-- ───────────────────────── movement ─────────────────────────
local origSpeed
local noclipSaved = {}

track(RunService.Heartbeat:Connect(function()
  if S.speed then
    local h = humanoid()
    if h then h.WalkSpeed = SPEED end
  end
end))

track(RunService.Stepped:Connect(function()
  if not S.noclip then return end
  local c = lp.Character
  if not c then return end
  for _, p in ipairs(c:GetDescendants()) do
    if p:IsA("BasePart") and p.CanCollide then
      noclipSaved[p] = true
      p.CanCollide = false
    end
  end
end))

track(UIS.JumpRequest:Connect(function()
  if S.infjump then
    local h = humanoid()
    if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
  end
end))

-- ───────────────────────── teleports ─────────────────────────
local tpIndex = { portals = 0, points = 0 }

local function tpNext(kind)
  local pts = {}
  if kind == "portals" then
    for _, i in ipairs(CollectionService:GetTagged("%Portal")) do pts[#pts + 1] = i end
  else
    for _, i in ipairs(CollectionService:GetTagged("TeleTargetAttachment")) do pts[#pts + 1] = i end
  end
  table.sort(pts, function(a, b) return a:GetFullName() < b:GetFullName() end)
  if #pts == 0 then notify("no " .. kind .. " found (not streamed in?)"); return end
  tpIndex[kind] = tpIndex[kind] % #pts + 1
  local target = pts[tpIndex[kind]]
  local p, r = posOf(target), root()
  if p and r then r.CFrame = CFrame.new(p + Vector3.new(0, 4, 0)) end
  notify(("%s %d/%d: %s"):format(kind, tpIndex[kind], #pts, target.Name))
end

-- ───────────────────────── remote log ─────────────────────────
local function installSpy()
  if env.__JK_HOOKED then return true end
  if not (hookmetamethod and getnamecallmethod and newcclosure) then return false end
  local old
  old = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
    local m = getnamecallmethod()
    if env.__JK_SPY and (m == "FireServer" or m == "InvokeServer") and not (checkcaller and checkcaller()) then
      local args = table.pack(...)
      local name = self.Name
      task.spawn(function()
        local parts = {}
        for i = 1, args.n do
          local a = args[i]
          parts[i] = (typeof(a) == "Instance") and a:GetFullName() or tostring(a)
        end
        print(("[JK] %s:%s(%s)"):format(name, m, table.concat(parts, ", ")))
      end)
    end
    return old(self, ...)
  end))
  env.__JK_HOOKED = true
  return true
end

-- ───────────────────────── UI ─────────────────────────
local host = (gethui and gethui()) or cloneref(game:GetService("CoreGui"))
local gui = Instance.new("ScreenGui")
gui.Name = "JK_" .. tostring(math.random(100000, 999999))
gui.ResetOnSpawn = false
gui.Parent = host

espFolder = Instance.new("Folder")
espFolder.Name = "esp"
espFolder.Parent = gui

local BG, ON, OFF = Color3.fromRGB(24, 24, 30), Color3.fromRGB(70, 160, 90), Color3.fromRGB(48, 48, 58)

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(210, 0)
frame.AutomaticSize = Enum.AutomaticSize.Y
frame.Position = UDim2.fromOffset(12, 70)
frame.BackgroundColor3 = BG
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
local pad = Instance.new("UIPadding", frame)
pad.PaddingTop, pad.PaddingBottom, pad.PaddingLeft, pad.PaddingRight = UDim.new(0, 6), UDim.new(0, 6), UDim.new(0, 6), UDim.new(0, 6)
local layout = Instance.new("UIListLayout", frame)
layout.Padding = UDim.new(0, 4)
layout.SortOrder = Enum.SortOrder.LayoutOrder

local order = 0
local function mk(text, h)
  order = order + 1
  local b = Instance.new("TextButton")
  b.Size = UDim2.new(1, 0, 0, h or 28)
  b.BackgroundColor3 = OFF
  b.TextColor3 = Color3.new(1, 1, 1)
  b.Font = Enum.Font.GothamMedium
  b.TextSize = 13
  b.Text = text
  b.BorderSizePixel = 0
  b.LayoutOrder = order
  b.Parent = frame
  Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
  return b
end

local function toggle(label, key, onChange)
  local b = mk(label)
  local function paint() b.BackgroundColor3 = S[key] and ON or OFF end
  b.Activated:Connect(function()
    S[key] = not S[key]
    paint()
    if onChange then onChange(S[key]) end
  end)
  paint()
end

local function button(label, fn)
  mk(label).Activated:Connect(fn)
end

-- plain label (not a button) so the panel can be dragged from the title on touch screens too
local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 24)
title.BackgroundTransparency = 1
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 14
title.Text = "Jurang Kelam"
title.LayoutOrder = 0
title.Parent = frame

toggle("Auto collect eggs / UGC drops", "collect")
toggle("Auto insert gift tokens", "gifts", function(on)
  if not on and giftHome then
    local r = root()
    if r then r.CFrame = giftHome end
    giftHome = nil
  end
end)
toggle("ESP: mobs", "espMobs")
toggle("ESP: drops", "espDrops")
toggle("ESP: portals / pedestals", "espPortals")
toggle("Fullbright / no fog", "bright", function(on) if not on then brightOff() end end)
toggle("Speed x2 (" .. SPEED .. ")", "speed", function(on)
  local h = humanoid()
  if on then
    origSpeed = h and h.WalkSpeed or 16
  elseif h then
    h.WalkSpeed = origSpeed or 16
  end
end)
toggle("Noclip", "noclip", function(on)
  if not on then
    for p in pairs(noclipSaved) do
      if p.Parent then p.CanCollide = true end
    end
    noclipSaved = {}
  end
end)
toggle("Infinite jump", "infjump")
toggle("Remote log (print)", "spy", function(on)
  if on and not installSpy() then
    S.spy = false
    notify("executor lacks hookmetamethod/newcclosure")
    return
  end
  env.__JK_SPY = on
end)
button("TP next portal", function() tpNext("portals") end)
button("TP next TeleTarget", function() tpNext("points") end)

status = Instance.new("TextLabel")
status.Size = UDim2.new(1, 0, 0, 30)
status.BackgroundTransparency = 1
status.TextColor3 = Color3.fromRGB(170, 170, 185)
status.Font = Enum.Font.Gotham
status.TextSize = 11
status.TextWrapped = true
status.Text = "ready"
status.LayoutOrder = 999
status.Parent = frame

-- ───────────────────────── unload ─────────────────────────
local function unload()
  alive = false
  for k in pairs(S) do S[k] = false end
  env.__JK_SPY = false
  for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
  brightOff()
  local h = humanoid()
  if h and origSpeed then h.WalkSpeed = origSpeed end
  for p in pairs(noclipSaved) do
    if p.Parent then p.CanCollide = true end
  end
  for inst in pairs(marks) do dropMark(inst) end
  gui:Destroy()
  env.__JK_UNLOAD = nil
end
env.__JK_UNLOAD = unload

button("Unload", unload)
