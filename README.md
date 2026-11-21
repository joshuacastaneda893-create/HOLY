--[[
	SAB code sniper (Luau, executor environment, client side).
	Status: NOT run in-game and not executed anywhere. See the "UNSET" block in CONFIG.

	Flow:
	  1. Owner (Sammy) says a trigger phrase ("the code is", "use code", "code is").
	  2. Text after the trigger (if any) is piece 1. Later owner messages that look like
	     a single code token are the next pieces.
	  3. When CONFIG.partsToCollect pieces are collected (1..4), pieces are joined and redeemed.
	  4. Pieces that do not arrive within collectWindowSeconds cancel the attempt.

	Runtime control:
	  F6 (CONFIG.cycleKey)              cycles partsToCollect 1 -> 2 -> 3 -> 4 -> 1
	  getgenv().SAB_SNIPER.setParts(n)  set 1..4
	  getgenv().SAB_SNIPER.feed(text)   push a fake owner message (offline logic test)
	  getgenv().SAB_SNIPER.stop()       disconnect everything
]]

local Players = game:GetService("Players")
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

local env = (getgenv and getgenv()) or _G
if env.SAB_SNIPER and env.SAB_SNIPER.stop then
	pcall(env.SAB_SNIPER.stop)
end

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CONFIG = {
	-- Given by you
	triggers = { "the code is", "use code", "code is" },
	partsToCollect = 2, -- 1..4, total pieces needed (the piece inside the trigger message counts)
	enabled = true, -- power switch (panel button); false = ignore all announcements
	autoRedeem = true, -- false = assemble the code and copy it to the clipboard, but do not redeem

	-- Defaults I chose (change freely)
	countTriggerPiece = true, -- text after the trigger counts as piece 1
	collectWindowSeconds = 45, -- cancel if the remaining pieces do not arrive in time
	joinWith = "", -- separator between pieces
	codeCase = "keep", -- "keep" | "upper" | "lower" (game's case rule is UNKNOWN)
	maxPieceLength = 24,
	piecePattern = "^[%w_%-]+$", -- what a valid piece looks like
	cycleKey = Enum.KeyCode.F6,

	-- Controller capture (learnController). Function names are matched as lowercase substrings.
	functionKeyPatterns = { "redeem", "promo", "coupon", "usecode", "entercode", "submitcode", "claimcode" },
	controllerModulePattern = "[Cc]ontroller", -- used only with { allInControllers = true }
	maxHooked = 300,

	-- UNSET: must come from you / from your own inspection of the game
	ownerUserIds = {}, -- e.g. [123456789] = true  (Sammy's UserId, NOT provided)
	ownerNames = {}, -- optional, lowercase Name or DisplayName, e.g. { "sammyname" }
	acceptUnattributed = false, -- true = also trust chat/system lines that carry no speaker
	announceRemotePath = nil, -- optional: path of the RemoteEvent the game uses for announcements,
	-- e.g. { "ReplicatedStorage", "Packages", "Net", "<name>" }. Its string arg is treated as owner text.

	redeem = {
		mode = "remote", -- "remote" (default), "controller" (set by learnController) or "gui"
		call = nil, -- mode "controller": { fn, template, idx, sub }, filled by learnController
		remote = nil, -- UNSET: the redeem Remote instance; filled automatically by SAB_SNIPER.learn
		remotePath = nil, -- UNSET: path table to it, e.g. { "ReplicatedStorage", "...", "<name>" }
		buildArgs = function(code) -- argument shape is UNVERIFIED; confirm with a remote spy
			return { code }
		end,
		textBoxPath = nil, -- mode "gui": path to the code TextBox
		buttonPath = nil, -- mode "gui": path to the redeem button
	},
}

----------------------------------------------------------------------
-- Pure logic (testable without the game)
----------------------------------------------------------------------
local function stripRichText(s)
	return (s:gsub("<[^>]+>", ""))
end

local function sanitizePiece(raw, cfg, singleTokenOnly)
	local s = raw:gsub("^%s+", "")
	s = s:gsub("%s+$", "")
	if s == "" then
		return nil
	end
	if singleTokenOnly and s:find("%s") then
		return nil -- normal chatter, not a code piece
	end
	s = s:match("^%S+")
	s = s:gsub("^[:;,%.!%?\"'`]+", "")
	s = s:gsub("[:;,%.!%?\"'`]+$", "")
	if s == "" or #s > cfg.maxPieceLength or not s:match(cfg.piecePattern) then
		return nil
	end
	return s
end

local function findTrigger(text, triggers)
	local lower = text:lower()
	local bestS, bestE
	for _, t in ipairs(triggers) do
		local s, e = lower:find(t:lower(), 1, true)
		if s and (not bestS or s < bestS or (s == bestS and e > bestE)) then
			bestS, bestE = s, e
		end
	end
	return bestS, bestE
end

local function newCollector(cfg, onCode, clock)
	local active, parts, deadline = false, {}, 0
	local self = {}

	local function reset()
		active, parts, deadline = false, {}, 0
	end

	local function add(piece)
		parts[#parts + 1] = piece
		if #parts >= cfg.partsToCollect then
			local code = table.concat(parts, cfg.joinWith)
			reset()
			onCode(code)
		end
	end

	function self.feed(text)
		if cfg.enabled == false then
			return
		end
		text = stripRichText(text)
		local now = clock()
		if active and now > deadline then
			reset()
		end

		local s, e = findTrigger(text, cfg.triggers)
		if s then
			reset()
			active = true
			deadline = now + cfg.collectWindowSeconds
			if cfg.countTriggerPiece then
				local piece = sanitizePiece(text:sub(e + 1), cfg, false)
				if piece then
					add(piece)
				end
			end
			return
		end

		if active then
			local piece = sanitizePiece(text, cfg, true)
			if piece then
				add(piece)
			end
		end
	end

	function self.progress()
		return { active = active, parts = table.clone(parts), needed = cfg.partsToCollect }
	end

	return self
end

----------------------------------------------------------------------
-- Game-facing code
----------------------------------------------------------------------
local function log(...)
	print("[SAB-Sniper]", ...)
end

local function resolvePath(path)
	if type(path) ~= "table" then
		return nil
	end
	local cur = game
	for _, seg in ipairs(path) do
		cur = cur and cur:FindFirstChild(seg)
		if not cur then
			return nil
		end
	end
	return cur
end

local redeemed = {}

local function applyCase(code)
	if CONFIG.codeCase == "upper" then
		return code:upper()
	elseif CONFIG.codeCase == "lower" then
		return code:lower()
	end
	return code
end

local function redeem(code)
	local r = CONFIG.redeem

	if r.mode == "remote" then
		local remote = (r.remote and r.remote.Parent and r.remote) or resolvePath(r.remotePath)
		if not remote then
			log("redeem remote UNSET. Run SAB_SNIPER.learn(\"TEXT\") and redeem TEXT once by hand. Code was:", code)
			if setclipboard then
				pcall(setclipboard, code)
			end
			return
		end
		local args = r.buildArgs(code)
		local ok, result = pcall(function()
			if remote:IsA("RemoteEvent") then
				remote:FireServer(table.unpack(args, 1, args.n or #args))
				return "fired"
			elseif remote:IsA("RemoteFunction") then
				return remote:InvokeServer(table.unpack(args, 1, args.n or #args))
			end
			error("redeem path is neither RemoteEvent nor RemoteFunction: " .. remote.ClassName)
		end)
		log("redeem", code, ok and "sent" or "FAILED", tostring(result))
		return
	end

	if r.mode == "controller" then
		local c = r.call
		if not c then
			log("controller call UNSET. Run SAB_SNIPER.learnController(\"TEXT\"). Code was:", code)
			if setclipboard then
				pcall(setclipboard, code)
			end
			return
		end
		local out = table.clone(c.template)
		if c.sub ~= nil then
			local inner = table.clone(out[c.idx])
			inner[c.sub] = code
			out[c.idx] = inner
		else
			out[c.idx] = code
		end
		task.spawn(function()
			local ok, res = pcall(c.fn, table.unpack(out, 1, out.n))
			log("controller redeem", code, ok and "returned" or "FAILED", tostring(res))
		end)
		return
	end

	if r.mode == "gui" then
		local box, btn = resolvePath(r.textBoxPath), resolvePath(r.buttonPath)
		if not (box and btn and box:IsA("TextBox")) then
			log("gui redeem paths not resolved; code was:", code)
			return
		end
		if not firesignal then
			log("executor has no firesignal; code was:", code)
			return
		end
		local ok, err = pcall(function()
			box.Text = code
			firesignal(btn.Activated)
		end)
		log("gui redeem", code, ok and "triggered" or ("FAILED " .. tostring(err)))
		return
	end

	-- Gated: redeem method is UNSET. Code is detected and surfaced, not sent.
	log("REDEEM NOT CONFIGURED (set CONFIG.redeem). Detected code:", code)
	if setclipboard then
		pcall(setclipboard, code)
		log("copied to clipboard")
	end
end

local function onCode(rawCode)
	local code = applyCase(rawCode)
	if not CONFIG.autoRedeem then
		log("auto redeem OFF. Assembled code:", code)
		if setclipboard then
			pcall(setclipboard, code)
		end
		return
	end
	if redeemed[code] then
		log("already handled, skipping:", code)
		return
	end
	redeemed[code] = true
	log("assembled code:", code)
	redeem(code)
end

local collector = newCollector(CONFIG, onCode, os.clock)

-- Owner filter
local ownerNameSet = {}
for _, n in ipairs(CONFIG.ownerNames) do
	ownerNameSet[n:lower()] = true
end

local function ownerConfigured()
	return next(CONFIG.ownerUserIds) ~= nil or next(ownerNameSet) ~= nil
end

local function isOwnerByUserId(userId)
	if CONFIG.ownerUserIds[userId] then
		return true
	end
	local p = Players:GetPlayerByUserId(userId)
	return p ~= nil and (ownerNameSet[p.Name:lower()] or ownerNameSet[p.DisplayName:lower()]) == true
end

local function isOwnerByName(name)
	return ownerNameSet[name:lower()] == true
end

local conns = {}
local function track(c)
	conns[#conns + 1] = c
end

-- Source 1: Roblox chat (exactly one of the two, to avoid double counting)
if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
	local ok, err = pcall(function()
		track(TextChatService.MessageReceived:Connect(function(m)
			local src = m.TextSource
			if src then
				if ownerConfigured() and isOwnerByUserId(src.UserId) then
					collector.feed(m.Text)
				end
			elseif CONFIG.acceptUnattributed then
				collector.feed(m.Text)
			end
		end))
	end)
	if not ok then
		log("TextChatService hook failed:", tostring(err))
	end
else
	local events = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
	local done = events and events:FindFirstChild("OnMessageDoneFiltering")
	if done then
		track(done.OnClientEvent:Connect(function(d)
			if type(d) == "table" and type(d.Message) == "string" and type(d.FromSpeaker) == "string" then
				if ownerConfigured() and isOwnerByName(d.FromSpeaker) then
					collector.feed(d.Message)
				end
			end
		end))
	else
		log("legacy chat event not found")
	end
end

-- Source 2: the game's own announcement RemoteEvent (only if you give its path)
if CONFIG.announceRemotePath then
	local remote = resolvePath(CONFIG.announceRemotePath)
	if remote and remote:IsA("RemoteEvent") then
		track(remote.OnClientEvent:Connect(function(...)
			for _, v in ipairs({ ... }) do
				if type(v) == "string" then
					collector.feed(v)
					break
				end
			end
		end))
		log("announcement remote hooked")
	else
		log("announceRemotePath did not resolve to a RemoteEvent")
	end
end

-- Learn mode: captures the game's own redeem call from one manual redeem.
-- The __namecall hook is installed once per session and only passes through when not armed.
local learnShared = env.SAB_LEARN
if not learnShared then
	learnShared = { pending = nil, installed = false }
	env.SAB_LEARN = learnShared
	if hookmetamethod and getnamecallmethod and newcclosure then
		local old
		old = hookmetamethod(
			game,
			"__namecall",
			newcclosure(function(self, ...)
				local method = getnamecallmethod()
				local pending = learnShared.pending
				if
					pending
					and (method == "FireServer" or method == "InvokeServer")
					and not (checkcaller and checkcaller())
				then
					-- Defer: calling Instance methods inside the hook would clobber the namecall method.
					task.defer(pending, self, method, table.pack(...))
				end
				return old(self, ...)
			end)
		)
		learnShared.installed = true
	end
end

local function learn(typedText)
	if type(typedText) ~= "string" or typedText == "" then
		log('usage: SAB_SNIPER.learn("TEXT"), then redeem exactly TEXT once in the game UI')
		return
	end
	if not learnShared.installed then
		log("learn unavailable: executor lacks hookmetamethod / getnamecallmethod / newcclosure")
		return
	end
	local armedAt = os.clock()
	log("learn armed for 60s. Redeem exactly:", typedText)
	learnShared.pending = function(remote, method, args)
		if os.clock() - armedAt > 60 then
			learnShared.pending = nil
			log("learn timed out")
			return
		end
		if typeof(remote) ~= "Instance" then
			return
		end
		local idx
		for i = 1, args.n do
			if args[i] == typedText then
				idx = i
				break
			end
		end
		if not idx then
			return
		end
		learnShared.pending = nil

		local path, cur = {}, remote
		while cur and cur ~= game do
			table.insert(path, 1, cur.Name)
			cur = cur.Parent
		end

		local template = args
		CONFIG.redeem.mode = "remote"
		CONFIG.redeem.remote = remote
		CONFIG.redeem.remotePath = (cur == game) and path or nil
		CONFIG.redeem.buildArgs = function(code)
			local out = table.clone(template)
			out[idx] = code
			return out
		end

		local types = {}
		for i = 1, args.n do
			types[i] = (i == idx) and "<CODE>" or typeof(args[i])
		end
		log("LEARNED", method, "on", remote:GetFullName(), "| code is arg #" .. idx)
		log("arg shape:", table.concat(types, ", "))
		log("path table: { \"" .. table.concat(path, "\", \"") .. "\" }")
		if args.n > 1 then
			log("NOTE: other args are replayed as captured; if they are tokens/timestamps they may go stale")
		end
	end
end

-- Controller capture: hook the game's own client-side redeem function(s), capture the call from one
-- manual redeem, then replay it directly with the assembled code.
local function collectCandidates(allInControllers)
	local found, seen = {}, {}

	local function consider(tbl, key, fn)
		if #found >= CONFIG.maxHooked or seen[fn] then
			return
		end
		if iscclosure and iscclosure(fn) then
			return
		end
		seen[fn] = true
		found[#found + 1] = { tbl = tbl, key = key, fn = fn }
	end

	local function keyMatches(key)
		if type(key) ~= "string" then
			return false
		end
		local k = key:lower()
		for _, p in ipairs(CONFIG.functionKeyPatterns) do
			if k:find(p, 1, true) then
				return true
			end
		end
		return false
	end

	if getgc then
		for _, t in ipairs(getgc(true)) do
			if type(t) == "table" then
				pcall(function()
					for k, v in next, t do
						if type(v) == "function" and keyMatches(k) then
							consider(t, k, v)
						end
					end
				end)
			end
		end
	end

	if allInControllers and getloadedmodules then
		for _, m in ipairs(getloadedmodules()) do
			if m.Name:match(CONFIG.controllerModulePattern) then
				local ok, res = pcall(require, m)
				if ok and type(res) == "table" then
					pcall(function()
						for k, v in next, res do
							if type(v) == "function" then
								consider(res, k, v)
							end
						end
					end)
				end
			end
		end
	end

	return found
end

local function learnController(typedText, opts)
	if type(typedText) ~= "string" or typedText == "" then
		log('usage: SAB_SNIPER.learnController("TEXT"), then redeem exactly TEXT once in the game UI')
		return
	end
	if not (hookfunction and newcclosure and getgc) then
		log("learnController unavailable: executor lacks hookfunction / newcclosure / getgc")
		return
	end

	if learnShared.ctrl then
		learnShared.ctrl.armed = false
	end
	local st = { armed = true, armedAt = os.clock() }
	learnShared.ctrl = st

	local cands = collectCandidates(opts and opts.allInControllers)
	if #cands == 0 then
		log("no candidate functions found. Try learnController(text, { allInControllers = true })")
		st.armed = false
		return
	end

	st.onCall = function(c, original, args)
		if not st.armed then
			return
		end
		local idx, sub
		for i = 1, args.n do
			if args[i] == typedText then
				idx = i
				break
			elseif type(args[i]) == "table" then
				for k, v in next, args[i] do
					if v == typedText then
						idx, sub = i, k
						break
					end
				end
				if idx then
					break
				end
			end
		end
		if not idx then
			return
		end
		st.armed = false

		CONFIG.redeem.mode = "controller"
		CONFIG.redeem.call = { fn = original, template = args, idx = idx, sub = sub }

		local types = {}
		for i = 1, args.n do
			types[i] = (i == idx and sub == nil) and "<CODE>" or typeof(args[i])
		end
		log("LEARNED controller function key:", tostring(c.key), "| code is arg #" .. idx
			.. (sub ~= nil and (" field " .. tostring(sub)) or ""))
		log("arg shape:", table.concat(types, ", "))
		if args.n > 1 then
			log("NOTE: other args are replayed as captured (including a self table if it is a method call)")
		end
	end

	local hooked = 0
	for _, c in ipairs(cands) do
		local original
		local ok = pcall(function()
			original = hookfunction(
				c.fn,
				newcclosure(function(...)
					if st.armed then
						task.defer(st.onCall, c, original, table.pack(...))
					end
					return original(...)
				end)
			)
		end)
		if ok and original then
			hooked += 1
		end
	end

	log("learnController armed for 60s on", hooked, "of", #cands, "candidate functions. Redeem exactly:", typedText)
	task.delay(60, function()
		if st.armed then
			st.armed = false
			log("learnController timed out (hooks stay installed but pass through)")
		end
	end)
end

-- Hotkey + API
local uiRefresh = function() end -- replaced by the control panel below

local function setParts(n)
	n = math.floor(tonumber(n) or 0)
	if n < 1 or n > 4 then
		log("parts must be 1..4")
		return
	end
	CONFIG.partsToCollect = n
	log("partsToCollect =", n)
	uiRefresh()
end

track(UserInputService.InputBegan:Connect(function(input, processed)
	if not processed and input.KeyCode == CONFIG.cycleKey then
		setParts(CONFIG.partsToCollect % 4 + 1)
	end
end))

env.SAB_SNIPER = {
	stop = function()
		if env.SAB_GUI then
			pcall(function()
				env.SAB_GUI:Destroy()
			end)
			env.SAB_GUI = nil
		end
		learnShared.pending = nil
		if learnShared.ctrl then
			learnShared.ctrl.armed = false
		end
		for _, c in ipairs(conns) do
			c:Disconnect()
		end
		table.clear(conns)
		env.SAB_SNIPER = nil
		log("stopped")
	end,
	setParts = setParts,
	learn = learn,
	learnController = learnController,
	feed = function(text)
		collector.feed(text)
	end,
	progress = collector.progress,
}

-- Control panel: redeemer counter, method selector, auto-redeem toggle, one-tap learn buttons.
local METHODS = { "remote", "controller", "gui" }

local function methodIndex()
	for i, m in ipairs(METHODS) do
		if m == CONFIG.redeem.mode then
			return i
		end
	end
	return 1
end

local function buildUi()
	local parent = (gethui and gethui()) or Players.LocalPlayer:WaitForChild("PlayerGui")
	local gui = Instance.new("ScreenGui")
	gui.Name = "SabSniperUi"
	gui.ResetOnSpawn = false
	gui.Parent = parent
	env.SAB_GUI = gui

	local function mk(class, props, par)
		local o = Instance.new(class)
		for k, v in pairs(props) do
			o[k] = v
		end
		o.Parent = par
		return o
	end

	local frame = mk("Frame", {
		Size = UDim2.fromOffset(310, 250),
		Position = UDim2.fromOffset(20, 80),
		BackgroundColor3 = Color3.fromRGB(16, 26, 58),
		BorderSizePixel = 0,
		Active = true,
		Draggable = true,
	}, gui)
	mk("UICorner", { CornerRadius = UDim.new(0, 8) }, frame)

	local function label(text, x, y, w, align)
		return mk("TextLabel", {
			Text = text,
			Position = UDim2.fromOffset(x, y),
			Size = UDim2.fromOffset(w, 26),
			BackgroundTransparency = 1,
			TextColor3 = Color3.fromRGB(235, 240, 255),
			Font = Enum.Font.Gotham,
			TextSize = 14,
			TextXAlignment = align or Enum.TextXAlignment.Left,
		}, frame)
	end

	local function button(text, x, y, w, cb)
		local b = mk("TextButton", {
			Text = text,
			Position = UDim2.fromOffset(x, y),
			Size = UDim2.fromOffset(w, 26),
			BackgroundColor3 = Color3.fromRGB(34, 52, 110),
			TextColor3 = Color3.fromRGB(235, 240, 255),
			Font = Enum.Font.GothamMedium,
			TextSize = 13,
			AutoButtonColor = true,
		}, frame)
		mk("UICorner", { CornerRadius = UDim.new(0, 6) }, b)
		b.Activated:Connect(cb)
		return b
	end

	label("SAB code sniper", 10, 6, 200)

	label("Redeemer", 10, 36, 100)
	button("-", 130, 36, 26, function()
		setParts(math.max(1, CONFIG.partsToCollect - 1))
	end)
	local countLbl = label("", 160, 36, 46, Enum.TextXAlignment.Center)
	button("+", 210, 36, 26, function()
		setParts(math.min(4, CONFIG.partsToCollect + 1))
	end)
	local powerBtn = button("ON", 262, 36, 40, function()
		CONFIG.enabled = not CONFIG.enabled
		uiRefresh()
	end)

	label("Method", 10, 72, 80)
	local methodBtn = button("", 100, 72, 202, function()
		CONFIG.redeem.mode = METHODS[methodIndex() % #METHODS + 1]
		uiRefresh()
	end)

	label("Auto redeem on announcement", 10, 108, 220)
	local autoBtn = button("ON", 242, 108, 60, function()
		CONFIG.autoRedeem = not CONFIG.autoRedeem
		uiRefresh()
	end)

	local box = mk("TextBox", {
		Text = "",
		PlaceholderText = "text to type into the game's redeem box",
		Position = UDim2.fromOffset(10, 146),
		Size = UDim2.fromOffset(292, 26),
		BackgroundColor3 = Color3.fromRGB(10, 16, 38),
		TextColor3 = Color3.fromRGB(235, 240, 255),
		PlaceholderColor3 = Color3.fromRGB(120, 130, 160),
		Font = Enum.Font.Gotham,
		TextSize = 13,
		ClearTextOnFocus = false,
	}, frame)
	mk("UICorner", { CornerRadius = UDim.new(0, 6) }, box)

	button("Learn remote", 10, 182, 142, function()
		learn(box.Text)
	end)
	button("Learn controller", 160, 182, 142, function()
		learnController(box.Text)
	end)

	local statusLbl = label("", 10, 216, 292)

	uiRefresh = function()
		if not gui.Parent then
			return
		end
		countLbl.Text = CONFIG.partsToCollect .. "/4"
		powerBtn.Text = CONFIG.enabled and "ON" or "OFF"
		powerBtn.BackgroundColor3 = CONFIG.enabled and Color3.fromRGB(40, 110, 200) or Color3.fromRGB(110, 40, 50)
		local mi = methodIndex()
		methodBtn.Text = "Method " .. mi .. " (" .. METHODS[mi] .. ")"
		autoBtn.Text = CONFIG.autoRedeem and "ON" or "OFF"
		autoBtn.BackgroundColor3 = CONFIG.autoRedeem and Color3.fromRGB(40, 110, 200) or Color3.fromRGB(110, 40, 50)
		local p = collector.progress()
		if not CONFIG.enabled then
			statusLbl.Text = "Paused"
		elseif p.active then
			statusLbl.Text = "Collecting " .. #p.parts .. "/" .. p.needed .. ": " .. table.concat(p.parts, " ")
		else
			statusLbl.Text = "Waiting for trigger"
		end
	end

	task.spawn(function()
		while gui.Parent do
			uiRefresh()
			task.wait(0.5)
		end
	end)
	uiRefresh()
end

local okUi, errUi = pcall(buildUi)
if not okUi then
	log("control panel failed to build:", tostring(errUi))
end

if not ownerConfigured() and not CONFIG.announceRemotePath and not CONFIG.acceptUnattributed then
	log("WARNING: no owner / announcement source configured, so chat is ignored. Set ownerUserIds.")
end
log("loaded; partsToCollect =", CONFIG.partsToCollect)
